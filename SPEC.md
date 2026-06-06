# AGEF v0.1.3 Specification

## 1. Status and Versioning

This document defines **AGEF v0.1.3** (Agent Governance Evidence Format), a pre-stable format for portable, tamper-evident AI agent session evidence.

This specification is versioned independently from any implementation. Repositories MAY tag this text as `v0.1.3-seed`. A bundle's `manifest.json` `agef_version` (three-part semantic version; see Section 6) **MUST** name the highest feature layer the bundle uses: `"0.1.1"` for the baseline bundle format, `"0.1.2"` when the bundle carries detached signatures (`manifest.signatures[]`, Section 19), or `"0.1.3"` when it carries operator attestations (`manifest.operator_attestations[]`, Section 20). When a bundle already carries a head signature (Section 19), its `agef_version` is pinned by the `AGEF-SIG-v1` statement, which commits to that value; adding later additive metadata such as operator attestations (Section 20) **MUST NOT** change it, as that would invalidate the existing signature.

AGEF v0.1.2 and v0.1.3 are **additive minors**: the `signatures` and `operator_attestations` envelopes are OPTIONAL, and a bundle that omits both is byte-for-byte a valid v0.1.1 bundle. Within the `0.1.x` line, a reader **MUST** accept any `0.1.x` `agef_version` it does not specifically recognize and **MUST** ignore unknown `manifest.json` fields. A v0.1.1 or v0.1.2 reader therefore reads a v0.1.3 bundle unchanged, ignoring fields it does not recognize.

Per pre-stable policy, **v0.x MAY introduce breaking changes** across minor lines. Readers and writers **MUST** check `agef_version` and reject values outside a `major.minor` line they support.

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
- a full signature or cryptographic-identity *standard*. AGEF v0.1.2 adds an OPTIONAL detached-signature envelope over the session head (`manifest.signatures[]`; see Section 19), but it mandates no PKI, no key-distribution mechanism, no keyless/Sigstore signing, and no transparency log. Public-key trust is established out of band.
- a human-identity, PKI, or DID standard for operator accountability. AGEF v0.1.3 adds an OPTIONAL operator-attestation envelope (`manifest.operator_attestations[]`; see Section 20) that binds a self-asserted operator identity to a session, but the binding is a separately-signed claim with out-of-band key trust — no PKI, DID, directory, or cloud service is required or defined.

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

- `agef_version` (string, required): **MUST** be a three-part semantic version string (`major.minor.patch`). For this specification: `"0.1.1"`.
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

`manifest.json` **MAY** additionally contain optional fields. Readers **MUST** ignore any field they do not recognize (forward compatibility within the `0.1.x` line). The following optional fields are defined:

- `signatures` (array, optional; AGEF v0.1.2): detached signatures over the session head. Absence means the bundle is unsigned. Each entry is an object with:
  - `scheme` (string, required): signature scheme; `"ed25519"` in v0.1.2.
  - `key_id` (string, required): lowercase-hex SHA-256 digest of the raw 32-byte public key.
  - `statement_version` (string, required): `"AGEF-SIG-v1"`.
  - `signature` (string, required): lowercase-hex detached signature bytes.
  - `created_at` (RFC3339 timestamp, required): self-reported signing time; **NOT** part of the signed statement.

  Signatures are manifest metadata, outside the event hash chain (Section 19). Adding or counter-signing entries does not change the head or any event/object hash. Multiple entries are permitted (counter-signing).

- `operator_attestations` (array, optional; AGEF v0.1.3): operator/human-accountability attestations bound to the session head. Absence means the session is **unattributed** (still fully valid). Each entry is an object with:
  - `scheme` (string, required): `"ed25519"` in v0.1.3.
  - `key_id` (string, required): lowercase-hex SHA-256 digest of the raw 32-byte operator public key.
  - `statement_version` (string, required): `"AGEF-OPERATOR-v1"`.
  - `operator_id` (string, required, non-empty): self-asserted operator identifier.
  - `display_name` (string, optional, default `""`): self-asserted display name.
  - `role` (string, optional, default `""`): self-asserted role.
  - `org` (string, optional, default `""`): self-asserted organization.
  - `signature` (string, required): lowercase-hex detached signature bytes.
  - `created_at` (RFC3339 timestamp, required): self-reported attestation time; **NOT** part of the signed statement.

  `operator_id`, `display_name`, `role`, and `org` are all covered by the signature and **MUST NOT** contain `\n` or `\r`. Attestations are manifest metadata, outside the event hash chain, and do not affect `signatures[]` or the `AGEF-SIG-v1` head statement (Section 20). Multiple entries are permitted (e.g. operator plus reviewer/approver).

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
- In v0.1, every non-`SessionStart` event **MUST** have exactly one parent, and that parent **MUST** equal the hash of the immediately preceding event by `sequence`. Multi-parent events are reserved for future versions.
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
- Timestamps in CBOR-encoded events **MUST** use CBOR tag 1 (epoch-based time, number).
- Producers **MAY** emit tag 1 timestamps as either integer epoch seconds (UTC) or floating-point values for sub-second precision.
- Readers **MUST** accept both integer and floating-point tag 1 timestamp encodings.
- The `akmon-journal` v0.1 reference implementation emits integer epoch seconds.
- Implementations **MAY** use any internal storage format; AGEF canonical-CBOR requirements apply only to events emitted in `events.bin` and to bytes used for event hashing.
- Timestamps in `manifest.json` **MUST** use RFC3339 strings.

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

