# CyberGuardian — Product Requirements Document (PRD)

Status: dev-stage plan, v1.0
Owner: Akash (solo; a friend may lend a laptop for an optional measurement)
Deliverables: a research paper and a working end-to-end demo
Time box: about 6 months for the final, plus about 2 days for the demo. Work is organised in dateless sprints (see `phases.md`).

Companion documents: `architecture.md` (system), `design.md` (low-level design), `memory.md` (memory layer), `rules.md` (engineering and safety rules), `phases.md` (sprints).

---

## 1. Summary

Software can be attacked at two moments: when a developer writes a vulnerable line, and after the application is live. CyberGuardian covers both in one system.

- **Shift-left (code review):** on every pull request, a code model (Qwen2.5-Coder with a trained soft prompt) reviews the changed Python code on a self-hosted runner. Findings appear as a PR comment and a pass/fail check inside GitHub. The developer reviews and merges. On merge, a snapshot of the application and a journal entry are written to a context database.
- **Runtime (agentic SOC):** logs from a demo web app are normalised and turned into time windows. A detector (LightGBM, optionally a GRU, plus Isolation Forest) classifies traffic. A team of LLM agents investigates each alert using the context memory and produces a report with a suggested fix. The agents only report. A developer reads the report on a dashboard, acknowledges it, or bans manually. Separately, the ML detector itself can auto-ban an IP for 15 minutes when its calibrated score is above a threshold.
- **Memory:** a context database links the two halves. What was learned at code-review time (open findings, routes, auth, developer decisions) is available to the runtime investigators, keyed by endpoint and commit.

## 2. Problem

1. Most tools cover only one phase: static analysis before deployment, or monitoring after it. Knowledge gained in one phase is not used in the other.
2. Alerts from log monitoring are noisy and lack code context. A human has to work out which route is affected, whether it was known to be weak, and what the fix is.
3. Teams lose the reasoning behind code decisions when the original authors leave. New staff cannot tell what a route is supposed to do or why it is unauthenticated.
4. LLM-based security agents are exposed to attacker-controlled text (log lines, code comments), so giving them power to act is risky.

## 3. Goals and non-goals

### Goals
- G1. Detect vulnerabilities in Python code changes on PRs, with a threshold calibrated to at least 90% precision.
- G2. Detect four attack classes in web logs (SQL injection, brute force, path traversal, scanning), including one class never seen in training.
- G3. Turn each alert into a clear report: what happened, which IP and endpoint, severity, suggested fix, and which memory items were used.
- G4. Test whether dev-time memory (snapshots, journal, knowledge base) improves investigation quality. This is a hypothesis, not a promise.
- G5. Keep LLMs unable to take actions, so prompt injection cannot cause damage beyond a misleading report.
- G6. Run fully self-hosted, so code and logs never leave the owner's machines.

### Non-goals
- Multi-tenancy, sign-up flows, billing, or a hosted SaaS.
- Detecting real network-layer DDoS (this is handled upstream by a CDN or firewall).
- Languages other than Python for code review.
- Full fine-tuning of any model.
- Attacking any system other than the owner's isolated lab.
- Production-grade scale. The target is a convincing, measured prototype.

## 4. Users

| User | Needs |
|---|---|
| **Developer / operator** (the project owner) | Clear PR feedback, a dashboard of reports, manual ban and acknowledge, a place to record decisions |
| **Reviewer / guide / paper reader** | Reproducible experiments, honest limitations, a demo that works end to end |
| **Future new staff member** (qualitative use case) | A journal and snapshots that explain how the system works and why |

## 5. Scope

**In scope (core):** shift-left review, runtime detection with four attack classes, report-only agent team, local small LLMs, context memory, dashboard, auto-ban by the detector, manual ban by the developer.

**Stretch (cut first if time runs short, in the order given in `phases.md`):** tier-2 auto-close of PRs, a fifth attack class (rate-limited application-layer flood), GRU sequence model (S5b), 7B model in 4-bit, LoRA baseline, agent LoRA fine-tune.

## 6. Functional requirements

