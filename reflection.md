
# Reflection Brief — Harness Engineering Capstone

**Name:** Srijith chenna
**Date:** 2026-09-28

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.**
   The loop control is implemented in `claims_intake/loop.py`, where `response.stop_reason` determines whether execution continues or stops. In `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl`, the recorded sequence is `tool_use → tool_use → tool_use → tool_use → tool_use → end_turn` across 6 turns. The first five turns invoke tools for policy lookup, fact recording, classification, severity assessment, and routing; the sixth turn ends the interaction. This demonstrates that termination is driven by the API response state rather than a fixed iteration count.

2. **Anti-pattern.**
   One anti-pattern checked by `tests/test_antipatterns.py` is the use of a fixed iteration count rather than the API's `stop_reason` to control the loop. If the implementation relied on a fixed number of turns, it could stop before the model had completed the required terminal routing/escalation action or continue after the model had already reached `end_turn`. The recorded System 1 traces demonstrate why termination needs to be tied to the actual response state.

3. **Tool design.**
   The terminal routing tools have overlapping claim information because both operate on the same claim context, but their structured schemas and descriptions distinguish the actions the model is allowed to take. The routing path records a structured decision, while escalation records the need for human review. Structured tool errors preserve machine-readable information about what failed, allowing the agent to correct its next action instead of treating the failure as an undifferentiated string.

4. **Your numbers.**
   The `claim_01_kitchen_fire` trace completed in **6 turns**: five `tool_use` turns followed by `end_turn`. The trace records **22,638 input tokens** and **982 output tokens**, for **23,620 total tokens**. The claim was routed through `route_to_adjuster` on turn 5, followed by the final `end_turn` on turn 6. The trace does not record a dollar cost, so I report the measured token usage rather than estimating a monetary cost.

### System 2 — Context strategy

5. **The reduction.**
   `budget.json` records a baseline of **38,708 tokens**, an assembled context of **16,776 tokens**, and a **56.66% reduction**. The active section dominates the assembled context at **15,789 tokens**, compared with 204 for case facts, 363 for the resolved refund history, and 438 for the resolved subscription history. The active issue is kept verbatim because it contains the current operational details the copilot needs to answer accurately.

6. **Summarize vs preserve.**
   The context strategy summarizes resolved history while preserving the active issue. The budget shows 363 tokens for the resolved refund section and 438 for the resolved subscription section, while the active section remains 15,789 tokens. This keeps historical information compact while preserving the current issue in detail.

7. **Facts block.**
   The assembled `eval.jsonl` passed all six questions. In `eval_control.jsonl`, the stripped control specifically regressed on **Q6**, which asks for the exact structured status token `in_progress`; the control returned an unknown answer instead. This demonstrates that the compact case-facts block preserves structured information that cannot reliably be reconstructed from the remaining conversational context.

### System 3 — Claude Code config

8. **Path-scoped rules.**
   `.claude/rules/react.md` uses path frontmatter such as `paths: - "src/components/**/*" - "src/pages/**/*"`. This is more targeted than placing every convention in a directory-level `CLAUDE.md` because the rule activates based on the file being edited rather than requiring the entire directory hierarchy to carry the same convention. The repository also has separate scoped rules for API files and test files.

9. **Forked skill.**
   `.claude/skills/deploy-check/SKILL.md` specifies `context: fork` and allows only `Read`, `Grep`, `Glob`, and narrowly scoped read-only Git commands such as `git status`, `git diff`, and `git log`. Running the check in a fork isolates the deployment validation from the main session, while the allowlist prevents the check from modifying application state. Without those restrictions, a validation skill could have a larger blast radius because it could operate with the main session's context or broader tools.

10. **Scope.**
    The System 3 validator output is `OK` with `EXIT_CODE=0`, and the project's `CLAUDE.md` documents both project-level and user-level scope. Project-level configuration includes `./CLAUDE.md`, `.claude/standards/`, and `.claude/rules/`; these are version-controlled and shared with the team. User-level configuration lives under `~/.claude/`, such as `~/.claude/CLAUDE.md`, `~/.claude/commands/`, or `~/.claude/skills/`, and is explicitly not shared through version control. A concrete user-level example given by the configuration is a personal `/morning` summary command or a preferred commit-message style.

### System 4 — Orchestration

11. **Push work down.**
    The warm database contains **40** fixture defects: 14 for shift A, 13 for B, and 13 for C. The recorded shift run returned **0 new defects**, because `gather_new_defects()` delegates to `WarmStore.defects_since()` and the SQL performs the time filtering before the data reaches the model. The indexed query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, with `idx_defects_ts` and `idx_defects_shift_ts` created in the schema. The model therefore receives only the SQL-filtered slice rather than the entire 40-row history.

