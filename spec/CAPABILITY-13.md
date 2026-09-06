# CAPABILITY-13: Capability Authentication and Asynchronous Human-In-The-Loop (HITL) Specification (v1.0.0-RFC-4)
> © 2026 CommonIntents. Licensed under CC BY-ND 4.0 (https://creativecommons.org/licenses/by-nd/4.0/).

## 1. Introduction and Objectives

This specification defines **CAPABILITY-13**, the standard for AI-native capability authentication, dynamic security scope mapping, and asynchronous, non-blocking Human-In-The-Loop (HITL) consensus within the **CommonIntents-144 (CI-144)** suite.

In complex, multi-agent co-biotic environments, static compile-time security controls lead to system rigidity, while unconstrained autonomy leads to catastrophic payload execution or secret leakage. CAPABILITY-13 solves this by:
- Defining an asynchronous, token-based challenge-response protocol for HITL validation, preventing FSM thread lockups or resource starvation.
- Establishing a dynamically updateable, cryptographically signed permission mapping registry (`capability_mapping.toml`).
- Standardizing the direct boundary mapping between logical intent scopes and physical execution sandbox permissions (such as Tentacle Manifests).

---

## 2. Dynamic Permission Mapping Registry

Rather than hardcoding security mappings inside the compilation binary of the executor (Anaphase), CAPABILITY-13 delegates authority mapping to a dynamic registry file: `capability_mapping.toml`.

### 2.1 Schema Structure

```toml
# config/capability_mapping.toml
# Dynamic permission mapping signed via Ed25519

[meta]
version = "1.0.0"
updated_at = "2026-06-26T12:00:00Z"
signer_l0_hash = "f3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca4..." # Creator's L0 Genesis Key Hash

[mappings.standard_scopes]
"read" = ["filesystem:read", "memory:read"]
"write" = ["filesystem:write", "memory:write"]
"execute" = ["execute:wasm", "execute:js"]
"external_network" = ["network:outbound"]

[mappings.custom_scopes]
"media_control" = ["system:audio", "execute:media_player"]
"scientific_simulation" = ["compute:float_matrix", "filesystem:read:/opt/data"]

[integrity]
algorithm = "Ed25519"
# Cryptographic signature covering all [mappings] blocks, signed by Creator's private key
signature = "8b625cf0a359873d2efba93de89fa1237de1e604f378d389fe39a1c890bfca12f9..."
```

### 2.2 Boot-up Cryptographic Verification Chain

During the initialization phase of the executor (Anaphase), the system MUST perform a DNSSEC-style signature check:

1.  **Extract Mappings Block**: Read the raw byte array of all `[mappings]` blocks in the configuration file.
2.  **Resolve Creator Key**: Query the system's trusted **L0 Controller** (via the standardized L0 gRPC interface over local UDS) for the current trusted Creator Public Key matching the L0 Gene Lock bloodline hash.
3.  **Decrypt and Verify**: Validate the signature inside the `[integrity].signature` block against the extracted mappings payload.
4.  **Fail-Secure Lockout**: If the signature verification fails, or the `signer_l0_hash` mismatches with current L0 identity, Anaphase MUST immediately throw error code `0x05FA` (`INVALID_CONFIGURATION_SIGNATURE`), refuse to start, and completely lock down all execution sandboxes.

---

## 3. Asynchronous Non-Blocking HITL Challenge-Response Protocol

When an action is flagged as high-risk or a critical permission is invoked, CAPABILITY-13 mandates a non-blocking asynchronous suspension flow, utilizing the BIND-19 `CON` (`0x02`) flag.

```
                  Anaphase Intercepts High-Risk Action
                                   │
                                   ▼
                [ Step 1: Suspend Task & Generate Token ]
          - Create Pending_HITL_Token (mapped to correlation_id)
          - Yield current task FSM thread to avoid resource lockups
                                   │
                                   ▼ (Stream token to UI Viewport)
                [ Step 2: Visual and Physical Challenge ]
          - Interface displays "Consensus Required" visual lock
          - Wait for human creator to physically sign payload
                                   │
                                   ▼ (Return signed Response_HITL_Token)
                [ Step 3: Verify and Resume Execution ]
          - Verify cryptographic signature against L0 key
          - Resume suspended FSM context, execute tool, commit 2PC
```

### 3.1 `Pending_HITL_Token` Payload Specification

```json
{
  "correlation_id": "4bf92f35-77b3-4da6-a3ce-929d0e0e4736",
  "challenge_hash": "sha256_hash_of_target_command_and_parameters",
  "requested_permissions": ["network:outbound", "filesystem:write"],
  "timeout_ms": 900000,
  "timestamp": 1782376405
}
```

#### Fields Description
- `correlation_id`: Optional string. Used for end-to-end tracing and transaction correlation across protocol layers.
- `challenge_hash`: String. Cryptographic SHA-256 hash of the target command payload.
- `requested_permissions`: Array of strings. Specific physical permissions being requested.

### 3.2 Non-Blocking State Machine Transition

- **Yield Thread**: Upon issuing the `Pending_HITL_Token`, the FSM immediately yields the active thread. It does NOT block the OS thread or spawn infinite wait loops.
- **Task Scheduling**: The scheduler context-swaps the current task to `WAITING` status and allows other pending tasks in the execution queue to consume CPU cycles.
- **Verification & Wake**: Once the signed `Response_HITL_Token` is pushed back via BIND-19 `Control (0x03)` frame, Anaphase's reactor verifies the signature. On success, it moves the task back to `ACTIVE`, restoring the execution context seamlessly.

---

## 4. Standard Capability Mapping Profiles

Every execution sandbox (e.g., Tentacle WASM) MUST declare its requested permissions in its static Manifest. CAPABILITY-13 enforces the following definitive, granular boundary mapping:

| Logical Scope | Tentacle Manifest Capability | Sandbox Enforcement Behavior |
|:---|:---|:---|
| **`read`** | `"filesystem:read"` | Grants read-only access to specific chroot directories. |
| **`write`** | `"filesystem:write"` | Grants write access ONLY inside the temporary CoW sandbox layer. |
| **`execute`** | `"execute:wasm"` / `"execute:js"` | Spawns sandboxed WASM/QuickJS threads with memory and CPU boundaries. |
| **`external_network`** | `"network:outbound"` | Allows outbound TCP connections. Outbound calls MUST route via the local Tuck proxy. |
| **`admin`** | `"admin:gene_lock"` | Allows sending administrative reload instructions to the Mind L0 controller. |

---

## 5. Creator Key Rotation and Revocation Protocol

To safeguard against private key leakages, CAPABILITY-13 integrates with the L0 controller to support GPG-style asymmetric key rotation:

1.  **Rotation Submission**: The owner submits a signed `KeyRotationTransaction` to the administrative channel:
    ```json
    {
      "previous_key_hash": "sha256_of_current_signing_key",
      "new_public_key": "raw_ed25519_public_key_string",
      "rotation_proof": "ed25519_signature_by_previous_private_key(new_public_key_hash)"
    }
    ```
2.  **Proof of Life Verification**: Anaphase compiles this rotation transaction and uses the **old public key** to verify the `rotation_proof`.
3.  **Atomic Rollover**: Upon success, `Anaphase` replaces the active public key with `new_public_key` in its memory, immediately reloads and verifies the current `capability_mapping.toml` under the new signature, and logs this event.
4.  **Key Loss Recovery**: If the private key is physically lost, a full revision of the L0 `gene_lock.md` file is required. This alters the Genesis L0 bloodline hash, forcing the system to perform a new identity registration on the network.

---

## 6. Error Codes

When a capability authentication or verification failure occurs, the execution layer MUST halt and return the following diagnostic codes:

| Error Code | Hexadecimal | Name | Description |
|:---|:---|:---|:---|
| `1408` | `0x0580` | `PERMISSION_DENIED` | Sandbox requested a capability not declared in its Manifest. |
| `1500` | `0x05DC` | `CONSENSUS_UNAVAILABLE` | External CAP/HITL consensus engine is offline or unreachable. |
| `1530` | `0x05FA` | `INVALID_CONFIGURATION_SIGNATURE` | Dynamic permission registry mapping file failed signature verification. |
| `1531` | `0x05FB` | `UNAUTHORIZED_KEY_ROTATION` | Key rotation transaction signature failed verification against active key. |
| `1532` | `0x05FC` | `CONSENSUS_TIMEOUT` | HITL token expired without receiving a valid signed response. |

