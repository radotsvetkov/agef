# AGEF Governance

## Current governance model (v0.1)

AGEF governance is currently **benevolent dictator**.

For v0.1, final decision authority for spec changes is held by **Rado Tsvetkov**. This keeps early-stage direction coherent while the format is still pre-stable (`v0.x`).

## Roadmap to v1.0 governance

Before AGEF reaches v1.0, governance is expected to transition to a **core-maintainer model**:

1. Add 1–3 maintainers with demonstrated implementation and review contributions.
2. Require maintainer review for normative changes.
3. Publish a lightweight change policy distinguishing:
   - editorial clarification,
   - additive change,
   - breaking change.

## How to propose changes

1. Open a GitHub issue describing the problem and proposed change.
2. For normative changes, include:
   - affected sections,
   - compatibility impact,
   - migration expectation.
3. Submit a PR referencing the issue.
4. Mark PR as one of:
   - Editorial
   - Additive (non-breaking)
   - Breaking

## Decision policy (v0.1)

- Editorial fixes can merge quickly.
- Normative changes should include explicit rationale.
- Breaking changes are allowed in v0.x but should be clearly labeled.
- Rejected proposals receive a short rationale.

## Licensing

Licensing split is defined in `README.md` and `LICENSE`:
- code/repository artifacts: Apache-2.0
- normative spec text (`SPEC.md`): CC BY 4.0