12. **Crash recovery.**
    `shift_monitor/recovery.py` defines `STALE_RESUME_THRESHOLD_MINUTES = 30` and uses `decide()` to choose `resume` only when there is incomplete state and the latest step is within that 30-minute window. Missing or complete state, and incomplete state older than 30 minutes, result in `fresh`. Starting fresh can be more reliable than resuming stale work because the system avoids continuing from an old partial state while still allowing a new invocation to use the available summary and current defect slice.

13. **Small state.**
    The recorded `data/hot_state.json` is only **643 bytes**, well below the system's approximately 5 KB hot-state budget. This matters for a system that runs once per shift indefinitely because an ever-growing state file would eventually consume more prompt context on every invocation. The architecture therefore keeps compact active state hot while longer-lived defect information remains in the warm/cold tiers.

---

## Part 2 — Synthesis

14. **Three layers.**
    **Model:** `claims_intake/loop.py` demonstrates the model-driven agentic loop, where the API response's `stop_reason` determines the next action.
    **Harness:** the System 3 `.claude/` configuration provides deterministic project rules, path-scoped instructions, commands, and the forked `deploy-check` skill.
    **Orchestration:** System 4's `shift_monitor/pipeline.py`, `warm.py`, `recovery.py`, and `fork.py` coordinate persistent state, SQL filtering, recovery, and isolated hypothesis work around model calls.

15. **Deterministic vs prompt.**
    Deterministic enforcement is appropriate when a safety or correctness boundary must not depend on model compliance: System 4's SQL-side filtering and 643-byte hot state are enforced by code, while System 1's loop termination is explicitly controlled by `stop_reason`. Prompt guidance is useful for behavior that requires judgment, such as how the agent should interpret evidence and formulate a shift report. The architectural distinction is that code should enforce hard boundaries while prompts guide model behavior inside those boundaries.

16. **Context, two faces.**
    System 2 manages context within a session by reducing a **38,708-token baseline to 16,776 assembled tokens**, a **56.66% reduction**, while preserving the active issue. System 4 manages context across shifts by keeping `hot_state.json` at **643 bytes** and storing the larger defect history in SQLite, where SQL filters the data before model invocation. Both systems apply the same principle—keep only the context needed for the current operation—but System 2 uses summarization and section selection while System 4 uses persistent tiers and database queries.

17. **Reliability you can't see in one run.**
    System 4's recovery tests guarantee behaviors that a single successful shift run would not expose, including the resume/fresh truth table around the 30-minute threshold. The suite also verifies that two forks have independent scratchpads and that merging appends findings without rewriting existing entries. These properties matter because failures such as stale recovery or cross-fork contamination may not occur during an ordinary successful run but can affect later shifts.

18. **Blast radius.**
    In System 4, a failure in state management could affect subsequent shift analyses because hot state is carried from one invocation to another. The architecture limits this through the small hot-state budget, the 30-minute recovery threshold, SQL-side data selection, and isolated hypothesis forks implemented in `fork.py`. The fork mechanism copies the base hot state into a hypothesis-specific directory, and scratchpad findings are kept separate until an explicit merge, limiting cross-hypothesis contamination.

---

## Part 3 — Honest assessment

19. **What broke.**
    The first attempts exposed environment and evidence issues rather than a single clean end-to-end path: System 2 initially produced a 30-test checkout while the reviewer required the original 17-test suite, and System 3's validator evidence file was initially empty. I corrected the System 2 evidence by running the four repository test files that correspond to the original 17-test suite, producing `17 passed`. I also reran the System 3 validator and captured `OK` with `EXIT_CODE=0`. The System 4 repository itself reports `33 passed`, which I have preserved rather than deleting tests to manufacture the reviewer's requested count of 28.

20. **What you'd change.**
    I would make the evidence-generation process part of the architecture rather than treating evidence capture as a final submission step. The System 2 test-count discrepancy and System 4's 33-versus-28 expectation show that the implementation, repository documentation, and submission rubric can describe different verification targets. A dedicated evidence command could record the exact test selection, artifact paths, run identifiers, and configuration version at the time each system is verified, reducing ambiguity during review.


### System 4 test-count clarification

The official project instructions specify the four system test-suite counts as **29 + 30 + 35 + 33**, and the System 4 solution README states **33 passed** for the complete suite. My System 4 run therefore reports **33 passed**, rather than reducing the suite to the rubric's 28-test expectation. The 33 tests come from `test_us01_tiered_state.py` (9), `test_us02_invocation_pipeline.py` (6), `test_us03_crash_recovery.py` (14), and `test_us04_fork_scratchpad.py` (4). I retained the complete reference test suite because the additional recovery and fork tests exercise distinct behaviors such as crash-recovery boundary conditions, incomplete/complete state handling, fork isolation, and scratchpad merging; deleting valid tests solely to produce a count of 28 would not match the project's stated 33-test verification target.
