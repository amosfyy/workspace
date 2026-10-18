# Reflection Brief — Harness Engineering Capstone

**Name:** Amos Fang
**Date:** 18 October 2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): `claude-haiku-4-5-20251001`
- OS / Python: 3.13.0
- Approx. API spend: US$0.1419

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In `runs/20261015_075030/traces/claim_03_water_damage.jsonl`, the `stop_reason` sequence is `tool_use → tool_use → tool_use → tool_use → tool_use → tool_use → end_turn` across turns 1–7. `claims_intake/loop.py`, function `run()`, controls continuation by looping while the response's `stop_reason` is `tool_use`; it returns on `end_turn` and raises `UnexpectedStopReason` for any other value. The trace shows the model asking for clarification at turn 3, classifying and assessing the claim at turns 4–5, routing it at turn 6, and finishing at turn 7. This is better than inferring completion from the presence or absence of a tool call because the API's explicit stop signal controls the loop.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One checked anti-pattern is a hard-coded integer iteration cap; `tests/test_antipatterns.py::test_no_integer_literal_iteration_cap_in_loop` checks for it. A fixed cap could terminate a valid but longer claim workflow before it reaches `end_turn`, or force an arbitrary failure on a claim needing more clarifications/tools. In the saved water-damage trace, the agent needs seven model turns, including a clarification; a smaller cap could prevent the `route_to_adjuster` call or final completion. The same test file also checks that `stop_reason` actually governs loop control.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `lookup_policy` and `record_claim_fact` can both involve policy/claim-related identifiers or facts, but their descriptions define distinct jobs: one retrieves policy information by `policy_id`, while the other records a named fact/value into the current claim. The agent should retrieve policy terms with `lookup_policy` and persist claimant evidence with `record_claim_fact`, rather than confusing retrieval with mutation. In `claims_intake/tools.py`, tool failures are structured with `is_error: true` and an `error_category` of `transient` or `permanent`; this lets the agent distinguish a potentially retryable failure from a terminal/input problem and choose a different action. A generic error string would require the model to guess the error class from prose and could trigger unsafe or wasteful retries.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For `claim_03_water_damage` in run `20261015_075030`, the summary reports **7 turns** and **US$0.0316 estimated cost**, with 25,769 input tokens and 1,160 output tokens. The exercise README describes an expected approximate cost of **US$0.05** for the Haiku 4.5 run, so this saved run is about **US$0.0184 lower** than that indicative sample. That difference is plausible because model outputs, tool-call sequence, token usage, and pricing estimates vary between runs; the saved trace also shows one clarification and a specific seven-turn path. The observed run, not the README estimate, is the evidence for my actual result.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `runs/20261016-054807/budget.json` reports **38,708 baseline tokens**, **16,909 assembled tokens**, and a **56.32% reduction**. The `active` section dominates at **15,789 tokens**, compared with `case_facts` at 204, `resolved_refund` at 396, and `resolved_subscription` at 538. The active conversation is kept in detail because it contains ongoing troubleshooting and recent-turn dependencies that may not be safely reconstructed from a summary. The budget file records the token count as coming from Anthropic's model-authoritative `messages.count_tokens` endpoint.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The rule is to compress resolved, older conversation segments into concise summaries, while preserving small structured case facts and exact identifiers/status tokens that downstream questions may require. In `budget.json`, resolved refund and subscription summaries take **396** and **538** tokens, while the case-facts block takes **204**; the active section remains **15,789** tokens because it is still live context. The evaluation supports preserving exact fields: Q6 asks for the literal `in_progress` status token, and the assembled-context evaluation answers it correctly. The principle is lossy compression for narrative history, exact preservation for authoritative structured facts and tokens.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In `eval.jsonl`, Q6 passes: it returns the exact `payment_update_status: in_progress` token. In `eval_control.jsonl`, Q6 fails and answers `unknown`, saying the structured status is absent; the control's Q1 still passes. This demonstrates that the facts block materially protects retrieval of a small but critical structured field that narrative conversation alone may omit or blur. The result is not proof that every fact will always be retrieved, but it is direct evidence that removing/presenting less structured fact context regressed Q6.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The frontmatter in `.claude/rules/api.md` is:
```yaml
---
description: Conventions for Node.js API handlers
paths:
  - "src/api/**/*"
---
```
This rule activates specifically for files under `src/api/`, while the React rule targets `src/components/**/*` and `src/pages/**/*`, and the tests rule targets `**/*.test.tsx` and `**/*.test.ts`. That is better than placing all detailed rules in a directory-level `CLAUDE.md`, because cross-cutting conventions can follow file patterns throughout the monorepo without loading every API, React, and test-specific instruction for unrelated edits. The configuration README and the rule files make this separation explicit.


