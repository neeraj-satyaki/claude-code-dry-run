---
name: dry-run
description: Dry-run code like a step debugger, in any language. Traces execution line by line from an entry point with concrete inputs, showing the call stack, variable values, branches taken, loop iterations, return values and side effects, and flags the exact step where behaviour goes wrong. Use when the user says "dry run", "dry the code", "trace this", "step through", "walk me through execution", "what happens when I call X with Y", "debug this by hand", or /dry-run.
---

# /dry-run — universal step debugger

Act as a debugger attached to the code. Execute it in your head, step by step, with real values. Works for any language: Python, JS/TS, Java, C/C++, Go, Rust, C#, PHP, Ruby, Kotlin, Swift, SQL, shell, etc.

## Usage

```
/dry-run <file|function|snippet>                 # pick entry + inputs, ask if unclear
/dry-run <function> with <args>                  # trace one call
/dry-run <file> --break <line|function>          # fast-forward, then step from breakpoint
/dry-run <file> --watch <var1,var2>              # only print watched vars each step
/dry-run <file> --bug "<symptom>"                # hunt: find the first step that diverges
/dry-run <file> --run                            # also execute for real and compare
```

## Step 1 — Attach

1. Read the entry file and every function the path will reach. Follow imports into the repo. Stop at stdlib/third-party: treat them as known behaviour and say what they return.
2. Fix the **entry point** (main, handler, test, function named by the user).
3. Fix **concrete inputs**: args, env vars, config, file/DB/network contents. If the user gave none, pick a realistic one and state it. If the choice changes the path a lot, ask once.
4. Note language semantics that matter here (see "Language traps").

## Step 2 — Step through

Print a trace table. One row per executed statement that changes state or control flow. Skip pure no-ops.

```
#  | loc            | code                      | state after                 | note
---|----------------|---------------------------|-----------------------------|------------------
1  | app.py:12      | total = 0                 | total=0                     |
2  | app.py:13      | for x in items:           | x=3 (iter 1/3)              |
3  | app.py:14      | if x > limit:             | 3 > 5 → False               | branch skipped
4  | app.py:16      | total += x                | total=3                     |
```

Rules:
- **Call stack**: on each call, show `→ enter fn(args)` with a frame depth indent; on return, `← fn returns <value>`.
- **Branches**: show the condition with values substituted and the result.
- **Loops**: show every iteration up to 5. For longer loops show iterations 1–3, `…`, and the last one, plus the invariant that holds between.
- **Recursion**: show the frame tree; collapse identical sub-calls once shown.
- **Mutation and aliasing**: when two names point at one object, say so and update both.
- **Side effects**: mark I/O, DB writes, network calls, globals, file writes with `[SIDE EFFECT]`.
- **Exceptions**: show where raised, each frame it unwinds, and where caught (or that it crashes).
- **Async/concurrency**: show what yields at each `await`/goroutine/thread start, and the order you assume. Name other orders that are possible.
- **Unknowns**: if a value cannot be known statically (random, time, external API), say `UNKNOWN: <why>` and branch on the plausible values that change the path.

## Step 3 — Breakpoints and watch

- `--break`: summarise everything before the breakpoint in 1–3 lines (state at arrival), then step from there.
- `--watch`: trace table shows only watched variables, but every change to them.
- At any point the user may say `step`, `next` (step over call), `into`, `out`, `continue`, `print <expr>`, `set <var>=<value>`. Obey them like a debugger and keep the current state.

## Step 4 — Bug hunt (`--bug` or when something looks wrong)

1. State the **expected** value at each key step, next to the actual.
2. Stop at the **first** step where actual ≠ expected. That is the defect site, not where the symptom shows.
3. Report: `loc`, what the code does, what it should do, why (root cause), minimal fix.
4. Re-run the trace with the fix applied on the same input to show it now matches.
5. Check edge inputs that commonly break this kind of code: empty, zero, one, negative, max size, null/None/undefined, duplicate, unicode, off-by-one bounds.

## Step 5 — Verify against reality (`--run`, or when cheap and safe)

If the code can run locally without side effects on real systems, run it (add temporary prints or use the language debugger: `python -m pdb`, `node inspect`, `gdb`/`lldb`, `dlv`, `jdb`) and compare to the dry run. Report any mismatch: it means the mental model was wrong. Remove temporary prints afterwards.

Never run code that writes to production DBs, sends network requests to real services, deletes files, or spends money. Dry-run those parts only.

## Output format

```
Entry:  <fn/file>   Inputs: <values>   Assumptions: <list>

<trace table>

Result:  returns/prints/raises <value>
Side effects: <list or none>
Findings:
- <loc>: <bug or risk> — <fix>
```

Keep it tight. Large traces: collapse repeated identical work and say how many steps were collapsed.

## Language traps (check the ones that apply)

- **Python**: mutable default args, late-binding closures in loops, `is` vs `==`, integer division `//`, shallow copy, generator exhaustion, truthiness of `0`/`""`/`[]`.
- **JS/TS**: `==` coercion, `this` binding, `var` hoisting, closures in loops, floating point, `NaN`, async ordering (microtask vs macrotask), `sort()` sorts as strings by default.
- **C/C++**: undefined behaviour, integer overflow, uninitialised vars, pointer aliasing, off-by-one on arrays, null terminator, signed/unsigned compare, order of evaluation.
- **Java/C#/Kotlin**: `==` on objects vs `.equals`, integer caching, null unboxing, pass-by-value of references, `ConcurrentModificationException`.
- **Go**: loop var capture (pre-1.22), nil map write panics, slice sharing backing array on `append`, goroutine ordering, nil interface vs typed nil.
- **Rust**: ownership moves, borrow scope, integer overflow panics in debug only, `unwrap` panics.
- **SQL**: NULL comparison, join row multiplication, GROUP BY semantics, transaction isolation.
- **Shell**: word splitting, unquoted vars, exit codes in pipes, `set -e` gaps, subshell variable loss.
