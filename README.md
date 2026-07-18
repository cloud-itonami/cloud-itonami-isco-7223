# cloud-itonami-isco-7223

Open Occupation Blueprint for **ISCO-08 7223**: Metal Working Machine Tool Setters and Operators.

This repository designs a forkable OSS business for a metal-working machine shop scheduling and logistics coordination practice: a workshop scheduling and supply-coordination robot manages crew/task records under a governor-gated actor, so a machine-shop crew keeps its own operating records instead of renting a closed workforce-management SaaS.

**Maturity: `:implemented`.** `src/machinist/` implements the
`MachinistActor` as a `langgraph.graph/state-graph`
(`machinist.actor`) wired to a `Machinist Advisor`
(`machinist.advisor`) and an independent `MachinistGovernor`
(`machinist.governor`), following the itonami actor pattern
(ADR-2607121000): `:intake -> :advise -> :govern -> :decide -+-> :commit
(:ok? true) +-> :request-approval (:escalate? true, human-in-the-loop
interrupt) +-> :hold (:hard? true)`. HARD invariants (always hold,
never overridable): worker provenance, workshop provenance,
no-actuation (`:effect` must be `:propose`), a closed op-allowlist
(`:log-work-record`, `:schedule-crew-operation`,
`:flag-safety-concern`, `:coordinate-supply-order` — nothing else may
ever be proposed), and a permanent, unconditional block on any
proposal that would directly finalize a machine-setup or
machining-execution decision (e.g. deciding to proceed with a specific
lathe, mill or press setup/run) or override a shop safety officer's
judgment. Always-escalate paths (human sign-off regardless of
confidence, mapping this repo's Trust Controls in
[`docs/business-model.md`](docs/business-model.md)):
`:flag-safety-concern` (always) and `:coordinate-supply-order` above
the registered cost threshold.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a workshop scheduling/logistics coordination robot performs crew scheduling, task/batch/materials-usage/progress-record logging and tooling-materials supply-order coordination for a metal-working machine-shop crew, under an actor that proposes actions and an independent **Machinist Governor** that gates them. The governor never dispatches hardware itself, never sets up or operates CNC or manual machine tools (lathes, mills, presses) on the shop floor, and never finalizes a machine-setup or machining-execution decision or overrides a shop safety officer's judgment; `:high`/`:safety-critical` actions (such as a flagged machinery-hazard/tool-condition concern, or an above-threshold supply order) require human sign-off. **This actor coordinates workshop scheduling/logistics only — it never sets up or operates machine tools itself.**

## Core Contract

```text
crew roster + workshop registration + safety-reporting policy
        |
        v
Machinist Advisor -> Machinist Governor -> log/schedule/coordinate, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, finalize
a machine-setup or machining-execution decision, override a shop safety
officer's judgment, suppress an operating record, or disclose sensitive data
without governor approval and audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `7223`). Required capabilities:

- :robotics
- :identity
- :audit-ledger

See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md).

## License

AGPL-3.0-or-later.