Signature verification (Section 19) and operator-attestation verification (Section 20) are OPTIONAL and **independent** of the procedure above and of each other: a verifier **MUST** complete the integrity checks regardless of whether signatures or operator attestations are present, and their presence, absence, or validity **MUST NOT** affect the integrity verdict.

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

When signature verification is performed against a trusted key (Section 19), a `signatures[]` entry that fails to verify under a trusted key matching its `key_id` **MUST** cause rejection. This signature-level check is independent of, and subsequent to, the integrity-level rejection rules above.

When operator-attestation verification is performed (Section 20), a verifier **MUST NOT** attempt to verify an `operator_attestations[]` entry whose `statement_version` is not `"AGEF-OPERATOR-v1"` or whose `scheme` is not `"ed25519"` (domain-separation guard), and **MUST** treat an entry whose signed identity fields contain `\n` or `\r` as malformed. An entry that fails to verify under a trusted operator key matching its `key_id` **MUST** cause rejection. These checks are likewise independent of the integrity-level rules above.

## 15. Compatibility and Evolution

- v0.x is pre-stable: breaking changes are permitted across minor lines.
- Within the `0.1.x` line, changes are additive: AGEF v0.1.2 introduces the OPTIONAL `manifest.signatures[]` envelope (Section 19) and v0.1.3 the OPTIONAL `manifest.operator_attestations[]` envelope (Section 20); neither changes anything else. A v0.1.1 or v0.1.2 reader reads a v0.1.3 bundle unchanged, ignoring fields it does not recognize.
- Future versions **SHOULD** preserve forward migration guidance.
- v1.0 is intended as first stable major.
- New event kinds in future majors **MUST** be version-gated.
- v0.1 readers **MUST NOT** silently ignore unknown required semantics, but **MUST** ignore unknown OPTIONAL manifest fields within a supported `0.1.x` line.

## 16. Security Considerations

AGEF provides tamper-evidence and portability; it does not provide identity attribution by itself.

Implications:
- AGEF v0.1.2 defines an OPTIONAL detached-signature layer (Section 19) for attributability; external signing tools **MAY** also be applied. Such a signature attests to the signing *key*, whose trust is established out of band.
- AGEF v0.1.3 defines an OPTIONAL operator-attestation layer (Section 20). A successful check proves only that a known *key* signed the claim; "verified" **MUST NOT** be presented as proof of the self-asserted identity string (`operator_id`/`display_name`/`role`/`org`). Binding a name to a key is an out-of-band trust decision.
- Bundles may contain sensitive content; storage and sharing controls **MUST** be handled by operators.
- Verification checks integrity, not semantic correctness of model/tool outputs.

## 17. Implementation Notes (Non-Normative)

Known libraries often used for canonical CBOR implementations:

- Rust: `ciborium`
- Go: `fxamacker/cbor`
- Python: `cbor2`
- JavaScript/TypeScript: `cbor-x`

These are known to produce RFC 8949 canonical encoding when configured correctly. Implementers should validate canonical-encoding behavior with test vectors before claiming conformance.

### 17.1 Conformance Profiles

AGEF defines two conformance profiles in v0.1:

- **Bundle Profile**: an implementation that produces and/or consumes AGEF bundles per Sections 5-14.
- **Substrate Profile**: an implementation that maintains a content-addressed event journal compatible with AGEF semantics, but does not necessarily produce bundles directly.

A Substrate Profile implementation **MUST** be able to produce Bundle Profile output via an export pathway when required.

`akmon-journal` is currently a Substrate Profile implementation. Akmon's planned Phase 4 export/import functionality is intended to provide Bundle Profile capability.

## 18. Licensing Note for Spec Text

The normative spec text in `SPEC.md` is intended for CC BY 4.0 licensing.
Reference implementations may use different licenses.

## 19. Detached Signatures (v0.1.2, OPTIONAL)

