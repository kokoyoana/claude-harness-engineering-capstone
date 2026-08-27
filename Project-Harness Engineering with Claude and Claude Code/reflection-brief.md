# Reflection Brief - Harness Engineering Capstone

**Name:** Joanna Castillo  
**Date:** 2026-08-27

## Environment

- **Model(s):** `claude-haiku-4-5-20251001`
- **OS / Python:** Windows 11, Python 3.11.9
- **Approx. API spend:** `$0.1416` for the final System 1 run, plus the System 2 inference calls. System 4 used a recorded response and did not require a live model call.
- **Evidence repository:** `evidence/`

---

## Part 1 - Per-system

### System 1 - Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

   The trace `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl` records this sequence:

   ```text
   tool_use -> tool_use -> tool_use -> tool_use -> end_turn
   ```

   Turns 1 through 4 invoked `lookup_policy`, `record_claim_fact`, `classify_claim`, `assess_severity`, and `route_to_adjuster`; turn 5 returned `end_turn` with no further tool calls. The decision is implemented by `run()` in `claims_intake/loop.py`: it continues for `stop_reason == "tool_use"`, returns for `stop_reason == "end_turn"`, and raises `UnexpectedStopReason` for any other value. Evidence: `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl` and `evidence/system1/system1-test-output.txt`.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

   One checked anti-pattern is using string-membership tests against assistant text to control the loop. The tests also reject a fixed integer-literal iteration cap as the primary stopping mechanism. If the stolen-bike run searched generated prose for words such as "complete" or "done," a valid tool response might stop accidentally, or a final answer using different wording might continue unnecessarily. Evidence: `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl` and the 29 passing tests in `evidence/system1/system1-test-output.txt`.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

   `classify_claim` and `assess_severity` both accept a decision value and a rationale, but their schemas separate claim category from operational severity. In `claim_02_stolen_bike`, `classify_claim` received `claim_type: "theft"`, while `assess_severity` received `severity: "low"`, showing that the descriptions prevented misrouting. Structured errors use `is_error=true`, allowing the agent to identify the failed call, correct its inputs, and retry while preserving the audit trail; a generic exception string could terminate the workflow or hide which field was invalid. Evidence: turn 3 of `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl`.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

   In `evidence/system1/20260827_113440/summary.md`, `claim_02_stolen_bike` completed in 5 turns with an estimated cost of `$0.0206`. The claim was routed to the theft queue with low severity. Exact figures can differ from samples or earlier runs because model-selected tool sequences and token usage are nondeterministic; my final complete run cost `$0.1416`, while an earlier run cost `$0.1181`. Evidence: `evidence/system1/20260827_113440/summary.md`.

### System 2 - Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

   `evidence/system2/budget.json` reports 38,708 baseline tokens and 16,769 assembled tokens, a 56.68% reduction. The active section dominates at 15,789 tokens. It remains verbatim because it contains the unresolved payment-method conversation and the latest customer statements; compressing it could remove exact codes, identifiers, or current intent. Evidence: `evidence/system2/budget.json` and `evidence/system2/context.md`.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

   Resolved conversation segments are summarized, while the active segment is preserved byte-exact. In my run, `resolved_refund` used 336 tokens, `resolved_subscription` used 458 tokens, and the durable `case_facts` block used 204 tokens; the active segment remained at 15,789 tokens. This produced a 56.68% reduction while retaining the operationally sensitive conversation without alteration. Evidence: `evidence/system2/budget.json`, `evidence/system2/context.md`, and `evidence/system2/system2-test-output.txt`.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

   All six questions passed with the case-facts block in `evidence/system2/eval.jsonl`. In the control run, Q6 regressed: the expected exact status token was `in_progress`, but the model said no formal status token was present and paraphrased the case as active and unresolved. Q1 still passed because the `$22.14` refund amount remained available in the preserved narrative. This proves that a durable facts block protects exact structured state that cannot always be reconstructed reliably from prose. Evidence: Q6 in `evidence/system2/eval.jsonl` and `evidence/system2/eval_control.jsonl`.

