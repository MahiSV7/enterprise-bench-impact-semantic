# Enterprise-Bench Extension: Impact-Semantic Representation

Working plan for extending [Enterprise-Bench](https://github.com/devrev/enterprise-bench) with a third
"impact-semantic" condition and testing it against the existing Raw / Semantic-augmented setup.

Status legend: `[ ]` not started · `[~]` in progress · `[x]` done

---

## Phase 0 — Environment setup

- [ ] Confirm host prerequisites: Python 3.12+, Docker Desktop running, Harbor installed
- [ ] Clone `devrev/enterprise-bench` into this project (e.g. `vendor/enterprise-bench` or a sibling dir)
- [ ] Obtain and store API credentials (model provider) needed to run agent trials — outside git, in `.env` or local secrets file
- [ ] Read `SKILL.md` and `README.md` in full; note exact commands for: building containers, launching an MCP server for a task, invoking the agent, collecting trajectories/results
- [ ] Run the repo's own smoke test / example task (whatever they ship) to confirm the toolchain works end-to-end before touching our own code

**Exit criterion:** you can run *some* stock Enterprise-Bench task and get a scored result back.

---

## Phase 1 — Baseline reproduction (no modifications)

- [ ] Identify and run `eng-l2-b` ("rank open engineering issues by potential revenue impact, joining paying accounts with open tickets on the same product area") completely unmodified
- [ ] Capture: task pass/fail, full agent trajectory, tool calls, token usage
- [ ] Repeat for at least 3 trials to confirm the harness is stable and results are reproducible (don't need all 10 yet)
- [ ] Document the exact invocation command(s) in this repo (a `README.md` or `Makefile` target) so later runs are one command

**Exit criterion:** one unmodified `eng-l2-b` baseline run, reproduced ≥3x, with saved trajectories.

---

## Phase 2 — Define the three conditions precisely

- [ ] Write a short spec (`docs/conditions.md`) defining exactly what data/context each condition receives for a given task:
  - **Raw** — existing records + schema only (Enterprise-Bench default)
  - **Semantic** — Raw + governed definitions / relationships / business concepts (what fields mean, how entities relate)
  - **Impact-semantic** — Semantic + an explicit, deterministically-derived impact/consequence signal
- [ ] Specify the impact function precisely, e.g. for `eng-l2-b`:
  `I(s) = f(severity, ARR_exposure, opportunity_exposure, status)`
  — write the actual formula, the fields it reads (confirm they already exist in Enterprise-Bench's ticket/account/opportunity schema), and how it's computed (offline preprocessing step, not new ground-truth info)
- [ ] Explicit check: verify no condition receives information unavailable to the others — Impact-semantic must be a *re-representation* of facts already present in Raw, not new data
- [ ] Get sign-off (self-review or advisor) on the spec before implementing — this is the crux of the paper's internal validity

**Exit criterion:** a written spec that could be handed to someone else and they'd implement the same three conditions.

---

## Phase 3 — Implement the representation layer

- [ ] Write a preprocessing/augmentation script that takes Enterprise-Bench's raw task data and emits three variants (Raw passthrough, Semantic-augmented, Impact-semantic-augmented)
- [ ] Implement the semantic layer: governed field/entity/relationship definitions injected into context (as system prompt content, tool descriptions, or retrievable docs — decide which and keep it consistent across tasks)
- [ ] Implement the impact layer: the deterministic scoring function from Phase 2, exposed the same way (not as a magic pre-computed answer — as a derived signal the agent can read/use)
- [ ] Unit-test the impact function against a handful of hand-checked examples from `eng-l2-b`'s data to confirm it computes sane, expected values
- [ ] Wire the harness so a single config flag selects which of the 3 conditions a run uses, without touching task logic

**Exit criterion:** can run `eng-l2-b` under Raw, Semantic, or Impact-semantic by flipping one config value.

---

## Phase 4 — Metrics & scoring

- [ ] Adopt Enterprise-Bench's native metrics as-is: task pass rate, reliability/pass@k, token efficiency, retrieval precision
- [ ] Implement our additional metrics on top of saved trajectories:
  - consequential-signal ranking accuracy
  - evidence recall
  - investigation trajectory quality (steps to correct answer, backtracking)
  - tool-call count
  - unsupported-claims rate
  - impact-prioritization accuracy (vs. the ground-truth ranking `eng-l2-b` already scores against)
- [ ] Write a single scoring script that ingests a batch of trajectories + condition label and outputs a row per (task, condition, trial) with all metrics

**Exit criterion:** given a folder of raw trajectories, one command produces a clean results table (CSV/parquet).

---

## Phase 5 — Pilot experiment

- [ ] Run `eng-l2-b` × {Raw, Semantic, Impact-semantic} × 5 trials each (15 runs) as a cheap pilot
- [ ] Score with the Phase 4 pipeline
- [ ] Sanity-check: does Impact-semantic show *any* directional signal over Semantic? Does Semantic beat Raw as expected (replicating Enterprise-Bench's own findings)?
- [ ] Decision gate:
  - Signal present → proceed to Phase 6 (full experiment)
  - No signal / noisy → revisit the impact function (Phase 2/3) or task choice before spending more budget

**Exit criterion:** a go/no-go decision backed by pilot data, made before the expensive run.

---

## Phase 6 — Full experiment

- [ ] Select the full task subset (start with the L2 analytical tasks most amenable to an impact signal — e.g. issues-to-revenue, ARR-at-risk, expansion-deal blockers, account risk/opportunity synthesis; expand to L1 retrieval tasks only if pilot suggests it matters there too)
- [ ] Run selected tasks × 3 conditions × 10 trials (matching Enterprise-Bench's own methodology: 10 trials/task) — budget the token/API cost before starting
- [ ] Also run the tool-interface variable Enterprise-Bench already exposes (curated tools vs. protocol-realistic APIs) crossed with our 3 conditions, if budget allows — this is the strongest "another architectural variable" framing for the paper
- [ ] Store all trajectories + scored results; keep raw trajectories archived (not just aggregates) in case reviewers ask for trajectory-level evidence

**Exit criterion:** complete results table covering all planned (task × condition × trial) cells.

---

## Phase 7 — Analysis

- [ ] Aggregate pass rate, pass@k, token cost, and our custom metrics by condition, with confidence intervals across the 10 trials
- [ ] Statistical test for Semantic vs. Raw and Impact-semantic vs. Semantic (paired, since same tasks) — pick an appropriate test (e.g. paired bootstrap or Wilcoxon) given small trial counts
- [ ] Break out results by task type (ranking/prioritization tasks vs. pure retrieval) — hypothesis is the impact layer should help more on prioritization-shaped tasks
- [ ] Produce figures: accuracy by condition, token cost by condition, per-task breakdown
- [ ] Write up findings honestly, including if Impact-semantic doesn't help on some tasks — that's a real result too

**Exit criterion:** a results section draft with numbers, tables, and figures ready to drop into the paper.

---

## Phase 8 — Paper integration

- [ ] Replace the old compliance-example framing with the Enterprise-Bench-grounded setup throughout
- [ ] Add correction/clarification that the dataset is synthetic-but-production-scale, citing the repo
- [ ] Write up the three-condition methodology and the "no information advantage" argument explicitly (reviewers will ask)
- [ ] Position against Enterprise-Bench's own tool-interface finding (18–19pp swing) as related-but-distinct architectural axis
- [ ] Cite Enterprise-Bench's methodology (10 trials/task, 14 tasks, 140 observations) as the basis adopted/extended

---

## Phase 9 — Optional: contribute back

- [ ] If results are solid, package the impact-semantic task extension (data augmentation scripts, new/modified tasks, scoring code) per Enterprise-Bench's contribution guidelines
- [ ] Open a PR / discussion with devrev/enterprise-bench proposing it as a community task extension
- [ ] Only after paper submission/acceptance timeline allows — don't block the paper on upstream review

---

## Immediate next action

Start Phase 0 → Phase 1. The first concrete milestone is: **one unmodified `eng-l2-b` baseline run working locally.**

```bash
git clone https://github.com/devrev/enterprise-bench.git
```
