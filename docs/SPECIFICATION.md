# AMIP Specification

**AI-Machine Interface Protocol**
**Version:** 1.0.0
**Status:** Draft
**License:** MIT

---

I started this project because I kept running into the same wall. Every
time I wanted to connect an AI system to a robot, I had to rebuild the
same plumbing from scratch. Custom command formats, safety checks written
in a hurry, no consistency between simulation and real hardware, and
feedback that told me what I asked for rather than what actually happened.
After doing this several times it became obvious the problem was not the
robots or the AI — it was the gap between them. AMIP is my attempt to
fill that gap properly, once, in a way that anyone can use and anyone can
extend.

This document explains what AMIP is, why it is designed the way it is,
and what the rules are. If you want to contribute, read it before you
write a line of code. The decisions here are not arbitrary — they come
from real problems, and understanding them will save you from reopening
them in PRs.

---

## Table of Contents

1. [What AMIP Is](#1-what-amip-is)
2. [The Problem It Solves](#2-the-problem-it-solves)
3. [Design Principles](#3-design-principles)
4. [The Theory Behind the Decisions](#4-the-theory-behind-the-decisions)
5. [Architecture](#5-architecture)
6. [Command Protocol](#6-command-protocol)
7. [Capability Descriptor](#7-capability-descriptor)
8. [Safety Contract](#8-safety-contract)
9. [Feedback Contract](#9-feedback-contract)
10. [Hardware Adapter Plugins](#10-hardware-adapter-plugins)
11. [State Representation](#11-state-representation)
12. [Error Handling](#12-error-handling)
13. [Versioning](#13-versioning)
14. [What v1.0.0 Covers](#14-what-v100-covers)
15. [What Comes Next](#15-what-comes-next)
16. [Glossary](#16-glossary)

---

## 1. What AMIP Is

AMIP is middleware. It sits between an AI system and a robot:

```
AI Brain  -->  AMIP  -->  Robot
               |
               +--> validates the command
               +--> checks it is safe
               +--> sends it to the hardware
               +--> reports back what happened
```

That is it. AMIP does not plan. It does not reason. It does not decide
what the robot should do. The AI does that. AMIP's only job is to make
sure that whatever the AI decides gets executed safely and that the AI
gets an honest account of what actually happened.

I want to be direct about this because it is the most common
misunderstanding when I explain this project: AMIP is not trying to make
the robot smarter. It is trying to make the connection between the AI and
the robot reliable enough that the AI can actually do its job.

---

## 2. The Problem It Solves

Here is what connecting an AI to a robot looks like without something
like AMIP.

You write a command format. The AI outputs something, you parse it, you
translate it into motor signals. You write safety checks — maybe. You
run it in simulation first, then rewrite parts of it for real hardware
because the interfaces are different. The feedback you send back to the
AI is whatever you had time to implement, which usually means the AI gets
told the command succeeded whether it did or not.

Then you want to connect a different AI to the same robot. Or the same
AI to a different robot. And you start over.

AMIP exists to stop that cycle. The goal is that an AI brain written for
one robot should work on any robot that has an AMIP adapter. And any
robot with an AMIP adapter should be driveable by any AI that outputs
AMIP commands. You write the integration once and it composes.

The other problem AMIP solves is trust. When you are running an AI on
real hardware, you need to know with certainty that certain things cannot
happen — the motor cannot be told to spin beyond its rated speed, the
robot cannot be commanded past a limit the hardware cannot physically
handle. Right now that safety logic lives in whatever the developer
remembered to add. AMIP makes it structural. It cannot be bypassed.

---

## 3. Design Principles

These are the principles I came back to repeatedly when making decisions.
When you are unsure whether something belongs in AMIP or not, apply
these.

### Safe by default

No command reaches an actuator without passing through the safety layer.
This is unconditional. There is one exception: the `stop` command, which
bypasses everything because halting the robot must always be possible.
That is the only exception and it is there for obvious reasons.

I have seen projects where safety checks were optional or configurable at
runtime. That approach fails because someone always disables them during
development and forgets to re-enable them. AMIP makes the safe path the
only path.

### AI-agnostic

AMIP does not care what kind of AI is driving it. A simple rule-based
system should work just as well as a sophisticated planning agent or a
language model. AMIP must not assume rationality, must not assume the AI
has accurate world knowledge, and must not assume it is well-behaved.
Some of the most important use cases involve AI that is actively
learning, which means it will deliberately issue commands outside safe
bounds to explore what is possible. AMIP handles that by enforcing
constraints, not by trusting the AI.

### Hardware-agnostic

The same command that drives a TurtleBot should drive a custom Arduino
robot or a Gazebo simulation. The AI does not need to know what is on
the other end. The capability descriptor and adapter handle translation.
Adding a new robot means writing a new adapter and a new descriptor —
nothing in the core changes.

### Honest feedback

AMIP reports what the robot actually did. If the robot moved 0.94 metres
when you asked for 1.0 metre, the feedback says 0.94 metres. It does not
round. It does not report the commanded value. It does not smooth the
number to look cleaner.

The reason is not philosophical — it is practical. An AI planner that
thinks its commands execute perfectly will accumulate positioning error
over time and eventually lose track of where the robot is. A learning
agent that receives adjusted reward signals will learn the wrong model.
Honest feedback is load-bearing, not cosmetic.

### Same code in simulation and on hardware

Switching from a Gazebo simulation to a real robot should require
changing exactly one thing: the adapter that is loaded. The AI code,
the command format, the safety configuration, and the feedback handling
stay identical. If your AI code has `if simulation:` branches in it,
something has gone wrong at the design level.

### Small public surface

The public API is three methods: `execute()`, `get_feedback()`, and
`get_state()`. That is deliberate. The more surface area a library
exposes, the more places there are for breaking changes to happen and
the harder it is for new contributors to understand what matters.

---

## 4. The Theory Behind the Decisions

I want to explain the AI theory that influenced specific design choices,
because some of the decisions look odd until you understand where they
come from. You do not need to have read these sources to contribute —
but if a decision ever seems wrong to you, this section is why it exists.

### Agents

The standard model of an intelligent agent is PEAS: Performance measure,
Environment, Actuators, Sensors. An agent perceives its environment
through sensors and acts through actuators. Its goal is to maximise its
performance measure given what it perceives.

AMIP sits at the actuator boundary:

```
Agent --> [AMIP] --> Actuators --> Environment --> Sensors --> Agent
```

AMIP is not the agent. It has no performance measure, no perception, and
no goals of its own. It is the interface between the agent's decisions
and the physical world. Keeping this boundary clear is important because
it explains why AMIP does not plan, does not reason about goals, and does
not try to be helpful by interpreting ambiguous commands. That is the
agent's job.

### Different AI types need different things

There are five agent types someone might build on top of AMIP, and each
one places different demands on the middleware.

A **simple reflex agent** reacts to the current situation with a
condition-action rule — if temperature exceeds 90 degrees, stop the
motor. It needs AMIP to accept a command and execute it. That is all.

A **model-based agent** maintains an internal model of what it cannot
directly see, updated from feedback over time. It needs AMIP's feedback
to be accurate and consistent, because it uses that feedback to build
and update its world model.

A **goal-based agent** plans a sequence of actions to reach a goal. It
needs to know what actions are available and what their effects are.
AMIP's command schema defines the action space. AMIP's feedback defines
the observable effects.

A **utility-based agent** is trying to maximise some measure — speed,
energy efficiency, or a combination involving trade-offs. It needs AMIP
to report resource consumption accurately: how far was actually
travelled, how long it took, how much battery was used. Without that,
it cannot compute expected utility.

A **learning agent** improves through experience. If the feedback AMIP
gives is distorted, the learning element updates on wrong information
and the agent learns the wrong model. This is the strongest argument
for honest feedback — it is not just bad practice, it actively breaks
the AI.

The key insight: AMIP has to support all five without knowing which one
it is talking to. That is why it is utility-agnostic, goal-agnostic, and
learning-agnostic. It executes commands, enforces constraints, and
reports results. Everything else belongs to the AI.

### Why the safety layer cannot trust the AI

A rational agent chooses actions that maximise expected performance. But
rationality is relative to the agent's knowledge and its performance
measure. A learning agent in early training is exploring the action space
on purpose — it is supposed to try things outside known-safe bounds to
learn where the boundaries are. A utility-based agent might rationally
request a speed that exceeds hardware limits if its performance measure
rewards speed and it has not been given accurate limit information.

Irrational or unsafe commands are not bugs. They are expected outputs
from systems doing exactly what they are designed to do. The safety layer
exists because the hardware has hard limits regardless of what the AI
thinks, and those limits must be enforced at the middleware level
unconditionally.

### Planning and the action space

When an AI planner searches for a sequence of actions to achieve a goal,
it needs four things: the current state, what actions are available, what
each action produces, and what each action costs.

AMIP provides all four. `get_state()` gives the current state. The
command schema defines available actions. Feedback after each `execute()`
shows the resulting state. And feedback includes duration and resource
information the AI can use to compute cost.

One thing worth flagging: AMIP must not enforce optimality. The AI
decides the best action. AMIP executes it safely. If AMIP started
second-guessing commands because a different action would have been more
efficient, it would be doing the AI's job badly while preventing the AI
from doing it at all.

### Uncertainty

Real robots do not execute commands perfectly. Ask a robot to move one
metre and it will move somewhere between 0.9 and 1.1 metres depending
on surface friction, wheel slip, and motor calibration. This is not a
problem to be fixed — it is a property of physical systems that the AI
needs to account for.

AMIP surfaces this uncertainty. The `distance_actual_m` field tells the
AI what actually happened. The simulated adapter includes configurable
odometry noise so AI brains can be tested against realistic imprecision
before touching real hardware. If AMIP collapsed everything to perfect
deterministic values, planners that depend on accurate transition models
would fail in the real world even if they worked perfectly in tests.

### State representation

There are three ways to represent robot state. Atomic representation
treats the whole state as a single opaque symbol — not useful for
planning. Factored representation expresses state as a vector of named
variables, which is what AMIP v1 uses. Structured representation adds
objects and relationships, needed for multi-robot and manipulation tasks,
planned for v2.

Factored representation was chosen for v1 because it is the simplest
representation that preserves the Markov property — the property that
the next state depends only on the current state and action, not on
history. MDP and POMDP planners require this. Without it, the AI cannot
reason correctly about what will happen next.

---

## 5. Architecture

### The pipeline

Every call to `execute()` runs through this sequence:

```
raw dict from AI brain
        |
        v
  CommandValidator     is this a valid AMIP command?
        |
        v
    SafetyLayer        is this safe for this robot?
        |
        v
  HardwareAdapter      send it to the robot
        |
        v
   FeedbackModule      assemble the result and return it
        |
        v
FeedbackPayload to AI brain
```

If anything fails at any stage, the pipeline stops there, assembles a
failure payload, and returns it. `execute()` never raises. The AI always
gets a payload back.

### Module responsibilities

Each module has one job. This is what makes the codebase maintainable
by contributors who do not all know each other.

| Module | Job |
|---|---|
| `exceptions/errors.py` | Define the exception types and error codes |
| `command/schema.py` | Define what commands look like |
| `command/validator.py` | Check that a command is well-formed |
| `hardware/capability.py` | Load and validate the robot descriptor |
| `hardware/adapter.py` | Define the interface all adapters must implement |
| `hardware/sim_adapter.py` | Implement the simulated adapter |
| `safety/layer.py` | Enforce the robot's physical limits |
| `feedback/module.py` | Assemble the result into a structured payload |
| `core.py` | Wire everything together and expose the public API |

### Dependency graph

```
core.py
  |-- command/validator.py
  |     |-- command/schema.py
  |     |-- exceptions/errors.py
  |-- safety/layer.py
  |     |-- command/schema.py
  |     |-- hardware/capability.py
  |     |-- exceptions/errors.py
  |-- hardware/adapter.py
  |     |-- command/schema.py
  |-- hardware/sim_adapter.py
  |     |-- hardware/adapter.py
  |     |-- command/schema.py
  |     |-- exceptions/errors.py
  |-- hardware/capability.py
  |     |-- exceptions/errors.py
  |-- feedback/module.py
        |-- hardware/adapter.py
```

No circular dependencies. Exceptions sit at the base with no dependencies
of their own. That is why they are built first.

---

## 6. Command Protocol

### The unit convention

Every numeric value in AMIP uses SI units. No exceptions, ever.

| Quantity | Unit |
|---|---|
| Distance | metres |
| Angle | radians |
| Linear speed | metres per second |
| Angular speed | radians per second |
| Time | seconds |

Mixed units are one of the most expensive bugs in robotics. If one module
uses centimetres and another uses metres and there is no explicit
conversion, the robot drives ten times as far as intended. SI throughout,
no exceptions.

### Schema versioning

Every command has a `schema_version` field. Omit it and the current
version is assumed. Include it and the major version must match the
running AMIP version — minor version differences are tolerated.

### The four v1 commands

**move**

```json
{
    "schema_version": "1.0",
    "action": "move",
    "direction": "forward",
    "distance_m": 2.0,
    "speed_ms": 0.5
}
```

`direction` is one of `forward`, `backward`, `left`, `right`.
`distance_m` must be positive. `speed_ms` is optional.

AMIP does not guarantee the robot travels exactly `distance_m`.
The feedback reports what actually happened. Plan accordingly.

---

**rotate**

```json
{
    "schema_version": "1.0",
    "action": "rotate",
    "angle_rad": 1.5708,
    "angular_speed_rads": 1.0
}
```

Positive angle is counter-clockwise, following the right-hand rule.
`angular_speed_rads` is optional.

---

**stop**

```json
{
    "schema_version": "1.0",
    "action": "stop",
    "reason": "obstacle detected by AI sensor layer"
}
```

Bypasses all safety checks. Always executes. Use it whenever the AI
needs to halt the robot immediately. The `reason` field is for logs only.

---

**set_speed**

```json
{
    "schema_version": "1.0",
    "action": "set_speed",
    "speed_ms": 0.3,
    "angular_speed_rads": 1.0
}
```

Does not move the robot. Updates the speed that subsequent `move` and
`rotate` commands will use when they do not specify their own.

---

## 7. Capability Descriptor

The descriptor is the contract between the robot and AMIP. It tells AMIP
what the robot can do and what its limits are. Nothing about a robot's
limits is hardcoded in the middleware — it all comes from here.

### Format

```json
{
    "robot_id": "turtlebot3_waffle",
    "amip_descriptor_version": "1.0",
    "mobility": "wheels",
    "sensors": ["camera", "lidar"],
    "supported_actions": ["move", "rotate", "stop", "set_speed"],
    "limits": {
        "max_speed_ms": 0.26,
        "max_angular_speed_rads": 1.82,
        "max_distance_m": 5.0,
        "max_angle_rad": 6.2832
    },
    "environment": "simulation"
}
```

`robot_id`, `supported_actions`, and `limits` are required. Everything
else is optional but recommended.

Unknown fields are ignored. A descriptor written for a future AMIP
version with additional fields will still load in the current version —
the new fields are silently skipped. This is for forward compatibility,
not for bypassing the schema.

---

## 8. Safety Contract

### What AMIP guarantees

No actuator receives a command that has not passed the safety check.
The only exception is `stop`, which always goes through.

Every safety violation is reported with a machine-readable error code,
the value that was requested, and the limit that was exceeded. Telling
the AI "speed limit exceeded" without telling it what the limit is would
be useless — the AI needs enough information to compute a corrected
value and replan.

### What is checked in v1

| Constraint | Error code |
|---|---|
| Distance is not negative | `E_SAFETY_NEGATIVE_DISTANCE` |
| Distance does not exceed `max_distance_m` | `E_SAFETY_MAX_DISTANCE` |
| Speed is not negative | `E_SAFETY_NEGATIVE_SPEED` |
| Speed does not exceed `max_speed_ms` | `E_SAFETY_MAX_SPEED` |
| Angle magnitude does not exceed `max_angle_rad` | `E_SAFETY_MAX_ANGLE` |
| Angular speed does not exceed limit | `E_SAFETY_MAX_ANGULAR_SPEED` |

### Speed clamping

There is an optional `clamp_speeds` mode. When enabled, speeds that
exceed the limit are reduced to the limit rather than rejected. Clamping
is always reported in the feedback — the AI must know its requested speed
was not executed. Silent clamping is not available.

Clamping defaults to off. It is better for the AI to learn the limits
than to have them silently corrected.

### What the safety layer does not do in v1

It does not detect obstacles. It does not enforce no-go zones. It checks
static limits from the descriptor and nothing else. Obstacle avoidance
is planned for v1.2. Do not add dynamic behaviour here.

---

## 9. Feedback Contract

### execute() never raises

An AI control loop runs continuously. If `execute()` could raise, every
call would need a try-except block, and an unhandled exception anywhere
would crash the loop. Instead, every failure is caught inside AMIP and
returned as a failure payload. The AI checks `payload.status` and
decides what to do. The loop keeps running.

### What every payload contains

Every payload, success or failure, contains:

- `status` — `"success"` or `"failure"`, nothing else
- `action` — what was attempted
- `robot_state` — the current factored state of the robot
- `execution` — on success, what actually happened
- `error` — on failure, what went wrong and the machine-readable code

Robot state is always included, even on failure. The AI must always know
where the robot is, including when something went wrong.

### Success payload

```json
{
    "schema_version": "1.0",
    "status": "success",
    "action": "move",
    "execution": {
        "success": true,
        "action": "move",
        "distance_actual_m": 1.94,
        "duration_s": 3.88,
        "obstacle_detected": false
    },
    "robot_state": {
        "position_x_m": 1.94,
        "position_y_m": 0.0,
        "heading_rad": 0.0,
        "speed_ms": 0.5,
        "battery_pct": 99.1
    }
}
```

### Failure payload

```json
{
    "schema_version": "1.0",
    "status": "failure",
    "action": "move",
    "error": {
        "error": "SafetyConstraintViolation",
        "code": "E_SAFETY_MAX_SPEED",
        "message": "Requested speed 2.0 m/s exceeds limit of 1.5 m/s.",
        "constraint": "max_speed_ms",
        "requested": 2.0,
        "limit": 1.5
    },
    "robot_state": {
        "position_x_m": 0.0,
        "position_y_m": 0.0,
        "heading_rad": 0.0,
        "speed_ms": 0.0,
        "battery_pct": 100.0
    }
}
```

### Minimum robot state keys

| Key | Description |
|---|---|
| `position_x_m` | Estimated X position in metres |
| `position_y_m` | Estimated Y position in metres |
| `heading_rad` | Estimated heading in radians |
| `speed_ms` | Current linear speed |
| `battery_pct` | Battery percentage, or null if unavailable |

Adapters may include additional keys. They are passed through unchanged.

---

## 10. Hardware Adapter Plugins

### How they work

An adapter subclasses `BaseHardwareAdapter` and implements two methods.
It translates AMIP's abstract commands into whatever the robot needs —
ROS topics, serial messages, SDK calls, GPIO signals. AMIP does not care
how the adapter works internally, only that it satisfies the contract.

### The contract

**execute(command)**

Run the command. Return an `ExecutionResult` describing what actually
happened. Raise `HardwareAdapterError` on unrecoverable fault. Do not
catch errors and return a success result.

The single most important rule for adapter authors: `distance_actual_m`
and `angle_actual_rad` must reflect what the robot actually did, from
sensors or encoders. If you report commanded values because measuring
actuals is inconvenient, you break the honest feedback guarantee for
every AI brain that ever uses your adapter.

**read_state()**

Return the current factored robot state as a dictionary. This must work
at any time, not only after a command. If the hardware is unavailable,
raise `HardwareAdapterError` — do not return stale data silently.

### ExecutionResult fields

| Field | What it is |
|---|---|
| `success` | True if the command completed without fault |
| `action` | The action string that was executed |
| `distance_actual_m` | Actual distance travelled, from sensors |
| `angle_actual_rad` | Actual angle rotated, from sensors |
| `duration_s` | Wall-clock execution time in seconds |
| `obstacle_detected` | True if a sensor detected an obstacle |
| `error_code` | Error code if success is False |
| `notes` | Optional text for debugging |

### Writing a new adapter

Open an issue first. Describe the robot, its communication interface,
and which descriptor fields it needs. Then subclass `BaseHardwareAdapter`,
implement both methods, write a descriptor, and write tests covering
every supported action, the unsupported action error path, and
`read_state()` before and after execution.

Read `sim_adapter.py` before you start. It is short and covers the
patterns you need.

---

## 11. State Representation

AMIP v1 uses factored representation. Robot state is a flat dictionary
of named variables with scalar values. This is the simplest
representation that preserves the Markov property — the property that
the next state depends only on the current state and action, not on
history. MDP planners require this. Without it, the AI cannot reason
correctly about what will happen next.

AMIP v2 will introduce structured representation for multi-robot and
manipulation tasks. Do not add relational state to v1 modules.

---

## 12. Error Handling

### The hierarchy

```
AMIPError
    CommandValidationError      the command was malformed
    SafetyConstraintViolation   the command was valid but unsafe
    HardwareAdapterError        execution failed at the hardware level
    CapabilityDescriptorError   the descriptor could not be loaded
```

The separation between `CommandValidationError` and
`SafetyConstraintViolation` matters. A validation error means the AI
sent a broken command — fix the code. A safety violation means the AI
sent a valid command that exceeded the robot's limits — replan with
different parameters. These require different responses and must stay
as separate exception types.

### All error codes

| Code | What happened |
|---|---|
| `E_CMD_NOT_DICT` | Input was not a dictionary |
| `E_CMD_MISSING_ACTION` | No `action` field |
| `E_CMD_INVALID_ACTION_TYPE` | `action` was not a non-empty string |
| `E_CMD_VERSION_MISMATCH` | Major schema version incompatible |
| `E_CMD_UNKNOWN_ACTION` | Action not in the registry |
| `E_CMD_UNSUPPORTED_ACTION` | Action not in the descriptor |
| `E_CMD_FIELD_ERROR` | Field had wrong type or value |
| `E_SAFETY_NEGATIVE_DISTANCE` | Negative distance |
| `E_SAFETY_MAX_DISTANCE` | Distance exceeded limit |
| `E_SAFETY_NEGATIVE_SPEED` | Negative speed |
| `E_SAFETY_MAX_SPEED` | Speed exceeded limit |
| `E_SAFETY_MAX_ANGLE` | Angle exceeded limit |
| `E_SAFETY_MAX_ANGULAR_SPEED` | Angular speed exceeded limit |
| `E_HW_ADAPTER_FAULT` | Hardware fault during execution |
| `E_HW_UNHANDLED_ACTION` | Adapter does not handle this action |
| `E_DESCRIPTOR_NOT_FOUND` | Descriptor file does not exist |
| `E_DESCRIPTOR_PARSE_ERROR` | Invalid JSON in descriptor |
| `E_DESCRIPTOR_MISSING_FIELDS` | Required fields missing |
| `E_DESCRIPTOR_VERSION_MISMATCH` | Major descriptor version incompatible |
| `E_DESCRIPTOR_INVALID` | General descriptor error |

### What the AI should do with each category

`E_CMD_*` — There is a bug in the command generation. Fix the structure.
Retrying unchanged will fail again.

`E_SAFETY_*` — The parameters exceeded the robot's limits. The error
payload includes what was requested and what the limit is, so the AI
has enough information to compute a corrected value and replan.

`E_HW_*` — Something went wrong at the hardware level. Call `get_state()`
before deciding what to do next. Do not assume the robot is in the state
it was in before the command.

`E_DESCRIPTOR_*` — AMIP could not initialise. No commands can be
processed until this is resolved. This is a configuration problem, not
a runtime problem.

---

## 13. Versioning

AMIP uses semantic versioning: `MAJOR.MINOR.PATCH`. Breaking changes
to the public API or command schema bump the major version. New
backward-compatible features bump the minor version. Bug fixes that do
not change behaviour bump the patch version.

The command schema version and descriptor version are independent of the
package version. Major versions must match between AI brain commands and
the running AMIP instance. Minor version differences are tolerated.

Error codes are stable within a major version. A code introduced in
v1.0.0 will exist with the same meaning in v1.9.0. Codes can be added
in minor versions but cannot be removed or renamed until the next major
version.

---

## 14. What v1.0.0 Covers

### In scope

- Exception hierarchy with machine-readable error codes
- Versioned command schema with four action primitives
- Capability descriptor loader with JSON v1 schema
- Command validator
- `BaseHardwareAdapter` interface and `ExecutionResult`
- `SimulatedHardwareAdapter` with configurable odometry noise
- Safety layer enforcing static physical limits
- Feedback module with structured payload
- AMIP core orchestrator and three-method public API
- README and CONTRIBUTING

### Not in scope

These will not be accepted as PRs against the v1.0.0 milestone:

- ROS 2 adapter
- Network service or REST API
- Obstacle avoidance and no-go zones
- Local planning
- Structured state representation
- Multi-robot coordination
- CLI interface
- PyPI publication

If you want to work on any of these, open an issue for the relevant
future milestone and we can spec it out there.

---

## 15. What Comes Next

**v1.1.0** adds a network service layer — REST and WebSocket — so
multiple AI brains or diagnostic tools can connect to the same robot
over a local network.

**v1.2.0** extends the safety layer with obstacle avoidance, no-go zone
enforcement, and dynamic environment constraints.

**v2.0.0** introduces structured state representation for manipulation
and multi-robot tasks, along with relational feedback and inter-robot
dependency declarations in descriptors.

---

## 16. Glossary

**Action primitive.** One of the named operations AMIP can execute.
In v1: `move`, `rotate`, `stop`, `set_speed`.

**Adapter plugin.** A class that subclasses `BaseHardwareAdapter` to
add support for a specific robot. The main extension point for the
community.

**AI brain.** Any system that outputs AMIP-compatible commands — a
planner, a learning agent, a language model, a rule-based system, or
a person typing JSON manually.

**Capability descriptor.** The JSON file that declares what a robot
can do and what its limits are. One file per robot configuration.

**Error code.** A stable string like `"E_SAFETY_MAX_SPEED"` that
identifies a specific failure condition programmatically.

**Factored representation.** Expressing state as a flat vector of named
scalar variables rather than a single symbol or a relational graph.

**Honest feedback.** The AMIP principle that feedback reports what the
robot actually did, not what was commanded.

**Markov property.** The property that the next state depends only on
the current state and action, not on history. Required for MDP planning.

**Middleware.** Software that mediates between two systems. AMIP is
middleware between AI and robot hardware.

**Odometry error.** The difference between a commanded movement and the
actual movement, caused by physical imprecision in actuators and sensors.

**Safe by default.** The AMIP guarantee that no command reaches an
actuator without passing the safety layer.

**Sim-real consistency.** The AMIP guarantee that the same AI code and
commands work identically in simulation and on real hardware.

**Transition model.** A function describing what state results from
taking an action in a given state. Written `T(s, a) -> s'`.