### System 3 - Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

   The React rule in `.claude/rules/react.md` uses:

   ```yaml
   paths:
     - src/components/**/*
     - src/pages/**/*
   ```

   This loads React conventions only for components and pages. Path-scoped rules can express cross-cutting conventions over several locations and file patterns without duplicating instructions in multiple directory-level `CLAUDE.md` files. The validator confirmed the matching behavior, including files that match both surface-specific and testing rules. Evidence: `.claude/rules/react.md`, `.claude/rules/tests.md`, and `evidence/system3/system3-test-output.txt`.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

   The deploy-check skill declares:

   ```yaml
   context: fork
   allowed-tools:
     - Read
     - Grep
     - Glob
     - Bash(git status:*)
     - Bash(git diff:*)
     - Bash(git log:*)
     - Bash(git rev-parse:*)
     - Bash(git ls-files:*)
     - Bash(gh pr view:*)
     - Bash(gh pr checks:*)
   ```

   The fork keeps verbose discovery and diff output outside the main conversation; only the structured deployment summary returns. The read-only allowlist prevents edits, pushes, deployments, or migrations. Without the fork, transient diagnostics would consume the main context; without the allowlist, the validator could alter the repository it was meant to inspect. Evidence: `.claude/skills/deploy-check/SKILL.md`, `evidence/system3/claude-structure.txt`, and `evidence/system3/system3-test-output.txt`.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

    `python -m ecommerce_team_config .` returned `OK`, and the suite passed 35 tests. Project-level scope includes the committed root `CLAUDE.md`, `.claude/rules/*.md`, `.claude/commands/review.md`, and `.claude/skills/deploy-check/SKILL.md`. User-level scope covers personal Claude preferences and memory outside the versioned repository, which are intentionally not committed. Evidence: `evidence/system3/validator-output.txt`, `evidence/system3/system3-test-output.txt`, and `evidence/system3/CLAUDE.md`.

### System 4 - Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

    The seeded warm tier contained 40 defect rows, recorded in `evidence/system4/warm-store-summary.txt`. The runtime uses `WarmStore.defects_since(since_ts, limit=50)`, an indexed and bounded SQL query, instead of loading the entire table into Python or model context. The database has `idx_defects_ts` on `defects(ts)` and `idx_defects_shift_ts` on `(shift, ts)`. Filtering, ordering, and limiting happen inside SQLite, so the model receives only the shift-relevant subset rather than the complete history. Evidence: `evidence/system4/warm-store-summary.txt`, `evidence/system4/shift-output.txt`, and `evidence/system4/system4-test-output.txt`.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

    `recovery.py` defines a 30-minute staleness threshold. An incomplete manifest no older than 30 minutes can resume, while a complete run, missing manifest, or incomplete manifest older than 30 minutes starts fresh. A fresh start with captured findings injected as a summary is safer when working state is stale because old assumptions may no longer match current shift data. The tests verify the boundary exactly: 29 and 30 minutes resume, while 31 minutes starts fresh. Evidence: `evidence/system4/system4-test-output.txt`.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

    My generated `hot_state.json` is 658 bytes, recorded in `evidence/system4/hot-state-size.txt`. The hot tier carries only compact information needed to start the next shift, while the 40-row history remains in indexed SQLite. A strict byte budget matters because the monitor runs indefinitely; unbounded state would continuously increase context, latency, cost, and failure risk. Evidence: `evidence/system4/hot_state.json`, `evidence/system4/hot-state-size.txt`, `evidence/system4/warm-store-summary.txt`, and `evidence/system4/system4-test-output.txt`.

---

