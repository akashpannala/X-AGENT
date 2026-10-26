# CyberGuardian — Phases (sprints without dates)

Each sprint ends on a pass-or-fail **exit check**. Move on only when it passes. There are no calendar dates: the order and the exit checks are the plan. About 6 months are available for the final, plus about 2 days for the demo.

Companion documents: `prd.md`, `architecture.md`, `design.md`, `memory.md`, `rules.md`.

---

## Overview

| Sprint | Name | Output |
|---|---|---|
| S0 | Foundations | Repo, Compose skeleton, schemas, empty LangGraph |
| S1 | Code data and baselines | Dataset, splits, Semgrep and zero-shot results |
| S2 | Soft prompt tuning (code) | Tuned prompt, calibrated threshold, E1 results |
| S3 | Push loop end to end | PR review, check, merge writes snapshot and journal |
| S4 | Lab and data | Demo app, attacks, benign traffic, labelled dataset |
| S5 | Runtime detector | Window features, LightGBM, Isolation Forest, auto-ban, E2 |
| S5b | Sequence model (GRU) | GRU vs LightGBM comparison |
| S6 | Agents and reports | Four report-only agents, dashboard reports, manual ban, E3 |
| S6b | Context memory | Memory layers, scenarios, soft prompt for logs, E4 |
| S7 | Hardening | E5, E6, isolation checks |
| S8 | Paper and demo | Paper, figures, demo rehearsal |

Start writing the paper's method sections during S3 and keep a running results table (rule R35).

---

## S0 — Foundations

**Goal:** a working skeleton that every later sprint plugs into.

Tasks:
- Create the repo (private), folder layout, pinned dependencies, config files.
- Docker Compose skeleton: API, database, dashboard placeholder.
- Define the unified log schema (including `app_version`) and the Postgres schema from `design.md` (including pgvector).
- API skeleton with the admin login from an environment variable.
- Empty LangGraph with four placeholder nodes.
- Choose the local small LLM backend interface (swappable).

**Exit check:** `docker compose up` works, and the dashboard shows a test alert posted through the API.

Risks: scope creep into features. Keep it empty on purpose.

---

## S1 — Code data and baselines

**Goal:** a clean dataset and honest baselines.

Tasks:
- Download CVEfixes and select the Python subset.
- Deduplicate near-identical functions. Split by repository into train, validation, test.
- Build a metrics harness: precision, recall, F1, false-positive rate, precision/recall curve.
- Run Semgrep and zero-shot Qwen2.5-Coder.

**Exit check:** a baseline results table on the test split, with the split recorded and reproducible.

Risks: label noise in CVEfixes; leakage if the split is wrong.

---

## S2 — Soft prompt tuning (code review)

**Goal:** the code reviewer and its threshold.

Tasks:
- Tune the soft prompt on Kaggle with the 1.5B model. Try 10, 20 and 50 virtual tokens.
- Calibrate the pass/fail threshold on the validation set to at least 90% precision.
- Export the prompt and verify it loads and runs on a CPU laptop.
- Stretch: 7B in 4-bit; LoRA baseline.

**Exit check:** E1 table (zero-shot, Semgrep, soft prompt variants) with precision, recall, F1 and false-positive rate on the test set, plus the frozen threshold.

Risks: modest gain over zero-shot (report it honestly); free GPU limits.

---

## S3 — Push loop end to end

**Goal:** a PR is reviewed and a merge writes memory.

Tasks:
- GitHub Action and self-hosted runner in a container (private repo, branch protection requiring the check).
- Runner extracts changed functions, calls the code reviewer, applies the gate.
- Post the PR comment (brief) and set the check status. Findings stay in GitHub.
- Developer-merge flow, including a recorded override of a failed check.
- Snapshot + journal writer runs on merge to main and posts to the API.
- Minimal dashboard views: snapshot viewer and journal list.
- Stretch (tier 2): auto-close PR for severe, very high-precision findings, reopenable; never delete.

**Exit check:** a PR with a planted SQL injection gets a comment and a failed check. After merging a clean PR, a snapshot and a journal entry appear in the database and on the dashboard.

Risks: runner security; model too slow on CPU for large diffs (limit to changed functions).

---

## S4 — Lab and data

**Goal:** a labelled runtime dataset.

Tasks:
- Flask demo app writing JSON logs with `app_version`.
- Benign traffic generator (logins, browsing, searches) at several times attack volume, including "benign but unusual" behaviour (nightly job, load test, mistyped passwords).
- Four attack scripts: SQL injection (sqlmap), brute force (Hydra), path traversal, scanning (gobuster or nikto). Run only inside the isolated lab.
- Log pipeline: normalise to the unified schema, reorder buffer, save raw logs by run ID, per-IP ring buffers, windows (10 s, 60 s, 5 min), global window.
- Build the dataset with labels from the scripts. Split by attack run. Hold out one class.
- Stretch: rate-limited application-layer flood from lab containers.

**Exit check:** a labelled dataset with run IDs, a train/validation/test split by run, one held-out class, and a test that the training and live window builders produce identical features on the same log file.

Risks: attacks too clean; leakage between runs.

---

## S5 — Runtime detector

**Goal:** detection and auto-ban.

