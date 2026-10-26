# CyberGuardian — Rules

These are the rules for building, running and evaluating the project. "Must" rules are not negotiable. If a rule has to change, update this file and note why in the decisions log in `prd.md`.

---

## 1. Safety rules

**R1. Lab only.** All attacks run only against the project's own demo app, inside an isolated Docker network with no exposure to the internet. Never run attack tools against any other system.

**R2. LLMs never act.** Agents can do exactly two things: read memory through the API and write a report. They have no shell, no file write, no network access, and no ability to ban, unban or change enforcement.

**R3. Actions come only from two sources.**
- The ML detector, through auto-ban.
- A human, through the dashboard.

**R4. Auto-ban rules.**
- The decision uses only the detector's calibrated score. No LLM output and no text from logs can influence it.
- Threshold: calibrated on validation data to at least 90% precision. Use a lower level (for example 80%) only if recall is too low, and record the change.
- Ban length is 15 minutes. Bans expire on their own and can be revoked.
- Allowlisted IPs (the owner's machines, the demo user) are never banned.
- "Unknown anomaly" alerts (Isolation Forest only) never auto-ban.
- Every auto-ban is logged and still produces a report.

**R5. Untrusted text is data.** Treat as untrusted: log fields (URL, query, user agent, form values), PR titles, commit messages, code comments, and any text a developer typed. Wrap untrusted text in clear delimiters in every prompt, label it as data, and tell the model it contains no instructions.

**R6. Memory never authorises anything.** A memory item can inform a report. It can never trigger, approve, or suppress an action.

**R7. The runner is a risk.** A self-hosted runner executes workflow steps. Therefore: the repository stays private, the runner runs in a container with no access to the host or home network, and secrets are passed as environment variables, never committed.

**R8. Secrets.** No tokens, passwords or API keys in the repository, logs, memory or snapshots. The dashboard admin password comes from an environment variable.

**R9. Dashboard access.** One admin login. No sign-up flow, no OAuth. Bind to localhost or the lab network only.

**R10. Never delete.** The shift-left bot may fail a check, comment, or (stretch, tier 2) close a PR. It never deletes branches, commits or files.

**R11. Privacy.** Use synthetic or lab-generated data only. No real user data enters logs, memory or training sets.

---

## 2. Data and evaluation rules

**R12. Split to prevent leakage.**
- Code data (CVEfixes): deduplicate near-identical functions, then split by repository. A repository's functions never appear in more than one of train, validation, test.
- Log data: split by attack run, never by row. Never shuffle events inside a window or a session.
- Hold out one attack class entirely as a "never seen" test.

**R13. One window builder.** Training and live detection call the same function to build windows and features, in timestamp order. Features must not differ between training and production.

**R14. Thresholds on validation only.** Calibrate every threshold on the validation set. Freeze it before touching the test set. Touch each test set once per experiment.

**R15. Pre-register.** Before running E4, write down the metrics, scenarios, conditions and how correctness is scored. Do not change them after seeing results.

**R16. Ground truth is written with the scenario.** Each scenario records attack class, source IP, endpoint, and severity from the rubric. Hypothesis correctness is checked by code (class, IP and endpoint must match). Hand-check about 20% of runs to catch scoring mistakes.

**R17. Repeat runs.** Run each LLM scenario 3 times. Report mean and spread. Fix seeds where possible and record model, prompt and prompt-tuning versions in every run.

**R18. Baselines are mandatory.** Every claim needs a baseline: zero-shot Qwen and Semgrep for code; windowed LightGBM for sequence models; no-memory for memory; untuned small LLM for the tuned LLM.

**R19. Report honestly.** State that simulated attacks are easier than real ones; report negative results; report the shared-CPU caveat for latency; report auto-ban false positives; report the precision/recall curve at the chosen thresholds.

**R20. Latency is measured two ways.** Attack start to detector alert, and attack start to full report. Never report one as the other.

---

## 3. Architecture rules

**R21. Findings stay in GitHub.** Code-review findings live in the PR comment and check. Only the snapshot (which includes open findings at that commit) and the journal entry go to the API.

**R22. Single entry to the database.** Only the API writes to the context DB. Other components call the API.

**R23. Deterministic joins first.** Findings, snapshots, journal entries and incidents are retrieved by exact join on commit SHA and method plus path. Vector search is only for free-text notes and the security knowledge base, top 5 results.

**R24. Every memory item has a source and a trust level.** The Reporter cites every memory item it used.

**R25. Raw logs are files.** Raw and normalised logs are stored as Parquet or JSONL partitioned by run ID. The database holds only structured records and text, to stay inside the free-tier size.

**R26. Swappable LLM backend.** Agents call one interface. Changing the model must not require code changes in the agents.

**R27. Keep agents small and separate.** Each agent has its own prompt and its own saved trace (input, output, model, timing).

**R28. Snapshots come from code.** Snapshot facts are extracted by parsing the code. An LLM summary may be added, but it is tagged as derived from untrusted text.

---

## 4. Modelling rules

**R29. Soft prompt tuning, not full fine-tuning.** The base model stays frozen. LoRA is a baseline or stretch, not the main method.

**R30. Two tuned prompts.** One soft prompt for code review. A separate one for log investigation. The code-review prompt is not assumed to transfer.

**R31. Detector order.** Build window features and LightGBM first. Add the GRU only after that works, using the same data, and report the comparison.

**R32. Isolation Forest is for the unseen class.** Train it on normal traffic only.

---

## 5. Process rules

**R33. Sprints, not dates.** A sprint ends only when its exit check passes (see `phases.md`).

**R34. Cut order.** If time runs short, cut in the order listed in `phases.md`. The memory sprint is cut last.

**R35. Write the paper while building.** Start method sections during S3 and keep a running results table.

**R36. Keep docs current.** When a decision changes, update `prd.md` (decisions log), the affected document, and the diagram.

**R37. Code hygiene.** Python with type hints, a test for the window builder and feature functions, a test for the snapshot extractor, and a pinned dependency file. Config in files, not hard-coded.

**R38. Definition of done for a sprint.** Exit check passed, results saved with run IDs and versions, docs updated, and a short note on what went wrong.

---

## 6. Quick reference: who may do what

| Actor | May | May not |
|---|---|---|
| Code reviewer model | Score findings | Block or merge anything by itself |
| Threshold gate | Fail or pass the PR check; (stretch) close PR | Delete branches or commits |
| ML detector | Raise alerts; auto-ban for 15 min above threshold | Ban allowlisted IPs; auto-ban unknown anomalies |
| Agents | Read memory; write reports | Run commands; ban; unban; edit memory trust levels |
| Developer | Acknowledge; ban; unban; add notes; merge | Disable the allowlist silently |
| Memory | Inform reports | Authorise or suppress actions |