## Part 2 - Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.

    **Model:** `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl` records the model-facing layer through structured tool choices and `stop_reason` values. The model selected theft classification, low severity, and the theft queue, while the harness retained control of execution.

    **Harness:** System 3's `CLAUDE.md`, `.claude/rules/`, `.claude/commands/review.md`, and `.claude/skills/deploy-check/SKILL.md` define instructions, scope, tools, permissions, and reusable workflows. The 35 passing tests and `OK` validator result show that this layer is checked deterministically.

    **Orchestration:** System 4's `warm.sqlite`, `hot_state.json`, `shift_scratchpad.jsonl`, recovery rules, and invocation pipeline coordinate work across shifts. The 33 passing tests verify tiered state, bounded prompts, crash recovery, and independent forks. Evidence: `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl`, `evidence/system3/system3-test-output.txt`, and `evidence/system4/system4-test-output.txt`.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code and one guided by prompt. When is each right?

    System 4 guarantees atomic state persistence and the hot-state byte budget in code, verified by `test_hotstate_atomic_write` and `test_hotstate_roundtrip_and_size_budget`. System 1 uses prompts and tool descriptions to guide semantic decisions such as classifying the stolen bicycle as theft and choosing the theft queue. Code is appropriate for invariants that must never be optional, including storage safety, schemas, permissions, and budgets. Prompts are appropriate for contextual decisions requiring flexible semantic judgement within deterministic boundaries. Evidence: `evidence/system4/system4-test-output.txt` and `evidence/system1/20260827_113440/traces/claim_02_stolen_bike.jsonl`.

16. **Context, two faces.** Compare context management in System 2 and System 4 with cited numbers from both. Same principle, different mechanism - how?

    System 2 manages context inside one long conversation: it reduced 38,708 tokens to 16,769 tokens, a 56.68% reduction, by summarizing resolved sections while preserving the 15,789-token active section. System 4 manages context between repeated sessions: its hot state is 658 bytes, while 40 historical defect rows remain in indexed SQLite and each shift retrieves only a bounded subset. Both systems keep high-value information close to the model and move low-value history out of immediate context. System 2 uses conversation compression; System 4 uses hot, warm, and cold state tiers. Evidence: `evidence/system2/budget.json`, `evidence/system4/hot-state-size.txt`, and `evidence/system4/warm-store-summary.txt`.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

    System 4's `test_mid_write_read_reveals_prior_complete_lines` verifies that the scratchpad remains readable during a partial append and exposes only previously complete JSONL entries. A successful shift would not reveal behavior during an interrupted write or process crash. The suite also verifies `fsync`, stale-manifest boundaries, recovery decisions, and independent fork scratchpads. These guarantees matter because a system can appear correct during normal operation while corrupting the recovery state needed during failure. Evidence: `evidence/system4/system4-test-output.txt`.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch?

    The deploy-check skill in System 3 has a limited blast radius because it runs in `context: fork` and receives only read-oriented tools, bounded `git` commands, and read-only GitHub CLI queries. If its analysis is wrong, the consequence is an incorrect recommendation, not an edited repository or accidental deployment. The tool allowlist is the kill switch: the skill has no `Write`, unrestricted shell, push, deploy, or migration capability, and the main session decides whether to proceed. Evidence: `.claude/skills/deploy-check/SKILL.md`, `evidence/system3/system3-test-output.txt`, and `evidence/system3/validator-output.txt`.

---

## Part 3 - Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it.

    Environment setup failed before the systems ran successfully. Vocareum supplied Python 3.10.14 while the projects required Python 3.11+, so I installed Python 3.11.9 locally; Windows path-length limits then prevented virtual-environment creation until long-path support was enabled and Windows restarted. In System 3, pytest initially used the wrong interpreter and reported `No module named 'yaml'`; I created a separate `.venv`, installed `.[dev]`, verified PyYAML 6.0.3, and ran `python -m pytest`. The final evidence records 29 passing tests for System 1, 28 passed and 2 initially skipped for System 2, 35 passed plus `OK` for System 3, and 33 passed for System 4. Evidence: all four `evidence/system*/system*-test-output.txt` files.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

    I would add a first-class recorded-response mode to Systems 1 and 2, similar to the offline path used by System 4. System 1 produced different outcomes and costs across two live executions, and the selected final run cost `$0.1416`; System 2 also required live inference before creating its context and evaluation artifacts. Recorded responses would make CI and local regression testing deterministic and remove API credentials as a prerequisite for basic artifact generation. I would retain a separate live-model acceptance run because model variability is still operationally important. Evidence: `evidence/system1/20260827_113440/summary.md`, `evidence/system2/budget.json`, and `evidence/system4/shift-output.txt`.