9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `.claude/skills/deploy-check/SKILL.md` contains:

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

The `context: fork` setting isolates the skill's execution context from the parent conversation, while `allowed-tools` restricts it to inspection-oriented operations. This prevents verbose discovery output from unnecessarily polluting the parent context and limits the skill's ability to modify files or perform deployment actions. 

In my application-level implementation, `04-fork-scratchpad/solution/shift_monitor/fork.py`, `fork_for_hypothesis()` copies the baseline `hot_state.json` into a hypothesis-specific directory, so changes to that copy do not directly modify the original state file. The same module's `merge_findings()` function explicitly appends findings from fork scratchpads to the main scratchpad, providing a deliberate reintegration boundary. However, this is file-level isolation, not complete process isolation: reusing a hypothesis ID can overwrite its previous state, and merging does not currently deduplicate entries or resolve conflicts. Without the Claude Code fork, verbose output could consume the parent context; without the read-only allowlist, the skill could have unnecessarily broad permissions. Without application-level state copies and a controlled merge boundary, independent investigations could interfere through shared state or leave their findings without a defined reintegration path.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The project-level example is the repository's `solution/CLAUDE.md` and its `.claude/rules/` files, which are versioned with the project and define shared conventions for API, React, and test files. The user-level example documented in the `04-plan-mode-and-explore-decision-doc/solution/CLAUDE.md` hierarchy is `~/.claude/`, suitable for personal preferences that should not be shared through version control. The source includes validator logic in `ecommerce_team_config/claude_md.py` and tests such as `tests/test_us01_claude_md_hierarchy.py`, but the ZIP does not include captured validator stdout; therefore I can cite the configured scope distinction and its tests, not claim a particular validator transcript.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The indexed query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` in `shift_monitor/warm.py::WarmStore.defects_since`; the schema creates `idx_defects_ts` on `(ts)` and `idx_defects_shift_ts` on `(shift, ts)`. The March fixture has **23 defects**, noted by `tests/test_us01_tiered_state.py`, while `defects_since` returns only rows after the supplied timestamp and caps results with `LIMIT` (the invocation pipeline uses a bounded limit, typically 50). The model receives the filtered new-defect rows and compact hot state, not the full historical table, because SQL performs the time filtering before prompt construction. This reduces context/cost and avoids asking the model to perform database filtering itself.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → shift_monitor/recovery.py::decide()` uses `STALE_RESUME_THRESHOLD_MINUTES = 30`: no steps or a completed manifest means `fresh`; an incomplete run whose last step is at most 30 minutes old means `resume`; an older partial run means `fresh`. The code comment explains that 30 minutes is roughly 1/16 of an eight-hour shift, treating a recent partial as the same working set and an older partial as stale. A fresh run with a summary injected from the manifest can be more reliable than resuming a stale, partially executed context because it reconstructs from captured findings rather than trusting an old conversational state that may be incomplete or inconsistent. The decision is implemented in `03-crash-recovery/solution/shift_monitor/recovery.py` and reused in `04-fork-scratchpad`.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → `04-fork-scratchpad/solution/data/hot_state.json` is **965 bytes** in the supplied ZIP (`wc -c` on the file). It holds recent defect hashes, a current-shift summary, active alerts, and threshold statuses instead of a full defect history. A byte budget matters for a process that runs every shift indefinitely because small state keeps every invocation cheap and predictable, reduces context growth, and makes state easier to inspect and recover. Detailed historical records remain in the warm SQLite store and scratchpad rather than accumulating in the hot state.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `Build a Claims Intake Agent.../exercises/03-dynamic-decomposition/solution/runs/20261015_075030/traces/claim_03_water_damage.jsonl` shows the model choosing to ask a clarification, classify the claim, assess severity, and route it; these are model decisions guided by the prompt and available tools.
    → Harness:`claims_intake/loop.py` and `tests/test_antipatterns.py` implement and test stop-reason-driven control flow, while `claims_intake/tools.py` validates/dispatches tools and returns structured errors. This layer constrains how model decisions execute.
    → Orchestration: `Build a Multi-Shift Quality Monitoring System.../04-fork-scratchpad/solution/shift_monitor/pipeline.py`, `recovery.py`, `data/hot_state.json`, and `data/warm.sqlite` coordinate persisted state, bounded SQL retrieval, and recovery across shifts. This layer controls the longer-lived workflow and memory.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the stop rule in `claims_intake/loop.py`: continue on `tool_use`, return on `end_turn`, and raise on unexpected stop reasons; `tests/test_antipatterns.py::test_stop_reason_is_loop_control` checks that the code actually uses the signal. A prompt-guided behavior is the decision to ask a focused clarification when `ambiguity_between` indicates plausible claim types, as seen at turn 3 of `claim_03_water_damage` in run `20261015_075030`. Deterministic enforcement is right for safety, termination, permissions, and data integrity; prompts are appropriate for semantic judgment where the model must interpret facts and decide which question or classification best fits.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    →  Intra-session context management in System 2 reduced the retail context from **38,708 to 16,909 tokens (56.32%)** in `runs/20261016-054807/budget.json`, while preserving the exact status token needed for Q6 in `eval.jsonl`. Cross-session context management in System 4 keeps `hot_state.json` to **965 bytes** and fetches bounded recent rows with `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` rather than loading the full SQLite history. The shared principle is to retain the smallest sufficient representation for the next decision, but System 2 compresses a conversation within one request whereas System 4 persists compact state and retrieves relevant records across repeated shifts.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `tests/test_antipatterns.py::test_no_integer_literal_iteration_cap_in_loop` protects against a fixed turn cap that might not fail on a short happy-path trace but could truncate a longer case. Likewise, `tests/test_us03_crash_recovery.py` tests recovery decisions across stale/recent/incomplete state conditions rather than relying on one successful shift run. This matters before shipping because edge cases such as delayed restarts or unusually ambiguous claims are unlikely to appear in every demonstration, yet can silently lose work or prevent a workflow from completing.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → I would choose System 4: if orchestration misbehaves, it could omit new defects, misstate a threshold, lose an alert, or carry incorrect state into subsequent shifts; in a manufacturing quality monitor that could delay quarantine or inspection. Enforcement points include SQL-side time filtering in `WarmStore.defects_since`, the pipeline's structured state update, the recovery manifest, and the persisted `hot_state.json` plus SQLite history. A practical kill switch is to disable/suspend scheduled invocation and preserve the last known-good state/manifest for human review; the model should not be allowed to overwrite the authoritative database or take physical action directly. The supplied `hot_state.json` already illustrates high-impact alerts for capacitor bank C-7 and suspected lot `2026-0430-B`, making state validation and rollback important.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The ZIP contains saved run outputs and test definitions, but not a shell transcript documenting a first-attempt failure, so I cannot honestly claim a specific environment failure or fix from the evidence provided. The clearest observed regression is in the context evaluation: `eval_control.jsonl` fails Q6 by answering `unknown`, whereas `eval.jsonl` passes with `in_progress`. I would describe that as an experimentally observed control failure rather than a setup failure, and say the structured case-facts block corrected the retrieval gap in the assembled-context run. If there was a genuine first-try installation or test failure, add its terminal output and the fix here before submission.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would add a stronger, automated end-to-end acceptance test that runs the full context assembly and verifies both token budget and all exact-field retrieval questions, rather than relying on the saved evaluation artifacts alone. The evidence is the Q6 regression between `eval.jsonl` and `eval_control.jsonl`, despite Q1 passing in both, which shows that aggregate success or a single passing answer can hide a targeted retrieval failure. I would also add a reproducible cost/latency report alongside `budget.json` so the context reduction can be evaluated against quality and spend in one place. This would make the trade-off measurable over repeated runs rather than inferred from one run and one control.
