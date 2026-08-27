# CAPABILITY-13 — Consensus Authentication Protocol [![Org](https://img.shields.io/badge/Org-CommonIntents--144-darkgray.svg)](https://github.com/CommonIntents)

**What you see is what you sign.** CAPABILITY-13 provides a verifiable confirmation mechanism based on Cellrix's deterministic visual rendering.

CAPABILITY-13 defines the structure of user confirmation events — carrying the visual content hash of the Cellrix interface that the user actually saw and agreed to. This bridges the semantic gap between AI intent and human authorization.

## What CAPABILITY-13 Defines
- **Confirmation Event** — structured event carrying Cellrix view hash + user action
- **Decision Queue** — asynchronous HITL (human-in-the-loop) state machine
- **Consensus Anchor** — the Cellrix-rendered interface is the ground truth for both human and AI

## Core Philosophy
> *Not declared, means it does not exist.*
> *Extensions not activated mean zero overhead.*

## Protocol Stack
```
Cellrix (visual consensus layer)
  ↑
CAPABILITY-13 ← You are here (consensus confirmation)
  ↑
INTENT-7 (intent syntax — SIDL)
  ↑
BIND-19 (transport binding)
  ↑
INTENT-7-SECURE (optional mTLS transport)
```

## Read the Spec
- [CAPABILITY-13 v1.0.0-RFC-4](spec/CAPABILITY-13.md)
- [中文版](spec/CAPABILITY-13.zh-CN.md)

## Related
| Protocol | Repository |
|:---|:---|
| INTENT-7 | [CommonIntents/INTENT-7](https://github.com/CommonIntents/INTENT-7) |
| BIND-19 | [CommonIntents/BIND-19](https://github.com/CommonIntents/BIND-19) |
| INTENT-7-SECURE | [CommonIntents/INTENT-7-SECURE](https://github.com/CommonIntents/INTENT-7-SECURE) |

## License
Apache 2.0 — see [LICENSE](LICENSE).
