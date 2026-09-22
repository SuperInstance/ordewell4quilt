<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/readme/logo-dark.png">
    <img src="assets/readme/logo-light.png" width="380" alt="Ordewell">
  </picture>
</p>

> A fork with a new spine: ordewell's planner keeps its body, and learns
> to keep receipts.

**ordewell4quilt** is the [ordewell](https://github.com/ordewell/ordewell)
coding-agent orchestrator adapted for the **Cocapn Fleet** — refactored,
per [docs/QUILT-FIRST.md](docs/QUILT-FIRST.md), so that plan state lives on
the fleet's **quilt kernel** (BIND / LINK / EFFECT / VIEW / TICK, +FORGET)
with a hash-chained receipt behind every status transition. The upstream
planner, runners, and verification stay; what changes is where truth lives
and how honestly it reports itself.

**Start here, by audience:**

| You are a… | Read |
|---|---|
| first-time visitor | this page, then upstream's [docs](https://ordewell.ai/docs.html) for the base product |
| engineer | [docs/QUILT-FIRST.md](docs/QUILT-FIRST.md) — the kernel-as-spine design, concept map, adapter seam, phases |
| CTO / evaluator | the *Why* below, then QUILT-FIRST.md's honest-gaps section |

---

## Why the fleet forks it

Upstream ordewell already believes the right things: one goal becomes an
ordered plan, every task its own runner and model, every verdict earned by
a marker in session output — never by a model's opinion. The fleet adds
what its own doctrine requires:

1. **The kernel is the spine.** Plan state (goal → task DAG → status) is
   bound to the quilt kernel and replayable from its WAL. A plan you can
   re-derive is a plan you can audit.
2. **Every transition books a receipt first.** Hash-chained, ordered,
   tamper-evident — the candor WAL doctrine. Authority never lags state;
   boot replays and verifies, and a broken chain refuses loudly rather
   than guessing.
3. **Statuses keep their 7 values; routing uses 3 bands.**
   not-live / live / settled — coarse sweep over everything, attention
   only where a band changes.
4. **No erasure.** Cancel and supersede mark rows inert and visible;
   no-ops book REFUSAL rows. A system that cannot show its refusals can
   be probed for free.
5. **Honest gaps, labeled.** Estimated or extrapolated state is marked,
   dimmed, and never presented as measurement.

## Status of the adaptation

| Piece | State |
|---|---|
| Upstream ordewell (planner, runners, CLI, VS Code, web) | works — see commands below |
| Quilt-first design | **shipped for review** — [docs/QUILT-FIRST.md](docs/QUILT-FIRST.md), PR #1 |
| `QuiltPlanStore` adapter (kernel+WAL spine) | designed; not yet built |
| Receipts on transitions / VIEW dashboards / plan rooms | phased, see design doc |

## Run it

```bash
npm install
npm run build        # core → vscode → cli → web (workspaces)
npm test             # all workspace test suites
npm run typecheck    # all workspaces
npm run start        # the TUI: node packages/cli/dist/main.js tui
```

## Upstream lineage

- Original project: [ordewell/ordewell](https://github.com/ordewell/ordewell) —
  © Alessandro Costanzo Ciano, [ordewell.ai](https://ordewell.ai).
- License: [Apache-2.0](LICENSE), retained for the upstream code.
- Fork discipline: fleet changes live in additive layers (new docs, a
  `core/plugins/quilt/` adapter when built) so upstream syncs stay clean.
  Upstream documentation remains the reference for the base product.

## The rest of this README

Everything below is upstream's original README — preserved because it
describes the product accurately and well.

---
