# AGEF (Agent Governance Evidence Format)

AGEF is an open specification for portable, tamper-evident AI-agent session evidence. It defines how a session can be represented as content-addressed objects plus merkle-linked events so evidence can be verified offline, transferred across systems, and reviewed by independent tools.

**Status:** `v0.1.1` (pre-stable).  
Bundles set **`agef_version`** to **`"0.1.1"`** in `manifest.json` (three-part semver; see `SPEC.md` Section 6).

The reference implementation is [Akmon](https://github.com/radotsvetkov/akmon) (**v2.0.0** and later ship bundle export and import, with journaling in `akmon-journal`).

## Conformance Profiles

AGEF v0.1 defines two conformance profiles:

- **Bundle Profile** - produces and/or consumes AGEF bundles per `SPEC.md` Sections 5-14.
- **Substrate Profile** - maintains AGEF-compatible content-addressed session journals, without necessarily emitting bundles directly.

A Substrate Profile implementation must be able to produce Bundle Profile output through an export pathway.

## Repository Layout

- `SPEC.md` - normative AGEF specification text.
- `GOVERNANCE.md` - current governance model and evolution path.
- `examples/` - interoperability-oriented examples and scaffolding.

## Licensing

- Specification text (`SPEC.md` and other normative docs): **CC BY 4.0**
- Code and repository implementation artifacts: **Apache-2.0**

## Get Involved

Contributions, review feedback, and interoperability notes are welcome.  
Please start with `GOVERNANCE.md` for process and decision flow.
