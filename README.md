# OpenROC

### The Language of Physical AI

**Open Robot Operation Commands** — the open protocol that connects AI intelligence to robots,
actuators, and physical systems through standardized commands.

`openroc-beta-1.0`

[Website](https://openroc.6com.group) · [Specification](https://openroc.6com.group/spec/) · [Download PDF](https://openroc.6com.group/OpenROC_Specification.pdf)

---

## Table of contents

- [What is OpenROC?](#what-is-openroc)
- [Why OpenROC](#why-openroc)
- [Ecosystem](#ecosystem)
- [Planes and lifecycle](#planes-and-lifecycle)
- [Encodings](#encodings)
- [OpenROC-Kit](#openroc-kit)
- [Quick start](#quick-start)
- [Authoring a schema](#authoring-a-schema)
- [Integrating with a robot](#integrating-with-a-robot)
- [Contributing](#contributing)
- [License](#license)

---

## What is OpenROC?

OpenROC is an open, transport-agnostic protocol for **Physical AI**: the layer where learned
intelligence becomes physical action. It standardizes how *why*, *what*, and *how* are expressed
as messages that travel between planners, controllers, and hardware.

A planner reasons about objectives. A controller turns those into tasks and trajectories. A motor
driver turns those into force and motion. OpenROC defines one vocabulary for all three, so a new
planning model can replace an old one without touching motion code, and the same intent can run on
different machines by swapping the adapter.

The protocol is defined by a versioned specification (`roc`, e.g. `openroc-beta-1.0`), published
on the [website](https://openroc.6com.group/spec/) and as a
[PDF](https://openroc.6com.group/OpenROC_Specification.pdf).

## Why OpenROC

- **Transport-agnostic.** Messages are self-delimiting and travel over any link — UDP, CAN,
  EtherCAT, shared-memory rings, or a plain byte stream. Discovery maps logical addresses to real
  transports.
- **Hardware-neutral.** Commands address logical endpoints (`robot.arm-1.joint-3`), not network
  locations or vendor APIs.
- **Layered by design.** Action and Actuation are separate planes with explicit boundaries, so
  intent and control evolve independently.
- **Two encodings, one meaning.** Human-readable OpenROC-JSON and compact OpenROC-BIN are
  semantically identical and losslessly convertible.
- **Canonical and deterministic.** Messages have a canonical form, so signatures and digests are
  stable across implementations and encodings.
- **Versioned and extensible.** A `MAJOR.MINOR` version plus an extension namespace let the
  protocol grow without breaking conformant receivers.

## Ecosystem

All components belong to the **OpenROC** ecosystem. The protocol is called **OpenROC** (not ROC),
and there is no separate *OpenROC Decision* or *OpenROC Motion* component — decisions, objectives,
movement, and trajectory operations all live in OpenROC Action and OpenROC Actuation.

| Component | Role |
| --- | --- |
| **OpenROC Core** | Message structure, command semantics, and execution lifecycle. |
| **OpenROC Action** | Objectives, decisions, high-level robot tasks, skills, movements, and trajectories — *what to do, and why*. |
| **OpenROC Actuation** | Low-level device control — *how*. |
| **OpenROC-JSON** | Human-readable encoding for tooling, debugging, and configuration. |
| **OpenROC-BIN** | Compact binary encoding for high-rate and constrained links. |
| **OpenROC Adapters** | Integration with existing robots, actuators, and middleware. |

## Planes and lifecycle

Every OpenROC message is an **envelope** (identity, addressing, classification, timing, integrity)
carrying a **plane-specific payload**.

| Plane | Question | Output |
| --- | --- | --- |
| `action` | *What to do, and why?* | Objectives, tasks, skills, trajectories. |
| `actuation` | *How, at the hardware?* | Setpoints, modes, limits, timing. |

Commands move through a shared lifecycle — `issued → received → admitted → validated → scheduled
→ executing → completed`, with `failed`, `cancelled`, `expired`, and `preempted` as terminal
states. `ack` and `event` messages report progress; `correlation` links a conversation together
and `id` makes re-delivery idempotent.

Read more in the [Core](https://openroc.6com.group/spec/#core),
[Action Stack](https://openroc.6com.group/spec/#action-stack), and
[Actuation Stack](https://openroc.6com.group/spec/#actuation-stack) chapters.

## Encodings

| Encoding | Use it for | Shape |
| --- | --- | --- |
| **OpenROC-JSON** | Development, debugging, config, test vectors, low-rate telemetry. | Strict UTF-8 JSON profile with canonical output; NDJSON for streams. |
| **OpenROC-BIN** | High-rate control, constrained links, deterministic framing. | Framed (`MAGIC`, version, flags, varint length), CRC-32C, LEB128 varints, dictionary opcodes, fixed-point numerics. |

The two are losslessly convertible: `json -> bin -> json` yields the canonical input. See
[OpenROC-JSON](https://openroc.6com.group/spec/#json) and
[OpenROC-BIN](https://openroc.6com.group/spec/#bin).

---

## OpenROC-Kit

**OpenROC-Kit** (`openroc-kit`, version `openroc-beta-1.0`) is the Rust toolkit for building with
OpenROC. It is schema-driven: you describe your robot in YAML, the kit compiles it to canonical
JSON Schema, and generates typed codecs, validators, and test vectors so you can start developing
immediately instead of hand-writing codecs.

> OpenROC-Kit is in active development; the interface below reflects the `openroc-beta-1.0`
> proposal.

It is usable as a **CLI** in CI and on a lab bench, as a **library** inside controllers and
adapters, and as a **C ABI** for firmware and non-Rust runtimes. The same rules run everywhere —
what CI checks is what the device enforces.

**Capabilities**

- **Schema authoring.** Write schemas for your robot's actions in readable YAML — the single
  source of truth the rest of the toolkit builds on.
- **Compile.** `schema compile` verifies the bundle and emits canonical JSON Schema 2020-12 for the
  application layer.
- **Codegen.** Generate serde-compatible Rust types and C structs, JSON and BIN codecs, and
  per-verb validators, plus an `openroc.h` header.
- **Encode / decode.** Convert losslessly between OpenROC-JSON and OpenROC-BIN, with canonical
  output for signatures and digests.
- **Validation diagnostics.** Every rejection is a structured diagnostic with a stable reason code
  and a JSON Pointer or byte offset.
- **Embeddable.** `std` for hosts, `no_std` (allocation-free) for firmware, and a stable C ABI.
- **Authoring loop.** `dev --watch` re-lints, re-generates, and re-checks on every save.

## Quick start

### CLI

```bash
cargo install openroc-kit

# Scaffold a robot project
openroc-kit init my-robot --template arm
cd my-robot

# Author, check, and generate
openroc-kit schema new action pick --version openroc-beta-1.0
openroc-kit schema lint
openroc-kit schema compile                       # YAML -> canonical JSON Schema
openroc-kit schema codegen --lang rust,c         # typed codecs + validators + openroc.h
openroc-kit dev --watch                          # tight local loop
```

### As a Rust dependency

```toml
# Host / server
openroc-kit = "openroc-beta-1.0"

# Firmware: no heap, BIN only
openroc-kit = { version = "openroc-beta-1.0", default-features = false, features = ["bin", "schema-embed"] }
```

## Authoring a schema

A schema is an ordinary JSON Schema plus an `x-openroc` block binding it to a plane, verb, and
skill. Files are named for the skill they describe.

```yaml
# schemas/action/pick.yaml
title: Action.pick
type: object
required: [object]
additionalProperties: false
x-openroc:
  plane: action
  verb: execute
  skill: pick
  version: "openroc-beta-1.0"
  roc: "openroc-beta-1.0"
  requires_capabilities: [grasp, force_control]
properties:
  object:
    type: string
  grasp_pose:
    $ref: "common.yaml#/$defs/pose"
  force:
    type: object
    required: [value, unit]
    properties:
      value: { type: number, minimum: 0 }
      unit: { const: N }
```

One robot can have many actions: each skill is its own schema file, and the robot's capability
record lists the set it implements.

```json
{
  "address": "robot.arm-1",
  "kind": "manipulator",
  "skills": ["pick", "place", "move_to", "follow", "home"],
  "actuators": ["robot.arm-1.joint-1", "robot.arm-1.joint-2", "gripper-a"],
  "limits": { "max_payload": "5 kg", "max_speed": "1.5 m/s", "max_force": "40 N" }
}
```

## Integrating with a robot

The pattern is always **decode → validate → translate or dispatch**, with the same rules on every
target.

- **ROS 2 (Linux).** A bridge node translates between ROS 2 and OpenROC using the Rust library or
  the C ABI from C++.
- **RTOS / MCU (Zephyr, FreeRTOS).** `no_std`, BIN-only, allocation-free frame validation inside a
  control task or ISR.
- **Linux-based robot.** A `std` host service that consumes NDJSON commands and forwards BIN to the
  control layer.

---

## Contributing

OpenROC is an open protocol, and contributions are welcome — specification clarifications, schema
and adapter designs, reference implementations, and conformance test vectors.

1. Read the [chapters](https://openroc.6com.group/spec/) for context.
2. Open an issue to discuss substantial changes before sending a pull request.
3. Keep proposals aligned with the OpenROC naming rule: the protocol is **OpenROC**, and movement
   and trajectory operations belong in OpenROC Action and OpenROC Actuation.

## License

OpenROC is released under the MIT License.

---

**OpenROC — The Language of Physical AI.**
