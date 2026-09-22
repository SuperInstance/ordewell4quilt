# Quilt-first ordewell — design note

2026-09-22, kimi1 (Cocapn Fleet), for the `ordewell4quilt` fork.
Status: design for review. Nothing here edits upstream files; the
refactor is **additive** (a core plugin + one adapter) to keep the
fork's upstream-sync surface clean.

## What "quilt-first" means

Today ordewell owns its plan state: `Task` objects, a 7-valued
`TaskStatus`, in-memory + file persistence, plan-utils over the DAG.
Quilt-first inverts the spine: the **quilt 5-opcode kernel**
(BIND/LINK/EFFECT/VIEW/TICK, +FORGET) becomes the source of truth for
plan state, and ordewell's `Task`/`Plan` models become *views* —
materialized from the kernel+WAL by replay, exactly the way hermit's
quilt kernel P1 made D1 a view of the WAL. The refactor is not a
rewrite. It is an adapter under the existing storage interface plus a
receipts layer under every transition.

## Why

- **Replayable plans.** A plan that can be re-derived from its WAL is a
  plan you can audit, diff, and resurrect. `verify()` re-deriving the
  chain and breaking at `broken_at` turns "what happened to my run"
  into a mechanical answer.
- **Receipts on every transition.** candor's doctrine, already shipped:
  `remember()` books the hash-chained receipt *before* storage; boot
  replays and verifies. Task status changes get the same treatment —
  authority never lags state.
- **One spine, many faces.** The quilt substrate already runs TS, Rust,
  C, WASM. An ordewell plan bound to the kernel is inspectable from
  every port and renderable by the quilt-view engine (live plan DAGs,
  Mermaid export) without new tooling.

## Concept mapping (grounded in `packages/core/src/models/Task.ts`)

| ordewell today | quilt-first |
|---|---|
| `Plan` (goal → ordered tasks) | `BIND` goal node in namespace `ordewell:plan:{id}` |
| `Task` | `BIND` task node; task fields (runner, model, mode, prompt) live in the node payload |
| dependencies / plan order (`plan-utils`) | `LINK` edges, kind `depends-on` — the DAG *is* the plan |
| `TaskStatus` (7 values: pending, approved, in_progress, completed, failed, blocked, awaiting_user) | full 7 values preserved in payload; **three bands** for routing/render: `{pending, approved, blocked}` not-live · `{in_progress, awaiting_user}` live · `{completed, failed}` settled |
| execution | `EFFECT` (runner invoked, result booked as receipt payload-hash) |
| verification (ordewell "execute and verify") | receipt row + verifier row; both hash-chained |
| progress/dashboards | `VIEW` — quilt-view engine subscription renders the live DAG |
| orchestrator tick (queue pressure, retries) | `TICK` — one beat, one receipt, never silent |
| cancel / supersede / retry-reset | `FORGET` per the candor doctrine: **authority without erasure** — the row stays, marked inert; a no-op books a visible REFUSAL row |

The 7→3 band mapping is deliberate, and it is the same grammar as
vector-novelty's receipts-sensor (coarse sweep over all rows, graded
latch on band crossing): render everything coarsely, commit attention
only where the band changes.

## The adapter — `QuiltPlanStore`

Lives in `packages/core/src/plugins/quilt/` (the plugins directory keeps
it out of upstream files). Implements ordewell's existing storage
interface, so swapping the spine is a config line, not a refactor:

1. Every state mutation → kernel ops + WAL receipt, then view update
   (view lags; chain leads).
2. Boot → replay-verify: re-derive every task's status from the chain;
   mismatch = loud refusal to start (tamper-evidence, D10 doctrine).
3. Snap point: `TaskStatus` writes are accepted only if the resulting
   band transition is legal per a small state machine (blocked →
   in_progress needs an EFFECT carrying an unblock receipt, etc.).

## Refactor sequence

| Phase | Deliverable | Verification |
|---|---|---|
| 0 — adapter spine | `QuiltPlanStore` behind the existing interface; file-backed kernel+WAL; replay-verify at boot | full existing suite green; new replay tests: kill mid-run, reload, state re-derived bit-for-bit |
| 1 — receipts | receipt booked before every status write; no-op mutations book REFUSAL rows | tamper test: hand-edit WAL row → boot refuses, names `broken_at` |
| 2 — VIEW | quilt-view subscription → live plan DAG; Mermaid export for PR descriptions | scene diff test: two plans differing in one edge render differently |
| 3 — rooms | tripartite-room orchestrator: three innate agents per plan room (Ground Truth/Constraint/Communication) scoring the plan state as a trit | room-trit composition table tests |

## Smallest first build (one evening)

Phase 0, one file of kernel adapter + one of WAL, one plan, three tasks,
one dependency edge: run it, kill the process mid-task, reload, prove
the plan re-derived itself from the chain alone. If that demo works,
every later phase is downhill.

## Honest gaps

- Fork carries upstream `ordewell` lineage (Apache-2.0, © Alessandro
  Costanzo Ciano); the adapter must remain droppable for upstream syncs.
- Kernel + WAL in-process, file-backed (same limit as candor PR #1/2:
  receipts-v2 signature envelope deferred).
- The 7→3 band mapping is a rendering/routing convenience; the 7
  original states remain the contract. Do not let bands leak into
  status logic.
- No perf claims: replay is O(plan size); fine for coding-agent plans,
  unproven at 10⁵ tasks.