Tasks:
- Per-request features and per-IP window features (rate, 4xx share, distinct paths, failed logins, gap statistics, status-code runs), plus global features if the flood class exists.
- Train LightGBM. Train Isolation Forest on normal traffic only.
- Calibrate the auto-ban threshold on validation data to at least 90% precision (fallback: lower only if recall is too low, and record it).
- Auto-ban via Enforcement: 15-minute expiry, allowlist, logging, unban from the dashboard.
- Optional late ablation row: add the feature "endpoint has an open finding".

**Exit check:** E2 table per attack class, including the held-out class, with auto-ban precision (benign IPs wrongly banned) and the detector-alert latency (attack to alert).

Risks: low recall at 90% precision; shared-CPU noise.

---

## S5b — Sequence model (GRU) — stretch

**Goal:** test whether order-aware modelling adds value.

Tasks:
- Encode each event (method, path template, status class, size bucket, time since previous event). Use the last ~50 events per IP or session.
- Train a small GRU on the same data and splits. Optionally add a next-event predictor or sequence autoencoder as an unseen-attack test.
- Compare: windowed LightGBM, GRU, both.

**Exit check:** an E2 extension with the three-way comparison on identical splits.

Cut first among runtime items if time is short.

---

## S6 — Agents and reports

**Goal:** report-only agents with a dashboard.

Tasks:
- LangGraph with four agents: Triage (posts the alert immediately), Investigator, Suggest, Reporter. State saved after every step. Per-node timeouts; a failed step leaves the report marked incomplete.
- Local small LLM backend: the soft-tuned model and an untuned baseline, behind one interface. Choose the "similar small model" now.
- Dashboard: live alerts, report detail (IP, endpoint, evidence, suggested fix, auto-ban status, citations placeholder), acknowledge, manual ban and unban.
- Manual ban path: Dashboard → API → Enforcement.
- Severity rubric implemented (see `design.md`).

**Exit check:** a simulated attack produces a dashboard alert, then a full report; manual ban works and expires; an agent cannot call any action tool (tested). E3 numbers: attack to alert and attack to report, 30 runs per class.

Risks: slow CPU inference; LLM output not parsing. Use structured output with validation and a retry.

---

## S6b — Context memory

**Goal:** test the memory hypothesis.

Tasks:
- Implement the memory layers from `memory.md`: snapshots and diffs, operational records, journal (automatic PR entries, human notes through the dashboard, monthly derived overview), knowledge base and notes with pgvector.
- Deterministic joins on commit SHA and method plus path; vector top 5 for notes and the knowledge base.
- Trust levels, delimiters and citations in reports.
- Tune a second soft prompt for log investigation on Kaggle.
- Build about 30 scenarios in three families plus a few "recent PR added an unauthenticated route" scenarios. Write ground truth for each.
- Pre-register the metrics, then run each scenario 3 times under four conditions: no memory; memory without snapshots; memory with snapshots; memory with snapshots plus soft-tuned model.
- Compare soft-tuned versus untuned small local model.

**Exit check:** E4 results table with hypothesis correctness, severity ranking error and false-positive rate on benign-but-unusual traffic, plus a hand-checked 20% sample.

Risks: no measurable difference (still a result); scenario quality; variance. Add scenarios if variance is high.

---

## S7 — Hardening

**Goal:** show the safety design holds.

Tasks:
- E5: prompt-injection samples in log fields and code comments; measure how often each agent is fooled.
- E6: three poison types (a note saying an endpoint is "safe, ignore alerts"; a fake accepted-risk entry; a commit message with embedded instructions), with and without trust labels. Report auto-ban behaviour separately because it depends on the detector.
- Verify runner isolation, secrets handling, dashboard login, allowlist, and that bans expire.

**Exit check:** E5 and E6 tables, plus a checklist signed off for rules R1 to R11.

---

## S8 — Paper and demo

**Goal:** the final deliverables.

Tasks:
- Figures, tables, limitations section, related work, reproducibility notes.
- Demo script and fixtures: a PR with a planted vulnerability, merge, a simulated attack, alert, report, manual ban.
- Offline mode: local Postgres container with pgvector, local models, no internet.
- Two-day demo rehearsal, including a recovery plan if a model or service is slow.

**Exit check:** the full demo runs end to end offline, twice in a row, and the paper's tables match the saved run IDs.

---

## Cut order if time runs short

Cut in this order. The first item goes first.

1. Tier-2 auto-close of PRs
2. Fifth attack class (flood)
3. S5b GRU sequence model
4. Agent LoRA fine-tune
5. LoRA baseline for code review
6. 7B run
7. Breadth of E5 and E6 (keep the core cases)
8. Optional detector ablation ("open finding" feature)
9. Memory sprint last, because it is the most original contribution

Never cut: rules R1 to R11, the held-out class, baselines, calibrated thresholds, E1 to E3.

---

## Dependencies

```
S0 -> S1 -> S2 -> S3
S0 -> S4 -> S5 -> S5b
S3 + S5 -> S6 -> S6b -> S7 -> S8
S2 feeds S3 (soft prompt). S4 feeds S5 and S6b (data and scenarios).
```

S1 and S2 are early on purpose: free GPU limits or a weak result should show up before anything depends on them.
