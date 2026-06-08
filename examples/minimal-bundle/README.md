# Minimal bundle example (AGEF v0.1.1)

This directory is a **concrete** AGEF v0.1.1 bundle: the normative layout from `SPEC.md` Section 5 (`manifest.json`, `events.bin`, `objects/<hex>`), plus a portable archive of the same bytes.

**Why this example is still `v0.1.1`.** It is deliberately a baseline bundle — no signatures, no operator attestations — so its `agef_version` stays `"0.1.1"` even though the specification has since grown two optional layers: detached signatures (`manifest.signatures[]`, v0.1.2) and operator attestations (`manifest.operator_attestations[]`, v0.1.3). Both layers are additive and optional, so an unsigned, unattributed bundle like this one remains fully valid and verifies unchanged. To produce a signed or attested bundle instead, run `akmon bundle sign` or `akmon bundle attest`; each writes manifest metadata only and leaves the event hash chain and object hashes untouched.

## Producer and date

- **Producer:** [Akmon](https://github.com/radotsvetkov/akmon) v2.0.0 (reference implementation).
- **Generated:** 2026-05-10 (bundle timestamps in `manifest.json` are UTC).

The session is intentionally trivial: a short headless run comparable to an “echo hello”–style prompt so the artifact stays small and easy to read. Event and object counts are minimal but real (not hand-edited).

## What v0.1.1 adds vs v0.1.0-seed

See `CHANGELOG.md` for the full list. In short, v0.1.1-seed clarified:

- **Conformance profiles:** Bundle Profile vs Substrate Profile (export path to bundles).
- **Parent rule:** every non-`SessionStart` event has exactly one parent—the previous event by `sequence`.
- **Timestamps:** CBOR tag 1 may be integer or float seconds; readers must accept both.
- **`agef_version`:** three-part semver in `manifest.json` (for this release, `"0.1.1"`); see `SPEC.md` Section 6 and the Fixed note under `[0.1.1-seed]` in `CHANGELOG.md`.

## Files

| Path | Role |
|------|------|
| `reference.agef` | `tar.zst` bundle (same layout as Section 5). Use this path with `akmon bundle import`. |
| `manifest.json` | Bundle manifest (sorted keys, UTF-8 JSON). |
| `events.bin` | Length-prefixed canonical CBOR events. |
| `objects/` | Content-addressed blobs (lowercase hex filenames). |

## `manifest.json` (reference)

```json
{
  "agef_version": "0.1.1",
  "event_count": 5,
  "hash_algorithm": "sha256",
  "object_count": 7,
  "producer": {
    "name": "akmon",
    "version": "2.0.0"
  },
  "session": {
    "created_at": "2026-05-10T16:25:54.801Z",
    "ended_at": "2026-05-10T16:26:03.256Z",
    "head": "66405ca6517c78d1bea3d69d5438bd6b7714c4ce5024ce2aa8ca7289c460b1f6",
    "id": "44b87e14-937e-4f1f-bcea-eb02efe81be3"
  }
}
```

## Verify with Akmon

Requires [Akmon](https://github.com/radotsvetkov/akmon) v2.0.0+ on your `PATH` (or pass the full path to the `akmon` binary). Validation does not write to your journal:

```bash
akmon bundle import examples/minimal-bundle/reference.agef --verify-only
```

Machine-readable report:

```bash
akmon bundle import examples/minimal-bundle/reference.agef --verify-only --format json
```

Successful verification exits `0` and reports `passed: true` in JSON mode.
