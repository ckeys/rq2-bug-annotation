# RQ2 Bug Annotation Guideline (v2)

## Task Overview

You are annotating **failed turn instances** from multi-turn LLM programming conversations. In each instance, an LLM was asked to modify code according to a requirement, and the resulting code **failed** some or all regression tests (tests from prior turns that previously passed).

Your goal: identify **why** the code failed by classifying the bug type(s).

## What You Will See

For each sample, you will see:

1. **Original Task** — the seed task description (e.g., "implement function `has_close_elements`")
2. **Turn Number & Change Type** — which turn this failure occurred at and what kind of change was requested
3. **Current Turn Prompt** — the exact instruction given to the LLM
4. **Previous Code** — the code from the prior turn (which passed tests). For T1 samples, this field is empty.
5. **Current Code** — the code the LLM produced this turn (which failed tests)
6. **Failed Tests** — specific test cases that failed, with error messages
7. **Regression Rate** — fraction of original tests that still pass (0.0 = total failure)

## Bug Taxonomy

### Category 0: Baseline Failure (T1 only)

| ID | Name | Definition | Example |
|----|------|-----------|---------|
| BF | Baseline Failure | The LLM fails to correctly implement the original task specification at Turn 1. This is a **single-turn** implementation error, not a multi-turn regression. | T1 implements `is_palindrome` but returns wrong result for edge case `""`. |

> **Important**: BF applies **only** to Turn 1 failures. If a sample is from T1 and the code does not pass the original tests, label it BF. Do **not** use any other label for T1 samples.

### Class I: Context-Loss — LLM loses prior context

| ID | Name | Definition | Example |
|----|------|-----------|---------|
| CL-1 | Context Loss | The LLM forgets or overwrites a constraint / behavior established in a prior turn. The previously-correct logic either disappears entirely or is replaced by new logic that breaks it. | T2 added `TypeError` for invalid types; T4's code no longer raises `TypeError`. T3 extends input handling but replaces T2's validation logic entirely. |
| CL-3 | Partial Context Recall | The LLM remembers some but not all critical details from prior code. Functions/methods exist but have incomplete internal logic. | T6 refactors to class; `clear_cache()` becomes a method but its body is empty. T4 adds caching but forgets that T3 added string-to-number conversion, so cached keys don't account for string inputs. |

> **CL-1 sub-labels** (optional, for finer analysis):
> - *Passive forgetting*: the constraint simply disappears from the output with no replacement.
> - *Active overwrite*: new turn's logic explicitly replaces / contradicts the prior constraint.

### Class II: Cross-Turn Conflict — new and old code conflict

| ID | Name | Definition | Example |
|----|------|-----------|---------|
| CC-1 | Semantic Collision | New logic contradicts or is incompatible with prior logic, producing incorrect results or runtime errors. | T4's caching returns stale results that conflict with T3's type conversion. T8 replaces the sorting algorithm but the new implementation returns results in wrong order. |
| CC-2 | Interface Mismatch | New code changes a function/method signature, return type, or calling convention that prior code or tests depend on. | T6 changes method signature from `func(a, b)` to `func(params_dict)`, breaking test assertions. |
| CC-3 | Integration Inconsistency | New code and existing code are individually correct but fail to integrate properly — their data flow, shared state, or assumptions are inconsistent. | T7 adds `self.stats` tracking but the caching code path bypasses the stats counter. T4 adds caching using unhashable keys, conflicting with T3's list-type inputs. |
| CC-4 | Over-Guarding | The LLM adds defensive code (validation, type checks, boundary guards) that **rejects inputs the original specification treats as valid**. The guard is overly restrictive relative to the task's intended behavior. | T2 adds `if not s: raise ValueError` for an empty string input, but the original tests expect `func('') == []` (empty string is a valid input). T2 adds `if n <= 0: raise ValueError` but original tests include `func(0) == []`. |

### Class III: Accumulation — progressive degradation

