# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html) for specification text revisions.

## [0.1.2-seed] - 2026-06-06

### Added
- **OPTIONAL detached signatures (`manifest.signatures[]`).** AGEF v0.1.2 adds an additive, OPTIONAL detached-signature envelope over the session head (new Section 19). Absence means the bundle is unsigned. Each entry carries `scheme` (`ed25519`), `key_id` (lowercase-hex SHA-256 of the raw 32-byte public key), `statement_version` (`AGEF-SIG-v1`), `signature` (lowercase hex), and a self-reported `created_at`.
- **`AGEF-SIG-v1` signed statement.** Signers sign a canonical, domain-separated UTF-8 statement binding `agef_version`/`hash_algorithm`/`session_id`/`head` (fixed field order, LF after every line), not the bare head hash.
- Offline verification recipe using stock `openssl pkeyutl -verify … -rawin`, including the fixed 12-byte Ed25519 SPKI DER prefix.
- OPTIONAL `require-signature` verification mode; per-signature outcomes (verified / hard failure / unverified-no-key / unsigned).

### Changed
- Section 3.2 non-goal reworded: AGEF remains not a PKI/identity *standard*, but now offers an OPTIONAL signature layer that mandates no PKI, key distribution, keyless/Sigstore, or transparency log.
- Section 1 / Section 15 clarify that v0.1.2 is an additive minor: `0.1.x` readers **MUST** accept any `0.1.x` bundle and ignore unknown manifest fields, so v0.1.1 readers read v0.1.2 bundles unchanged.

## [0.1.1-seed] - 2026-04-26

### Added
- Conformance Profiles guidance defining:
  - **Bundle Profile** (Sections 5-14 bundle producer/consumer behavior)
  - **Substrate Profile** (AGEF-compatible journal substrate with export path to Bundle Profile)
- Explicit implementation note that internal storage format is implementation-defined; canonical CBOR requirements apply to `events.bin` and event-hashing bytes, not internal persistence.

### Changed
- Clarified v0.1 event-parent invariant: every non-`SessionStart` event must have exactly one parent equal to the immediately preceding event by sequence.
- Clarified timestamp encoding rules:
  - producers may emit CBOR tag 1 timestamps as integer or float,
  - readers must accept both forms,
  - `akmon-journal` v0.1 reference behavior is documented as integer-seconds emission.

### Fixed
- **`agef_version` wire format clarification.**
  `manifest.json` `agef_version` MUST be a three-part semantic version string. For AGEF v0.1.1 the value is `"0.1.1"`. Earlier 0.1.0-seed documentation suggested a two-part form (`"0.1"`); that wording is corrected. The three-part form has always been required by the reference implementation and is now reflected in the specification text.

## [0.1.0-seed] - 2026-04-26

### Added
- Initial public publication of the AGEF v0.1 pre-stable specification (`SPEC.md`).
- Initial repository scaffolding with governance and examples directories.
