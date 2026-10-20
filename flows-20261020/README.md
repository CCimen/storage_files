# Eneo Flows: findings, plan and validated backlog (2026-10-20)

Flows-only analysis (Flows core, runtime, publish/validation, run history, webhooks, contract checks, packages, E2E/CI), written so it holds whether or not the Flow AI Builder is kept.
- Code: integration branch `feature/flows-and-builder` at **044ec559c**.
- Paths are repo paths. Local absolute paths (`/home/...`, `/tmp/flows-tip-read`) refer to the author's machine.

## Files
| File | What |
|---|---|
| [FLOWS-FINDINGS-AND-PLAN.md](FLOWS-FINDINGS-AND-PLAN.md) | Capabilities today, what landed 2026-10-19..20, findings ranked by consequence, the "Del för del" assessment, the plan in ROI order (each item adversarially validated), evaluation, lessons, how to resume |
| [FLOWS-BEADS-VALIDATED.md](FLOWS-BEADS-VALIDATED.md) | 102 Flows backlog items (beads), each validated against the code: still true or fixed, real improvement or not, category, priority check, file:line evidence |
| [flows-beads-validated.json](flows-beads-validated.json) | The same data, machine-readable (id, title, priority, status, description, validation) |

## How it was validated
- **Findings:** three dedicated passes (an inventory, a verification of the "Del för del" proposal, and a Flows-only extraction from the day's 8-reader architecture review and the Flows quality program), then a synthesis. An independent check found 14 issues in the first draft (wrong citations, two non-Flows commits, one false claim); all were fixed against the code.
- **Plan items:** an adversarial reviewer checked each one. Result: validated, revised or dropped, with file:line evidence at 044ec559c.
- **Beads:** 10 independent reviewers checked 117 candidates. Of these, 15 were Builder-only and excluded, leaving 102 Flows items:
  - **57 validated real improvements:** 14 bugs, 9 robustness, 9 quality, 8 maintainability, 8 features, 5 test debt, 3 docs, 1 security;
  - **19 recommended to close:** already fixed, or the premise is gone;
  - **26 true but not recommended now:** speculative, umbrella items or undecided design.

## Headlines
- **Highest-ROI fixes are real bugs that look like success or a server error:**
  1. publish answers 500 for a bad per-source step (should be a typed 400);
  2. an HTTP 3xx counts as success for webhooks and HTTP input;
  3. an `http_get` step with template_fill never fetches;
  4. a first step reading `föregående_steg` passes publish and fails every run.
  Then the deploy shutdown pair: SIGKILL mid-drain, followed by a 180 s stall before a run is marked failed.
- **No single owner decides which output part carries a step's content.** Publish decides by output type plus contract; the run decides by payload shape. They agree today only by coincidence.
- **"Del för del" after speaker naming is refused**, both at publish and at run, because the speaker step's output type is JSON although its text is the named transcript.
  - A zero-code path may already work (`{{ transkribering }}`); probe it first.
  - If a code change is needed, it is one Flows-core owner, decided per read path (a `.structured` read of a speaker step must stay structured), with publish and run changed together.
- **A run ends "completed" even when a model step got empty or wrong input.** There is no typed signal for it.
