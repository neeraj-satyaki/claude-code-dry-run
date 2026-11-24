# dry-run: a step debugger for every language, inside Claude Code

> Step through code without installing a debugger, setting breakpoints, or even being able to run it.

`/dry-run` turns Claude into a step debugger. Point it at a file, a function or a snippet. It runs the code in its head with concrete inputs and shows each step: the call stack, variable values, branches taken, loop iterations, side effects and exceptions. Then it stops at the **first step where the code goes wrong**.

It works with Python, JavaScript/TypeScript, Java, C/C++, Go, Rust, C#, PHP, Ruby, Kotlin, Swift, SQL and shell. If the language has semantics, it can trace it.

## Demo

```python
def add_tag(tag, tags=[]):
    tags.append(tag)
    return tags

a = add_tag("x")
b = add_tag("y")
print(a, b)
```

```
/dry-run ex.py --bug "b should be ['y']"
```

```
Entry: ex.py   Inputs: none   Assumptions: CPython 3

#  | loc      | code                  | state after                         | note
---|----------|-----------------------|-------------------------------------|---------------------------
1  | ex.py:1  | def add_tag(...)      | default tags → list@L1 = []         | default built ONCE, at def
2  | ex.py:5  | a = add_tag("x")      | → enter add_tag(tag="x", tags=L1)   |
3  | ex.py:2  |   tags.append(tag)    |   L1 = ["x"]                        |
4  | ex.py:3  |   return tags         | ← returns L1;  a = L1               |
5  | ex.py:6  | b = add_tag("y")      | → enter add_tag(tag="y", tags=L1)   | expected fresh [], got L1  ✗
6  | ex.py:2  |   tags.append(tag)    |   L1 = ["x","y"]   (a aliases L1!)  |
7  | ex.py:3  |   return tags         | ← returns L1;  b = L1               |
8  | ex.py:7  | print(a, b)           | ['x', 'y'] ['x', 'y']               | [SIDE EFFECT] stdout

Result: prints ['x', 'y'] ['x', 'y']
Findings:
- ex.py:1: mutable default argument is shared across calls — use `tags=None` and `tags = [] if tags is None else tags`.
```

## Install

As a plugin:

```
/plugin marketplace add neeraj-satyaki/claude-code-dry-run
/plugin install dry-run@dry-run
```

Or copy the skill by hand:

```bash
git clone https://github.com/neeraj-satyaki/claude-code-dry-run
cp -r claude-code-dry-run/skills/dry-run ~/.claude/skills/
```

## Usage

```
/dry-run <file|function|snippet>                 # pick entry + inputs, ask if unclear
/dry-run <function> with <args>                  # trace one call
/dry-run <file> --break <line|function>          # fast-forward, then step from breakpoint
/dry-run <file> --watch <var1,var2>              # only show watched variables
/dry-run <file> --bug "<symptom>"                # find the first step that diverges
/dry-run <file> --run                            # also execute for real and compare
```

Or just ask: *"dry run `parse_order` with an empty cart"*, *"step through this"*, *"what happens when I call X with Y?"*

While a trace is running you can drive it like a real debugger: `step`, `next`, `into`, `out`, `continue`, `print <expr>`, `set <var>=<value>`.

## What it tracks

- **Call stack:** frames are indented, with entry args and return values.
- **Branches:** each condition with real values substituted (`3 > 5 → False`).
- **Loops:** each iteration, collapsed with an invariant when the loop is long.
- **Aliasing and mutation:** when two names share one object, both are updated.
- **Side effects:** I/O, DB writes, network calls and globals are flagged.
- **Exceptions:** where one is raised, each frame it unwinds, and where it is caught.
- **Async and concurrency:** the assumed interleaving, plus the other orders that are possible.
- **Unknowns:** values from time, randomness or APIs are marked `UNKNOWN`, and it branches on the values that change the path.
- **Language traps:** a built-in checklist (JS coercion, Go loop capture, C undefined behaviour, SQL NULLs, shell word splitting, and more).

## Why not a real debugger?

Use one when you can. `/dry-run` is for the cases where you can't, or where a real debugger is slow:

- The code needs a database, a GPU, credentials or a cluster you don't have locally.
- You want to know *why*, not just *what*: every step comes with its reasoning.
- You are reviewing a PR, or learning a codebase you can't build yet.
- You want a polyglot debugger with one interface for every language.

With `--run`, it checks its own trace against real execution (`pdb`, `node inspect`, `gdb`/`lldb`, `dlv`, `jdb`, or prints), so a mistake in its reasoning shows up. It never executes code that touches production systems, real networks, file deletion or paid services.

## Limits

A trace is reasoning, not execution. On long or highly concurrent paths it can drift, so use `--run` when it matters.

## Contributing

Issues and PRs are welcome, especially new language traps and example traces that show it failing.

## License

MIT