### 6.1 Shift-left
- FR-S1. Trigger on pull requests and on pushes to main of a **private** repository.
- FR-S2. Extract changed Python functions from the diff and review them with Qwen2.5-Coder plus the trained soft prompt, running locally on a self-hosted runner inside a Docker container.
- FR-S3. Each finding contains: file, line, CWE type, confidence, explanation, suggested fix.
- FR-S4. A threshold gate (calibrated on validation data to at least 90% precision) decides pass or fail. Findings above it fail the check. Findings below it are warnings.
- FR-S5. Post one PR comment (the brief) and set the check status.
- FR-S6. Findings stay in GitHub. They are not sent to the API.
- FR-S7. When a PR is merged to main after the check passed (or after a manual override, which is recorded), write a snapshot and a journal entry through the API into the context database.
- FR-S8 (stretch). Tier 2: for severe finding types above a much higher calibrated precision, the bot also closes the PR with a comment. The author can reopen it. Branches and commits are never deleted.

### 6.2 Runtime
- FR-R1. The demo app writes one JSON log line per request, including `app_version` (the commit SHA).
- FR-R2. A log pipeline normalises events to the unified schema, orders them (short reorder buffer), saves raw logs as files by run ID, and builds per-IP and global windows.
- FR-R3. The detector classifies windows as normal, SQL injection, brute force, path traversal or scan (flood as stretch). Isolation Forest flags anomalies with no known class as "unknown anomaly".
- FR-R4. Auto-ban: when a class score is at or above the calibrated auto-ban threshold (at least 90% precision on validation data; fall back to a lower level such as 80% only if recall is too low), the system bans the IP for 15 minutes, skips allowlisted IPs, and logs the action. Unknown-anomaly alerts never auto-ban.
- FR-R5. Agent team (LangGraph): Triage posts the alert immediately, then Investigate, Suggest and Report run in order. Agents have no ability to execute commands or change enforcement.
- FR-R6. Every report contains: hypothesis, source IP, endpoint, severity (from the rubric), evidence events, suggested fix, whether an auto-ban already happened, and citations to every memory item used.
- FR-R7. The dashboard shows alerts and reports live, lets the developer acknowledge, ban, unban, add decision notes, and view snapshots and the journal.
- FR-R8. Manual ban path: Developer → Dashboard → API → Enforcement.

### 6.3 Memory
- FR-M1. Four layers: snapshots, operational records, journal, knowledge base and notes (see `memory.md`).
- FR-M2. Findings, incidents, snapshots and journal entries are retrieved by an exact join on commit SHA and method plus path. Vector search is used only for notes and the security knowledge base (top 5).
- FR-M3. Every memory item carries a source and a trust level. Memory can never trigger an action.
- FR-M4. The Reporter cites every memory item it used.

### 6.4 Platform
- FR-P1. FastAPI backend. PostgreSQL with pgvector (Neon during development, a local container for the demo).
- FR-P2. React dashboard (static build) behind a single admin login whose password comes from an environment variable. Bound to localhost or the lab network.

## 7. Non-functional requirements

| Area | Requirement |
|---|---|
| Privacy | Code, logs and models stay on the owner's machines. No hosted LLM is used for agents |
| Compute | Development laptops are CPU only. All soft-prompt tuning runs on free Kaggle GPU. Inference runs on CPU (slow is acceptable at dev stage; larger compute later) |
| Latency | Target: detector alert within 5 seconds of sufficient evidence. Report latency is measured and reported separately. Both are measured from attack start |
| Storage | Neon free tier is small. Raw logs live in files (Parquet or JSONL by run ID). The database holds only structured and text records |
| Reproducibility | Fixed seeds, pinned versions, run IDs, and model and prompt versions recorded in every alert |
| Safety | See `rules.md`. Attacks run only inside an isolated lab network |
| Offline demo | The demo works without internet using a local Postgres container and local models |

## 8. Success metrics and experiments

Numbers are measured, not promised. Thresholds are calibrated on validation data only.

| ID | Question | Method |
|---|---|---|
| E1 | Does the soft-prompted code reviewer beat baselines on unseen repositories? | Zero-shot Qwen, Semgrep, soft prompt (10, 20, 50 virtual tokens). Stretch: LoRA, 7B. Metrics: precision, recall, F1, false-positive rate. CVEfixes split by repository, deduplicated |
| E2 | Does runtime detection work per attack class and on an unseen class? | LightGBM on windows, GRU (S5b), both; Isolation Forest for the held-out class. Also report auto-ban precision (benign IPs wrongly banned) |
| E3 | How fast are alerts and reports? | 30 runs per attack class. Report attack to detector alert and attack to full report separately. Optional second-machine row |
| E4 | Does memory improve the report? | About 30 scenarios x 3 runs. Conditions: no memory; memory without snapshots; memory with snapshots; memory with snapshots plus soft-tuned model. Metrics: hypothesis correct (class, IP, endpoint), severity ranking vs rubric, false-positive rate on benign-but-unusual traffic. Compare soft-tuned vs untuned small local model |
| E5 | Can injected text in logs or code comments change agent output? | Planted injection samples; measure how often each agent is fooled |
| E6 | Can poisoned memory change reports? | Three poison types, with and without trust labels. Report auto-ban behaviour separately (it is driven by the detector, not by text) |

