# Orchestration loop

This file specifies the control loop that drives the plan. It is written so an
orchestrator (human or agent) can execute it mechanically.

## State

The single source of truth is `tasks.yaml`. Each task has a `status`
(`ready|blocked|in_progress|done|revised|dropped`). Derived artefacts live under
`artefacts/`, `reviews/`, and `prompts/`.

## Main loop

```
loop:
  refresh_ready_set()                 # mark task ready iff all deps are done and status in {blocked, revised}
  for task in ready_set (respect parallel_group, validation back-pressure):
      dispatch(task)                  # see Dispatch
  on task completion:
      record handback -> artefacts/handbacks/<id>.yaml
      set task.status = done
      if all tasks in stage S are done and review gate R<S> exists and is not done:
          mark R<S> ready
  if a review gate R<S> completed:
      apply its plan revisions to tasks.yaml
      ingest the prompts it generated under prompts/stage-<S+1>/
  halt when the terminal review gate (R5) is done
```

### Dispatch

To dispatch task `T`:
1. Read `prompts/stage-<T.stage>/<T.id>.md` (the cold-start prompt).
2. Start an agent of `T.agent` at model `T.model`, effort `T.effort`.
3. The prompt is fully self-contained (the agent has no memory of this conversation).
4. On completion the agent writes its **Handback** to
   `artefacts/handbacks/<T.id>.yaml` (schema: `schema/review.schema.json#/$defs/handback`)
   and its `produces` deliverables to the declared paths.

### Validation back-pressure

When selecting from the ready set, prefer to keep the pipeline busy:
- Dispatch all independent `local` tasks immediately.
- Cap concurrent `ci`/`corpus` tasks at the CI runner budget.
- If an `opus` agent would otherwise idle on a CI queue, give it a `local` task first.

## Stage-gate review (the adaptive core)

A review gate `R<S>` is itself a task (Architect · opus · high). Its job is to turn the
stage's learnings into a revised plan and the next stage's prompts. It MUST:

1. **Ingest** every handback from stage `S` (`artefacts/handbacks/S<S>.*.yaml`) and the
   stage deliverables.
2. **Write `reviews/stage-<S>.review.md`** — a human-readable retrospective: what was
   found, what surprised us, what assumptions broke.
3. **Write `reviews/stage-<S>.review.yaml`** — machine-readable (schema:
   `schema/review.schema.json`): for each downstream change, an op
   (`add|modify|drop|reorder`) on a task id, with rationale; plus `risks_discovered`,
   `decision` (`proceed|replan|halt`), and `next_stage_prompts` (the list of files
   generated).
4. **Apply revisions to `tasks.yaml`** (or emit a patch the orchestrator applies).
   Never renumber existing ids; `drop` and `add` instead.
5. **Generate `prompts/stage-<S+1>/*.md`** (one per next-stage task) and
   `prompts/stage-<S+1>/_index.yaml`, each conforming to `schema/prompt.schema.json`
   and following the structure of the stage-0 prompts.
6. **Set its own status `done`**, which unblocks stage `S+1`.

### Why only stage 0 is pre-written

The plan beyond stage 0 is deliberately *not* frozen. Stage 0 builds the measurement
harness and the checkpoint/replay capability; what those reveal (which stages actually
introduce nondeterminism, how big the analysis wobble really is, whether parallelism or
the algorithm is at fault) should reshape stages 1–5. Pre-writing later prompts would
bake in today's guesses. The review gates are where the plan learns.

## Review-gate task ids

| Gate | Runs after | Produces |
|---|---|---|
| `R0` | Stage 0 complete | `reviews/stage-0.*`, `prompts/stage-1/*` |
| `R1` | Stage 1 complete | `reviews/stage-1.*`, `prompts/stage-2/*` |
| `R2` | Stage 2 complete | `reviews/stage-2.*`, `prompts/stage-3/*` |
| `R3` | Stage 3 complete | `reviews/stage-3.*`, `prompts/stage-4/*` |
| `R4` | Stage 4 complete | `reviews/stage-4.*`, `prompts/stage-5/*` |
| `R5` | Stage 5 complete | `reviews/stage-5.*` (final retrospective; no next stage) |

## Invariants the orchestrator enforces

- A task is only dispatched when `status == ready` and all `deps` are `done`.
- A stage's `R<S>` gate is the *only* task permitted to modify downstream tasks or write
  next-stage prompts.
- If a review gate sets `decision: halt`, stop and escalate to a human.
- Handbacks are append-only; never overwrite a completed task's handback.
