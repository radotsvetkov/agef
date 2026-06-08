# AGEF (Agent Governance Evidence Format)

AGEF is an open specification for portable, tamper-evident AI-agent session evidence. It defines how a session can be represented as content-addressed objects plus merkle-linked events so evidence can be verified offline, transferred across systems, and reviewed by independent tools.

**Status:** `v0.1.3` (pre-stable).

On top of the baseline bundle format, AGEF defines two optional, additive layers:

- **Detached signatures** (`manifest.signatures[]`, added in v0.1.2) let anyone holding the signer's public key confirm that a session was sealed by a particular key, independently of the producer. See `SPEC.md` Section 19.
- **Operator attestations** (`manifest.operator_attestations[]`, added in v0.1.3) record which accountable operator or role stands behind a session. This is a separately-signed claim whose key trust is established out of band. See `SPEC.md` Section 20.

A bundle sets `agef_version` (a three-part semantic version; see `SPEC.md` Section 6) to the highest layer it actually uses: `"0.1.1"` when it uses neither, `"0.1.2"` once it carries signatures, and `"0.1.3"` once it carries operator attestations. Because every layer is additive, an older reader still reads a newer bundle and simply ignores the fields it does not recognize.

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
