# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

BugHound runs a simple agentic loop:

1. Plan: the agent sets up a quick scan-and-fix workflow.
2. Analyze: it looks for likely problems in the code. In heuristic mode, it uses built-in rules to detect patterns such as bare except blocks, print statements, and TODO comments. In Gemini mode, it sends the code to a Gemini model with a strict JSON-output prompt.
3. Act: it proposes a rewrite of the code. If the analyzer found no issues, the agent leaves the code unchanged. If the model path is enabled but produces unusable output, the agent falls back to the heuristic fixer.
4. Test: it runs a lightweight reliability assessment over the proposed change.
5. Reflect: it decides whether the fix should be auto-applied or deferred for human review.

The system uses heuristics by default when no LLM client is available or when the model output is malformed. Gemini is treated as an optional tool that can supplement or override the heuristic analyzer/fixer path.

---

## 3) Inputs and outputs

**Inputs:**

- Tested sample snippets from the sample_code folder, including cleanish.py, mixed_issues.py, print_spam.py, and flaky_try_except.py.
- The inputs were small Python functions and scripts, including print-based examples, TODO comments, and bare except blocks.

**Outputs:**

- Detected issues included print-statement warnings, bare except handling, and TODO comments.
- Proposed fixes ranged from replacing print() with logging.info() to changing bare except blocks to except Exception as e: and adding a comment or logging placeholder.
- Risk reports varied from low-risk/no-action cases to high-risk cases where the system explicitly refused to auto-fix.

---

## 4) Reliability and safety rules

Two important rules in assess_risk are:

1. Return-statement change check
   - What it checks: whether the fixed code still contains return statements when the original did.
   - Why it matters: removing returns can silently change control flow and function behavior.
   - False positive: a harmless refactor could be flagged even if it preserved behavior.
   - False negative: a bug could still slip through if the function behavior changed in a less obvious way while still returning a value.

2. Bare-except modification check
   - What it checks: whether code that used a bare except block was changed.
   - Why it matters: exception handling changes are safety-sensitive because they affect error semantics and control flow.
   - False positive: a simple, behavior-preserving exception rewrite could be over-penalized even when it is still correct.
   - False negative: a risky change to exception handling might still pass if it does not visibly alter the except syntax but changes behavior in other ways.

A third rule added during this activity is the no-substantive-change guardrail:
   - What it checks: whether the fixed code is identical to the original code.
   - Why it matters: a no-op edit should not be treated as a safe auto-fix.
   - False positive: a minimal but still useful comment or formatting change could be unnecessarily blocked.
   - False negative: the system may still present an unchanged output as a valid suggestion if the change is semantically important but syntactically identical.

---

## 5) Observed failure modes

1. Model output format mismatch
   - Example: the analyzer sometimes returned a payload that was not parseable JSON, or returned a single object instead of the expected array.
   - What went wrong: the agent had to fall back to heuristics instead of trusting the model result, which reduced the benefit of using Gemini for analysis.

2. Risky or unnecessary fix suggestion
   - Example: in the mixed_issues sample, the agent proposed a rewrite that touched exception handling and changed the code more aggressively than expected.
   - What went wrong: the fix felt more invasive than the stated issues warranted, and the risk layer correctly rejected it for auto-fix.

3. No-op overconfidence
   - Example: a clean sample such as cleanish.py could otherwise be treated as a safe auto-fix case even when no substantive change was made.
   - What went wrong: the system now guards against this by treating unchanged output as high-risk and blocking auto-fix.

---

## 6) Heuristic vs Gemini comparison

Heuristic mode was consistent and predictable. It reliably detected the obvious patterns in the sample files, especially print statements and bare except blocks. It also worked fully offline and did not depend on API quota.

Gemini mode was more flexible in language and could sometimes produce more nuanced explanations or rewrites, but it was also less deterministic. In the tested runs, the model output was not always in the exact structure the agent expected, which made format handling and fallback behavior important. The risk scorer often agreed with the intuition that exception-handling changes should be treated cautiously, even when the model suggested a rewrite.

In short: heuristics are good for stable, simple patterns; Gemini can add richer interpretation, but its output must be treated as draft material rather than guaranteed machine-readable instructions.

---

## 7) Human-in-the-loop decision

BugHound should refuse to auto-fix when the proposed change touches exception handling or when the change is effectively a no-op. A specific trigger would be: if the code includes exception-handling logic and the rewritten code changes that structure, or if the original and fixed code are identical.

This guardrail belongs in the risk assessment logic, because that layer already decides whether a fix is safe enough for auto-application. The UI should display a message such as: “This change affects error-handling behavior or makes no substantive change. Human review is recommended before applying it.”

---

## 8) Improvement idea

A low-complexity improvement would be to add a small “minimal diff” guardrail that checks whether a proposed fix changes more than the relevant issue. For example, if the agent is only addressing a print-statement issue but the patch also rewrites the function signature, changes return values, or reworks exception behavior, the system should downgrade confidence and require human review. This would make the agent less likely to over-edit code while keeping the overall workflow simple.