AGEF v0.1.2 defines an OPTIONAL detached-signature envelope so a session's *attributability* can be verified by a third party, independently of the producer. Signing is additive: it never alters the event hash chain, the objects store, or any content hash. AGEF is not a cryptographic-identity or PKI standard (Section 3.2); this layer is a signature envelope only, and key trust is established out of band.

### 19.1 Signed Statement (`AGEF-SIG-v1`)

Signers do not sign the bare head hash. They sign a canonical, domain-separated UTF-8 statement. The statement **MUST** be encoded exactly as follows: fixed field order, a single `\n` (LF) after every line including the last, and no other whitespace:

```
AGEF-SIG-v1
agef_version:0.1.2
hash_algorithm:<sha256|blake3>
session_id:<uuid>
head:<lowercase-hex-of-session-head>
```

The `AGEF-SIG-v1` first line versions the statement independently of the bundle. Binding `session_id`, `hash_algorithm`, and `head` prevents a signature from being replayed as if it covered a different session, algorithm, or protocol.

### 19.2 Schemes

v0.1.2 defines one scheme:

- `ed25519` — a raw Ed25519 signature (RFC 8032) over the statement bytes. The public key is distributed as SPKI/PEM (openssl-compatible).

`ecdsa-p256` with X.509 is **reserved** for PKI-oriented deployments and **MAY** land in a later minor. Keyless/transparency-log schemes are out of scope for the `0.1.x` line. (OpenPGP was considered and rejected by the reference implementation: every pure-Rust OpenPGP implementation transitively requires the advisory-bearing `rsa` crate.)

### 19.3 Placement

Signatures live in `manifest.signatures[]` (Section 6). Because the head already commits to the entire DAG, a single signature authenticates the whole session; multiple entries allow counter-signing. Signatures are manifest metadata and are excluded from event hashing — adding or counter-signing never mutates the head or any event/object hash.

### 19.4 Verification

Signature verification is OPTIONAL and runs only after, and independently of, the Section 13 integrity checks. Given one or more trusted public keys, a verifier reconstructs the `AGEF-SIG-v1` statement (§19.1) from the manifest's `agef_version`, `hash_algorithm`, `session.id`, and `session.head`, then verifies each `signatures[]` entry under its named `scheme`. Outcomes:

- trusted key present and signature valid ⇒ **verified**;
- trusted key present (matching the entry `key_id`) and signature invalid ⇒ **hard failure** (Section 14);
- signatures present but no trusted key available ⇒ **unverified (no key)** — integrity is still established and this is not a failure;
- no signatures present ⇒ **unsigned** — not a failure.

A verifier **MAY** offer a `require-signature` mode that treats absent or unverified signatures as failure.

### 19.5 Offline Verification with `openssl`

Because `ed25519` signs the raw statement bytes, a third party can verify a signature with stock `openssl` and no AGEF tooling. Given the statement bytes in `statement.bin`, the detached signature bytes in `signature.bin`, and the signer's public key in `pubkey.pem`:

```
openssl pkeyutl -verify -pubin -inkey pubkey.pem -rawin -in statement.bin -sigfile signature.bin
```

An Ed25519 public key in SPKI DER form is the fixed 12-byte prefix `30 2a 30 05 06 03 2b 65 70 03 21 00` followed by the raw 32-byte public key (44 bytes total); `pubkey.pem` is the PEM encoding of those bytes. The `key_id` recorded in `manifest.signatures[]` is the lowercase-hex SHA-256 of the raw 32-byte key.

### 19.6 Trust Model

Public keys are distributed out of band; AGEF specifies no PKI, web-of-trust, or transparency log in v0.1.2. Existence-at-time anchoring (e.g. RFC-3161 timestamping or a transparency log such as Rekor) is a possible future additive layer, not part of v0.1.2.

## 20. Operator Attestations (v0.1.3, OPTIONAL)

AGEF v0.1.3 defines an OPTIONAL operator-attestation envelope so a session's *human/role accountability* can be asserted and checked independently of who produced or sealed it. It targets the accountable-operator axis that record-keeping and human-oversight regimes call for (e.g. EU AI Act Art. 14 and Art. 12(3)) without introducing a PKI, DID, directory, or cloud dependency. It is additive: it never alters the event hash chain, the objects store, any content hash, the `signatures[]` envelope, or the `AGEF-SIG-v1` head statement (Section 19). An operator attestation is a *separately-signed* claim and is **never** folded into `AGEF-SIG-v1`.

### 20.1 Signed Statement (`AGEF-OPERATOR-v1`)

