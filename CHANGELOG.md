# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html) for specification text revisions.

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
- Updated spec document version label from v0.1 to v0.1.1 (wire format remains `agef_version: "0.1"`).

## [0.1.0-seed] - 2026-04-26

### Added
- Initial public publication of the AGEF v0.1 pre-stable specification (`SPEC.md`).
- Initial repository scaffolding with governance and examples directories.
