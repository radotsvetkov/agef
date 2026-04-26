# AGEF v0.1 Specification

## 1. Status and Versioning

This document defines **AGEF v0.1** (Agent Governance Evidence Format), a pre-stable format for portable, tamper-evident AI agent session evidence.

This specification is versioned independently from any implementation. Repositories MAY tag this text as `v0.1.0-seed` while the wire format version remains `v0.1`.

Per pre-stable policy, **v0.x MAY introduce breaking changes**. Readers and writers MUST check `agef_version` and reject unsupported versions.

## 2. Conformance Language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be interpreted as described in RFC 2119.

## 3. Scope and Non-Goals

### 3.1 Scope

AGEF defines a format for recording one AI-agent session as a portable bundle with:
- content-addressed objects,
- ordered event records,
- verifiable parent linkage,
- offline integrity verification.

### 3.2 Non-Goals

AGEF does **not** define:
- agent runtime behavior,
- model correctness guarantees,
- identity/signing standards for actor attribution (signing is expected in later versions).

## 4. Core Terms

- **Session**: logical run from `SessionStart` to `SessionEnd`.
- **Event**: one structured record in session order.
- **Object**: immutable blob addressed by content hash.
- **Head**: hash of the terminal session event.
- **Bundle**: portable archive containing manifest, events, and objects.

## 5. Bundle Format

An AGEF bundle **MUST** be a `tar.zst` archive containing exactly these top-level paths:

- `manifest.json`
- `events.bin`
- `objects/` (directory containing one file per object hash)

Additional files **MAY** be present, but verifiers **MUST** ignore unknown non-normative files unless explicitly configured to reject them.

## 6. Manifest (`manifest.json`)

`manifest.json` **MUST** be UTF-8 JSON with LF line endings and sorted object keys.

It **MUST** contain:

- `agef_version` (string, required; for this spec: `"0.1"`)
- `producer` (object, required)
  - `name` (string, required)
  - `version` (string, required)
- `session` (object, required)
  - `id` (UUIDv4 string, required)
  - `head` (hash string, required)
  - `created_at` (RFC3339 timestamp, required)
  - `ended_at` (RFC3339 timestamp, required)
- `hash_algorithm` (string, required; `"sha256"` default in v0.1, `"blake3"` optional)
- `object_count` (integer >= 0, required)
- `event_count` (integer >= 0, required)

Writers **MUST** ensure `event_count` and `object_count` match actual bundle contents.
Readers **MUST** reject malformed or incomplete required fields.

## 7. Events Stream (`events.bin`)

`events.bin` **MUST** contain an ordered sequence of event records using **length-delimited CBOR framing**:

For each event record:
1. a 4-byte unsigned big-endian length prefix `N`,
2. followed immediately by `N` bytes of canonical CBOR representing one `Event`.

Rationale: length-delimited framing supports partial recovery from truncated files and deterministic scanning.

Readers **MUST** reject records with invalid length prefix, truncated CBOR payload, or non-canonical encoding where canonical encoding is required.

## 8. Event Envelope

Each event **MUST** encode the following fields:

- `parents`: array of `Hash` (zero or more parent event hashes)
- `kind`: `EventKind` (see Section 9)
- `emitted_at`: timestamp
- `sequence`: integer, monotonic per session, starting at 0

### Envelope Rules

- `sequence` **MUST** start at 0 and increase by exactly 1 per event.
- Every event except `SessionStart` **MUST** have at least one parent.
- `SessionStart` **MUST** have exactly zero parents.
- In v0.1, every non-`SessionStart` event **MUST** have exactly one parent: the immediately preceding event by `sequence`. Multi-parent events are reserved for future versions.
- `parents` entries **MUST** reference previously seen event hashes in the same bundle.
- Event hash **MUST** be computed over canonical CBOR bytes of the full event envelope.
- Event ordering in `events.bin` **MUST** match `sequence`.

## 9. Event Kinds (v0.1)

Readers **MUST** recognize exactly the event kinds listed below for v0.1.

### 9.1 `SessionStart`

Fields:
- `cwd_hash`
- `config_hash`

### 9.2 `UserTurn`

Fields:
- `prompt_hash`

### 9.3 `ProviderCall`

Fields:
- `provider_id`
- `attempts` (array of `AttemptRecord`, required, length >= 1)
- `stream_hash` (optional)

`attempts` **MUST** preserve chronological attempt order.

#### AttemptRecord

Fields:
- `attempt_number` (u32, 1-indexed)
- `started_at`
- `ended_at`
- `status` (`AttemptStatus`)
- `request_hash`
- `response_hash` (optional)
- `stream_hash` (optional)
- `error_message` (optional)

`attempt_number` **MUST** begin at 1 and increase by exactly 1 per attempt in the same `ProviderCall`.

#### AttemptStatus (closed set)

`AttemptStatus` **MUST** be exactly one of:

- `Success`
- `RateLimited` (HTTP 429 or provider-specific rate-limit signal)
- `NetworkError` (DNS, connection refused, TLS, timeout, similar transport failures)
- `ServerError` (HTTP 5xx)
- `ClientError` (HTTP 4xx except 429, e.g. auth or malformed request)
- `Cancelled` (aborted by caller)
- `Other(string)` (escape hatch with free-form reason)

Writers **MUST NOT** emit additional status variants in v0.1.
Readers **MUST** reject unknown variants.