| ID | Name | Definition | Example |
|----|------|-----------|---------|
| AC-1 | Regression via Omission | The LLM silently drops an **entire function, method, or substantial code block** that was not mentioned in the current prompt. The missing code was present in the previous turn's output. | T8 rewrites the algorithm but omits the `_batch` function entirely. T7 adds logging but the output no longer includes `clear_cache()`. |
| AC-2 | Complexity Collapse | The accumulated code exceeds the LLM's generation capacity, producing syntactically invalid, truncated, or empty output. | Code is cut off mid-function; missing closing brackets; garbled output; empty response. |
| AC-3 | Style Drift | Inconsistent coding patterns across turns lead to subtle bugs (naming, conventions, data structure choices). | Variable renamed from `result` to `res` between turns causes `NameError`. Switching between `snake_case` and `camelCase` causes attribute lookup failures. |

### Distinguishing Rules

| Confusion pair | Rule |
|----------------|------|
| **CL-1 vs CL-3** | CL-1 = a constraint/behavior is **gone** (the relevant code is absent or replaced). CL-3 = the code **exists** but is **incomplete** (partial implementation, missing edge cases within an otherwise present function). |
| **AC-1 vs CL-3** | AC-1 = an **entire function/method/code block** is silently removed. CL-3 = function exists but internal logic has gaps or omissions. |
| **AC-1 vs CL-1** | AC-1 = structural removal (whole function/block gone). CL-1 = behavioral removal (a specific constraint within a function is lost). |
| **CC-1 vs CC-4** | CC-1 = new logic produces **wrong results** for valid inputs. CC-4 = new logic **rejects** valid inputs by raising an exception before any computation occurs. |
| **CC-3 vs CC-1** | CC-3 = both old and new code are individually correct, but they fail to work together (integration issue). CC-1 = new code's logic directly contradicts old code's semantics. |
| **CL-1 vs CC-4** | CL-1 = a prior constraint is lost (something that was being enforced is no longer enforced). CC-4 = a **new** guard is added that is too strict (something that was NOT being guarded is now incorrectly guarded). |

## Annotation Instructions

### Step 1: Check if Turn 1

If the sample is from **Turn 1**, check whether the code simply fails to implement the original task correctly. If so, label it **BF (Baseline Failure)** and skip to submission.

### Step 2: Compare Previous Code vs Current Code

Read the **previous code** (which passed) and the **current code** (which failed). Identify what changed.

### Step 3: Examine Failed Tests

Look at the failed test cases and their error messages. What specific behavior broke?

- `ValueError` / `TypeError` from new validation → likely **CC-4** (Over-Guarding)
- `AssertionError` (wrong return value) → likely **CL-1** or **CC-1**
- `NameError` / `AttributeError` (missing symbol) → likely **AC-1**, **CL-3**, or **AC-3**
- `SyntaxError` / truncated code → likely **AC-2**
- `Timeout` / infinite loop → **diagnose the root cause**:
  - New algorithm has a logic error (e.g., missing termination condition) → **CC-1**
  - A prior optimization (e.g., caching) was silently removed, causing redundant computation → **AC-1**
  - A loop bound or exit condition from a prior turn was forgotten → **CL-1**

### Step 4: Diagnose Root Cause

Ask yourself: **Why did the previously-passing tests fail?**

- Did the LLM forget or lose something from before? → **Class I (Context-Loss)**
- Did new code conflict with old code? → **Class II (Cross-Turn Conflict)**
- Did the code degrade due to growing complexity? → **Class III (Accumulation)**

### Step 5: Select Bug Type(s)

- Select **all applicable** bug types (multi-label).
- Then select the **primary type** — the single most important root cause.

### Rules

1. **Multi-label**: A single failure can exhibit multiple bug types. Select all that apply.
2. **Primary type**: Always select exactly one primary type — the root cause.
3. **When unsure**: If the failure could be two types, select both and choose the more specific one as primary.
4. **AC-2 (Complexity Collapse)**: Only use this when the code is syntactically broken, truncated, garbled, or empty. Logic errors are NOT AC-2.
5. **CC-4 (Over-Guarding)**: Use this when the LLM adds a guard/check that raises an exception for an input that the original tests treat as **valid**. The key signal is: (a) the error is `ValueError` or `TypeError`, (b) the error comes from NEW validation code added by the LLM, and (c) the test expects a normal return value, not an exception.
6. **BF (Baseline Failure)**: Only for T1 samples. Never use BF for T2-T8 samples.

## Pilot Round

This pilot round contains 20 samples. After annotation:
- We will compare all 4 annotators' labels.
- Disagreements will be discussed to calibrate understanding.
- The guideline may be refined based on pilot results.
