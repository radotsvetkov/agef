# AGEF (Agent Governance Evidence Format)

AGEF is an open specification for portable, tamper-evident AI-agent session evidence. It defines how a session can be represented as content-addressed objects plus merkle-linked events so evidence can be verified offline, transferred across systems, and reviewed by independent tools.

**Status:** `v0.1.3` (pre-stable).  
Bundles set **`agef_version`** to the highest feature layer they use — `"0.1.1"` for the baseline bundle format, `"0.1.2"` when they carry detached signatures (`manifest.signatures[]`; see `SPEC.md` Section 19), or `"0.1.3"` when they carry operator attestations (`manifest.operator_attestations[]`; see `SPEC.md` Section 20). All layers are additive: a v0.1.1 or v0.1.2 reader still reads a v0.1.3 bundle, ignoring fields it does not recognize.

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
