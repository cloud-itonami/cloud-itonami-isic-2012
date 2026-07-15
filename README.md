# cloud-itonami-isic-2012: Manufacture of fertilizers and nitrogen compounds

Open Business Blueprint for **ISIC Rev.5 2012**: manufacture of fertilizers and nitrogen compounds — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **plant operations**: production-batch data logging (product-grade / weight / N-P-K nutrient-content), ammonia-synthesis/granulation-line maintenance scheduling, safety-concern flagging, and outbound fertilizer/nitrogen-compound shipment coordination.

This repository designs a forkable OSS business for
fertilizer-and-nitrogen-compound plant operations: run by a qualified
operator so a plant keeps its own operating records instead of
renting a closed SaaS.

## Scope: fertilizers and nitrogen compounds

ISIC 2012 covers the chemical-process plant that synthesizes ammonia
(or takes ammonia/nitric-acid feedstock) and, via granulation/prilling
lines, produces straight nitrogen fertilizers (urea, ammonium nitrate,
ammonium sulfate, calcium ammonium nitrate, UAN solution), phosphate
fertilizers (DAP, MAP, superphosphates) and blended N-P-K compound
fertilizers, labeled by their own nitrogen (N), phosphate (P2O5) and
potash (K2O) nutrient content. This is a heavily regulated industrial
input — not a product engineered to harm its own end user — but one
with a real chemical-process hazard profile distinct from a purely
mechanical plant: ammonia exposure (toxic-release risk) and
ammonium-nitrate explosion risk are well-documented industrial hazard
classes for exactly this ISIC vertical, plus an environmental-release
hazard (nitrate runoff / groundwater contamination) this actor's
safety-concern flag also covers.

## What this actor does

Proposes **plant operations coordination**, not equipment operation:
- `:log-production-batch` — synthesis/granulation batch, product-grade and N-P-K nutrient-content data logging (administrative, not an operational decision)
- `:schedule-maintenance` — ammonia-synthesis/granulation-line maintenance scheduling proposal
- `:flag-safety-concern` — surface a chemical-hazard (ammonia exposure, ammonium-nitrate explosion risk)/environmental concern (always escalates)
- `:coordinate-shipment` — outbound fertilizer/nitrogen-compound shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-critical domain**
(ammonia-synthesis/granulation-line equipment, ammonia-exposure and
ammonium-nitrate-explosion-risk chemical process hazard):

- Does NOT control the ammonia-synthesis reactor or granulation line directly
- Does NOT make plant-safety, chemical-safety or environmental-release decisions (that's the plant supervisor's exclusive human authority)
- Does NOT actuate the ammonia-synthesis reactor or granulation line (human plant supervisor decides)
- ONLY proposes/coordinates operations back-office; all actuation requires explicit human approval
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`fertmfg.operation/build`, a langgraph-clj StateGraph):
1. **`fertmfg.advisor`** (sealed intelligence node, `FertAdvisor`): proposes decisions only, never commits
2. **`fertmfg.governor`** (independent, `Fertilizer Plant Operations Governor`): validates against domain rules, re-derived from `fertmfg.registry`'s pure functions and `fertmfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct synthesis/granulation-line-equipment control)
     - Directly actuating the ammonia-synthesis reactor or granulation line (`:actuate-line? true`) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped weight past its own logged production weight (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:product-grade` value on a production-batch patch
     - No physically implausible N-P-K nutrient-content label (`:n-percent`/`:p2o5-percent`/`:k2o-percent`) on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`fertmfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`fertmfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
