# Stage <N> gate review

> Human-readable retrospective produced by review gate `R<N>`. Its machine-readable
> counterpart is `stage-<N>.review.yaml`. Copy this template to `stage-<N>.review.md`.

## Did the stage meet its exit criteria?

- [ ] Every stage-`<N>` task `done`.
- [ ] Deliverables present at declared `produces` paths.
- [ ] Metric snapshot recorded.

State plainly: yes / partially / no, and why.

## What we now know that we did not

The point of the gate. Summarise the findings and surprises from each task's handback
(`artefacts/handbacks/S<N>.*.yaml`). Call out anything that breaks an assumption the
downstream plan was built on.

## Plan revisions

For each change, give the op, the target task id, and the rationale (mirror
`stage-<N>.review.yaml#plan_changes`). Explain *why the new information forces the
change* — not just what changed.

## Risks discovered

List with severity and mitigation. Flag anything that should change staffing
(e.g. a task that turned out to need an `opus` Diagnostician rather than a `sonnet`
Implementer).

## Decision

`proceed` | `replan` | `halt`. If `replan`, say what must be redone. If `halt`, escalate.

## Next-stage prompts generated

List the `prompts/stage-<N+1>/*.md` files written, each one-line summarised. Confirm each
is schema-valid (`../schema/prompt.schema.json`) and self-contained for a cold-start agent.