Decide metrics and analysis before seeing E4 results. A negative result is still a result.

## 9. Constraints and assumptions

- Solo, mostly. Hardware: one 16 GB and one 8 GB laptop (CPU only); a friend's 16 GB laptop is optional.
- Free resources: Kaggle GPU (about 30 hours a week), Neon free tier.
- Assumption A1: soft prompt tuning of Qwen2.5-Coder (1.5B, and 7B in 4-bit) works on a free Kaggle GPU. Untested; S2 is where this is found out.
- Assumption A2: the code-review soft prompt does not transfer to log investigation. A second soft prompt is tuned for log-investigation tasks, and an untuned baseline is kept for comparison.
- Assumption A3: simulated attacks are easier than real ones. The paper states this.
- Assumption A4: memory helps investigation. This is the hypothesis E4 tests.

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Soft prompt gives only a small gain over zero-shot | Weaker E1 | Report honestly; strong baselines make a modest result still publishable |
| Free GPU or storage limits | Slower tuning | Do S1 and S2 early; keep DB small; raw logs in files |
| Local CPU LLM is slow | Long report latency | Report latency separately; plan larger compute |
| Auto-ban false positives | Legitimate users blocked | 15-minute expiry, allowlist, calibrated threshold, report after every auto-ban |
| Shared CPU distorts latency | Noisy E3 | Run on a separate machine for a second row |
| Memory poisoning | Misleading reports | Trust levels, delimiters, memory never authorises action, E6 |
| Scope creep | Missed deadline | Sprint exit checks and the cut order in `phases.md` |

## 11. Decisions log

| Decision | Reason |
|---|---|
| Soft prompt tuning, not full fine-tuning | Far less compute; base model stays unchanged |
| Qwen2.5-Coder 1.5B for demo, 7B 4-bit as stretch | Fits CPU laptops and a free GPU |
| CVEfixes (Python), split by repository | Avoids leakage between near-identical functions |
| Findings stay in GitHub | Shift-left is a GitHub-native workflow; the DB only gets snapshots and journal at merge |
| Self-hosted runner in a container, private repo | Runner executes workflow steps; isolation limits risk |
| Window features + LightGBM first, GRU later | Strong CPU-friendly baseline; GRU tested as a comparison |
| Split by attack run; hold out one class | Prevents leakage; tests generalisation |
| Agents report only; detector may auto-ban | Actions are driven by an ML score, which cannot be prompt-injected |
| Local small LLMs for agents | Privacy and self-hosting; hosted comparison dropped |
| Structured memory with deterministic joins | Auditable and citable; vector search only for text |
| Neon (dev) with local Postgres container (demo) | Free tier for development; offline demo |
| Single-tenant, self-hosted | Matches the privacy argument; avoids months of multi-tenancy work |
| LangGraph for orchestration | Preferred by the owner; saved state per step |

## 12. Open questions

1. Auto-ban threshold: 90% calibrated precision is the default; 80% is the fallback if recall is too low.
2. Is tier-2 auto-close of PRs worth building? Default: optional stretch, never delete branches or commits.
3. Which small local LLM is the "similar small model" next to the soft-tuned Qwen? To be chosen at the start of S6.
4. How many scenarios does E4 need to show a difference? Start with about 30 and extend if variance is high.

## 13. Glossary

- **Shift-left:** moving security checks earlier in the development timeline.
- **SOC:** Security Operations Center. Here, an agentic one: LLM agents do the investigation.
- **Soft prompt tuning:** training a small block of virtual-token vectors placed in front of the input, while the base model stays frozen.
- **Snapshot:** a structured, code-extracted description of the application at a commit.
- **Journal:** a log of PRs and human decision notes explaining why the code is the way it is.
- **Window:** a time slice of events for one IP (or for all IPs) from which features are computed.
- **Held-out class:** an attack class removed from training and used only for testing.
- **Report-only agent:** an agent that can read memory and write a report, but cannot run commands or change enforcement.