An operator signs a canonical, domain-separated UTF-8 statement. The statement **MUST** be encoded exactly as follows: fixed field order, a single `\n` (LF) after every line including the last, and no other whitespace. All four identity fields are inside the signed bytes, so the displayed strings are exactly what was signed; `display_name`, `role`, and `org` default to the empty string when unset, `operator_id` is required and non-empty, and none may contain `\n` or `\r`:

```
AGEF-OPERATOR-v1
agef_version:0.1.3
hash_algorithm:<sha256|blake3>
session_id:<uuid>
head:<lowercase-hex-of-session-head>
operator_id:<...>
display_name:<...>
role:<...>
org:<...>
```

Binding `session_id` + `head` prevents lifting an attestation onto a different session. The `AGEF-OPERATOR-v1` first line and its additional fields domain-separate it from `AGEF-SIG-v1` (Section 19): because the first line and field set differ, an Ed25519 signature over one statement can never validate as the other.

### 20.2 Scheme

`ed25519` — a raw Ed25519 signature (RFC 8032) over the statement bytes, with the public key distributed as SPKI/PEM. This is the same primitive as Section 19, so an operator claim is verifiable with stock `openssl` over a second statement file (§20.6). `ecdsa-p256`/X.509 is **reserved**; keyless/transparency-log schemes are out of scope for the `0.1.x` line.

### 20.3 Placement

Attestations live in `manifest.operator_attestations[]` (Section 6), as manifest metadata excluded from event hashing. Multiple entries are allowed (e.g. operator plus reviewer/approver); a verifier reports all of them and **MUST NOT** collapse them into a single "the operator." Adding an attestation changes no event or object hash, does not affect any `signatures[]` entry, and — on an already-head-signed bundle — **MUST NOT** change `manifest.agef_version` (Section 1).

### 20.4 Verification

Operator verification is OPTIONAL and **independent** of both the Section 13 integrity checks and Section 19 head-signature verification. A verifier **MUST** reject any entry whose `statement_version` is not `"AGEF-OPERATOR-v1"` or whose `scheme` is not `"ed25519"` as a verification candidate **before** any signature check (domain-separation guard), and **MUST** treat an entry whose identity fields contain `\n` or `\r` as malformed. For each well-formed entry, the verifier reconstructs the `AGEF-OPERATOR-v1` statement (§20.1) from the manifest's protocol-context fields (`agef_version`, `hash_algorithm`), the `session.id` and `session.head`, and the entry's stored identity fields, then checks it against trusted operator keys. Outcomes:

- trusted key present and signature valid ⇒ **verified**;
- trusted key present (matching the entry `key_id`) and signature invalid ⇒ **hard failure** (Section 14);
- attestations present but no trusted key available ⇒ **unverified (no key)** — not a failure;
- no attestations present ⇒ **unattributed** — not a failure.

A verifier **MAY** offer a `require-operator` mode (absent or unverified ⇒ failure) and a `require-operator-key` mode (a specific trusted operator key **MUST** have attested; the presence of arbitrary self-asserted text is never sufficient).

### 20.5 Honesty (Normative)

"Verified" **MUST** be presented as a property of the **key**, never of the self-asserted identity string. The `key_id` is a matching hint, never the trust anchor; trust in the `operator_id` → key mapping is established out of band. A verifier **MUST NOT** present a verified signature as proof that the named person, role, or organization approved the session — only that a signature by the matching key was checked. Self-asserted fields are attacker-controlled text and **SHOULD** be escaped or quoted on display.

`created_at` is not part of the signed statement; it is unverifiable, self-reported provenance and **MUST NOT** be treated as an anchored timestamp.

### 20.6 Offline Verification with `openssl`

As in §19.5, an operator claim is verifiable with stock `openssl` and no AGEF tooling. Given the `AGEF-OPERATOR-v1` statement bytes in `operator_statement.bin`, the detached signature in `operator_signature.bin`, and the operator public key in `operator_pubkey.pem`:

```
openssl pkeyutl -verify -pubin -inkey operator_pubkey.pem -rawin -in operator_statement.bin -sigfile operator_signature.bin
```

The operator public key uses the same Ed25519 SPKI encoding described in §19.5; the `key_id` is the lowercase-hex SHA-256 of the raw 32-byte operator key.

### 20.7 Trust Model

Operator public keys are distributed out of band; AGEF specifies no PKI, DID, web-of-trust, directory, or transparency log in v0.1.3. This preserves offline, no-cloud, `openssl`-only verification. The statement deliberately does not bind the head-signer's `key_id`: cross-session transplant is fully prevented by `session_id` + `head`, while "operator O attested head X" and "key S sealed head X" are established by checking the two axes independently. A future `AGEF-OPERATOR-v2` **MAY** add a `sig_key_id` line if single-signature endorsement semantics are ever needed.
