# Reflection Brief — Harness Engineering Capstone

**Name:** Saksham Verma
**Date:** Nov 23 ,2026

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s):
- OS / Python:
- Approx. API spend:

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → The agentic loop is controlled exclusively through response.stop_reason. The assignment contract states that the loop continues only when stop_reason == "tool_use" and returns only when stop_reason == "end_turn". In claims_intake/loop.py, the decision is implemented inside the run() function through two explicit branches:
   The agentic loop is controlled exclusively through response.stop_reason. The assignment contract states that the loop continues only when stop_reason == "tool_use" and returns only when stop_reason == "end_turn". In claims_intake/loop.py, the decision is implemented inside the run() function through two explicit branches:
   Any other value triggers an UnexpectedStopReason exception. This design ensures that loop control is based on structured API output rather than natural-language content. Artifact cited: claims_intake/loop.py::run() and the System 1 contract specification.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One anti-pattern checked by test_antipatterns.py is using string-membership tests against assistant text to drive control flow. The audit flags expressions such as:
   "tool" in assistant_response
because they make execution depend on model wording instead of structured signals. The test test_no_string_membership_against_text_in_loop() exists specifically to prevent natural-language-driven control flow. If my implementation relied on string matching, even a minor wording change from the model could incorrectly continue or terminate execution. By enforcing use of stop_reason, the loop remains deterministic and robust across model responses. Artifact cited: test_antipatterns.py, Anti-pattern 1 specification.
3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → Tool routing uses explicit tool names and structured inputs instead of parsing assistant text. This prevents ambiguity even when tools accept similar JSON inputs. The ToolExecutor contract also returns structured errors (is_error=true) instead of raising exceptions, allowing the model to inspect failures and recover in later turns. Artifact cited: claims_intake/loop.py (ToolExecutor definition).

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → In run 20261123_052826, claim_08_minor_porch_damage took 5 turns and cost $0.0226, while claim_03_water_damage took 2 turns and cost $0.0066. Across all 8 fixtures, total estimated spend was $0.1258. The differences came from varying numbers of turns, tool calls, and token usage. Artifact cited: Run summary — 20261123_052826.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → From budget.json, the baseline context was 38,708 tokens and the assembled context was 16,830 tokens, a 56.52% reduction. The active section dominates the assembled context at 15,789 tokens. It is kept verbatim because it contains the current conversation state and unresolved issues the model must reason about accurately.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The strategy is to summarize resolved history and preserve active context. For example, resolved_refund is compressed to 367 tokens and resolved_subscription to 488 tokens, while the active section remains 15,789 tokens. Resolved cases only need key facts retained, whereas active work requires exact details and state.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → In eval.jsonl, the model correctly returned the exact status token in_progress and passed. In eval_control.jsonl, it answered with a paraphrase ("ongoing"/unresolved) and failed. This proves the facts block preserves critical structured values that summarization alone may lose.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → One rule file uses the following glob frontmatter:
   ---
description: Conventions for test files (co-located *.test.ts and *.test.tsx)
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
---Using path-scoped rules is better than a directory-level CLAUDE.md because the convention applies to test files anywhere in the repository, not just within a specific folder. This makes cross-cutting rules easier to maintain and automatically activates them wherever matching files exist. Artifact cited: .claude/rules/tests.md.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The deploy-check skill declares:context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
(along with several read-only Git and GitHub commands).

Running in a forked, read-only context keeps verbose outputs such as git status, diffs, and PR checks out of the main conversation while preventing the skill from modifying files or deploying changes. Without context: fork, the main session would be polluted with temporary analysis output, and without the read-only allowlist, the skill could accidentally make repository changes. Artifact cited: .claude/skills/deploy-check/SKILL.md.
10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The configuration separates shared team conventions from personal preferences. A project-level example is ./CLAUDE.md and .claude/standards/testing.md, which are committed to Git and shared with the team. A user-level example is ~/.claude/CLAUDE.md or ~/.claude/skills/, which stay on an individual's machine and are not version-controlled. This allows teams to enforce common standards while preserving personal workflows. Artifact cited: CLAUDE.md scope table.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?


    → The indexed query is defects_since(), which runs SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?. The idx_defects_ts index allows SQLite to filter and retrieve only the relevant defects before they reach the model. As a result, the model never sees the entire history stored in warm.sqlite; it only receives the limited set of records returned by the query. Artifact cited: shift_monitor/warm.py.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?


    → The recovery logic resumes a session only if the manifest is incomplete and the last recorded step is 30 minutes old or less. If the workflow is complete, empty, or older than 30 minutes (STALE_RESUME_THRESHOLD_MINUTES = 30), it starts fresh instead. A fresh start with an injected summary is often more reliable because it carries forward validated findings while avoiding partially completed or stale execution state that may no longer match the current data. Artifacts cited: test_recovery_decide_truth_table, test_threshold_constant_is_30_minutes, STALE_RESUME_THRESHOLD_MINUTES = 30.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?


    → My hot_state.json is only 643 bytes in size. It stores a compact summary of the current shift, active alerts, and threshold statuses instead of retaining full historical records. Keeping state small matters because the system runs repeatedly every shift and may operate indefinitely; without a strict state budget, storage, recovery time, and context size would continuously grow. Artifacts cited: data/hot_state.json and ls -lh data/hot_state.json (643 bytes).

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.


→ Model: The model performs reasoning, classification, and tool selection. Evidence: run 20261123_052826, where the model routed claims and decided when to call tools.

→ Harness: The harness controls execution through response.stop_reason and handles tool execution. Evidence: claims_intake/loop.py::run().

→ Orchestration: The orchestration layer manages persistence, retrieval, recovery, and state across runs. Evidence: shift_monitor/warm.py, recovery.py, and data/hot_state.json.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic behavior is the loop control in claims_intake/loop.py, where execution continues only on stop_reason == "tool_use" and stops on end_turn. A prompt-guided behavior is deciding how to classify or route a claim. Deterministic rules are best for safety and control flow, while prompts are appropriate when judgment and interpretation are required.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 reduced context from 38,708 to 16,830 tokens (56.52% reduction) by summarizing resolved information and preserving active context. System 4 applies the same idea across runs by storing only 643 bytes in hot_state.json instead of retaining complete defect history. Both systems minimize context, but System 2 handles active session context while System 4 manages long-term state.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → The anti-pattern tests guarantee that loop control never depends on string matching or hard-coded iteration limits. A single successful run would not reveal these problems because they often appear only under different model outputs or future code changes. The static audits in test_antipatterns.py provide confidence that the architecture remains correct before deployment.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → In System 1, a faulty loop could repeatedly call tools, increase token costs, or never finish processing a claim. The blast radius is limited by the Budget mechanism, which enforces token and time limits and raises BudgetExceeded instead of allowing unbounded execution. The enforcement points are the budget checks inside claims_intake/loop.py.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The first challenge was understanding how hidden Claude configuration files under .claude/ interacted with the project-level CLAUDE.md. I initially could not locate the rule and skill definitions because hidden directories were not visible in the file explorer. I resolved it by listing the files with find .claude -type f and then inspecting the relevant rule and skill files directly.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would make the orchestration system expose more recovery and query metrics directly through status commands. While the architecture correctly keeps state small (hot_state.json is only 643 bytes), understanding recovery decisions and warm-store behavior required inspecting implementation files and tests. Exposing those operational details through built-in diagnostics would make troubleshooting and maintenance easier without changing the underlying design.