### 9.4 `ToolCall`

Fields:
- `tool_id`
- `input_hash`
- `output_hash`
- `side_effects_hash` (optional)

### 9.5 `RetrievalCall`

Fields:
- `index_id`
- `query_hash`
- `results_hash`

### 9.6 `PermissionGate`

Fields:
- `policy_id`
- `decision`
- `context_hash`

`decision` **MUST** be a string. v0.1 defines no closed enum. Producers **SHOULD** use lowercase verbs (for example: `allowed`, `denied`, `deferred`) but **MAY** use producer-specific values. Future versions may close this set.

### 9.7 `AssistantTurn`

Fields:
- `message_hash`
- `tool_calls_hash` (optional)

### 9.8 `SessionEnd`

Fields:
- `summary_hash` (optional)

## 10. Objects Directory (`objects/<hex>`)

Each object referenced by any event hash field **MUST** exist as a file under `objects/<hex>` where `<hex>` is the lowercase hex digest for the active `hash_algorithm`.

Rules:
- Object filenames **MUST** be hash digests.
- Object bytes **MUST** hash to their filename digest.
- Objects are opaque bytes; AGEF does not require object MIME metadata in v0.1.

## 11. Hashing Rules

v0.1 default algorithm is `sha256`.

Writers:
- **MUST** set `manifest.hash_algorithm`.
- **MUST** use that algorithm consistently for event linkage and object addressing.

Readers:
- **MUST** support `sha256`.
- **MAY** support `blake3`.
- **MUST** reject bundles declaring unsupported `hash_algorithm`.

Within CBOR-encoded events, hashes **MUST** be encoded as CBOR byte strings (major type 2) of length 32 for SHA-256 and BLAKE3. Hex string representation is used only in `manifest.json` (`session.head`) and in object filenames.

## 12. Serialization Rules

- Event payloads in `events.bin` **MUST** use canonical CBOR (RFC 8949 canonical form).
- `manifest.json` **MUST** be UTF-8 JSON with LF endings.
- JSON object keys in `manifest.json` **MUST** be sorted.
- Canonical encoding requirements apply to any structured CBOR payload used for event hashing.
- Timestamps in CBOR-encoded events **MUST** use CBOR tag 1 (epoch-based time, number), expressed as integer seconds since Unix epoch in UTC. Sub-second precision **MAY** be encoded as a floating-point number under the same tag. Timestamps in `manifest.json` **MUST** use RFC3339 strings.

## 13. Verification Procedure

Given a bundle, verifier **MUST**:

1. Extract archive.
2. Parse `manifest.json`; reject on schema/version failure.
3. Read `events.bin` using 4-byte length-delimited framing.
4. For each event:
   - decode canonical CBOR,
   - recompute event hash,
   - verify `sequence` monotonicity,
   - verify all `parents` resolve to previously seen events,
   - verify all referenced content hashes resolve to files in `objects/`.
5. For each referenced object:
   - read bytes,
   - hash bytes with `manifest.hash_algorithm`,
   - compare to filename digest.
6. Confirm:
   - `manifest.event_count` equals decoded event count,
   - `manifest.object_count` equals object file count,
   - `manifest.session.head` equals terminal event hash,
   - `SessionStart` is reachable ancestor of head.

Verifiers **MUST** fail on first invariant violation in default operation. Verifiers **MAY** offer an optional "report-all mode" that continues past failures and emits a complete diagnostic report; behavior in this mode is implementation-defined.

## 14. Rejection Rules

A reader/verifier **MUST** reject if any of the following occur:

- unsupported `agef_version`,
- unsupported `hash_algorithm`,
- missing required manifest fields,
- invalid UUID or timestamp syntax in required manifest fields,
- malformed `events.bin` framing,
- non-canonical CBOR where canonical is required,
- unknown `EventKind`,
- unknown `AttemptStatus`,
- broken parent linkage,
- missing referenced object,
- object hash mismatch,
- sequence gaps or duplicates,
- head mismatch against manifest.

A reader **MAY** expose partial diagnostics for corrupted bundles but **MUST NOT** claim successful verification unless all required checks pass.

## 15. Compatibility and Evolution

- v0.x is pre-stable: breaking changes are permitted.
- Future versions **SHOULD** preserve forward migration guidance.
- v1.0 is intended as first stable major.
- New event kinds in future majors **MUST** be version-gated.
- v0.1 readers **MUST NOT** silently ignore unknown required semantics.

## 16. Security Considerations

AGEF provides tamper-evidence and portability; it does not provide identity attribution by itself.

Implications:
- If producer trust is required, external signing **SHOULD** be applied.
- Bundles may contain sensitive content; storage and sharing controls **MUST** be handled by operators.
- Verification checks integrity, not semantic correctness of model/tool outputs.

## 17. Implementation Notes (Non-Normative)

Known libraries often used for canonical CBOR implementations:

- Rust: `ciborium`
- Go: `fxamacker/cbor`
- Python: `cbor2`
- JavaScript/TypeScript: `cbor-x`

These are known to produce RFC 8949 canonical encoding when configured correctly. Implementers should validate canonical-encoding behavior with test vectors before claiming conformance.

## 18. Licensing Note for Spec Text

The normative spec text in `SPEC.md` is intended for CC BY 4.0 licensing.
Reference implementations may use different licenses.
