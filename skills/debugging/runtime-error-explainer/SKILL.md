---
name: runtime-error-explainer
description: Translates cryptic runtime error messages and stack traces into understandable explanations, pointing to the concrete line at fault and the most likely fix. Use when a user pastes an error they don't understand, when a stack trace is deep and the user doesn't know where to start, or when an error message misleads about the real cause.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "bug-localization, bug-reproduction-test-generator"
---

# Runtime Error Explainer

Turn a stack trace into a one-line answer: **what broke, where, and why.**

## Read the stack in the right order

| Language / Runtime | Read direction                        | Your code is usually                     |
| ------------------ | ------------------------------------- | ---------------------------------------- |
| Python             | Bottom-up (most recent call last)     | Near the bottom, above the library frame |
| Java / Kotlin      | Top-down (most recent call first)     | Near the top, below the exception line   |
| JavaScript / Node  | Top-down                              | First non-`node_modules` frame           |
| Go (panic)         | Top-down, per goroutine               | First frame in the panicking goroutine   |
| Rust               | Top-down (with `RUST_BACKTRACE=1`)    | First non-`std`/`core` frame             |
| C/C++ (gdb `bt`)   | Top-down, `#0` is crash site          | First frame with source you own          |

**Rule:** Find the first stack frame in code the user controls. That's where they look first, regardless of what the error message says.

## Errors that lie about where the problem is

The message points at line X. The bug is somewhere else. Catalog:

| Error                                    | Where it points              | Where the bug actually is                             |
| ---------------------------------------- | ---------------------------- | ----------------------------------------------------- |
| `NullPointerException` / `NoneType has no attribute` | The dereference        | The earlier line that was supposed to assign the value |
| `KeyError` / `undefined is not a function` | The lookup              | The code that built the dict/object and omitted the key |
| `UnboundLocalError` (Python)             | The read of the variable     | Any line in the function that assigns to it — Python decided it's local |
| `SyntaxError: unexpected EOF`            | Last line of the file        | An unclosed bracket/quote somewhere above — bisect by deleting halves |
| `StackOverflowError` / `RecursionError`  | Deep in the recursion        | The base case (missing or wrong), not the recursive call |
| SIGSEGV at `0x0`                         | The crash site               | Whatever returned NULL and wasn't checked — walk up the stack |
| `ConcurrentModificationException`        | The iterator's `.next()`     | Whatever mutated the collection during iteration — different line, maybe different thread |
| CORS error (browser console)             | The frontend fetch           | The **backend** — missing `Access-Control-Allow-Origin` header |
| `ECONNREFUSED`                           | The client connect call      | The server isn't running / wrong port / firewall — not a code bug at all |

## Chained/wrapped exceptions

`Caused by:` (Java), `The above exception was the direct cause of` (Python), `InnerException` (.NET). **The innermost cause is the real error.** Everything above it is packaging.

Unwrap until you hit the bottom. Explain *that*.

## Translation templates

- **NPE/AttributeError:** "`<var>` is null at `<file>:<line>`. It was supposed to be set by `<earlier-line>`, which `<didn't run | returned null | ran with wrong args>`."
- **KeyError/missing attribute:** "`<key>` isn't in `<object>` at `<file>:<line>`. `<object>` was built at `<earlier-line>` — `<key is conditional and the condition was false | key name is misspelled | the shape changed upstream>`."
- **Type error:** "`<fn>` at `<file>:<line>` expected `<type>` but got `<actual>`. `<actual>` came from `<source>` — `<the API changed | a transform is missing | wrong variable passed>`."
- **Timeout/Connection refused:** "This isn't a code bug. `<service>` at `<host>:<port>` is `<not running | unreachable | slow>`. Verify: `<concrete check command>`."

## Worked example

**Input:**

```
Traceback (most recent call last):
  File "/app/api/handlers.py", line 89, in get_order
    return serialize(order, user.preferences.currency)
  File "/app/models/user.py", line 34, in __getattr__
    return self._data[name]
KeyError: 'preferences'
```

**Analysis:** Python → bottom-up → error is `KeyError: 'preferences'`. First user-controlled frame is `handlers.py:89`, which accesses `user.preferences`. The `__getattr__` frame is a dynamic attribute proxy — it reports what was missing but not why.

**Explanation:**

> `user` has no `preferences` attribute at `handlers.py:89`.
>
> `user` here comes from whatever loaded it — likely a `User.load()` call earlier in `get_order`. The `preferences` field was never populated. Three common reasons: (a) the user row in the DB has no preferences set and the model doesn't default it; (b) the ORM query is doing a partial `SELECT` that skips the column; (c) `preferences` was recently renamed/moved.
>
> First check: `print(user._data.keys())` right before line 89.

## Edge cases

- **No stack trace, just an exit code:** `exit 137` → OOM-killed; `exit 139` → SIGSEGV. Look at system logs, not the app.
- **Stack trace is all library frames:** The user's code is the thing that *called into* the library. Find the boundary.
- **Async/event-loop traces:** The stack is shallow and unhelpful because the real caller is on the other side of an `await`. Look for "task was created at" / `asyncio` origin tracking.
- **Minified JS / stripped binary:** Stack frame names are garbage. Ask for source maps / debug symbols before continuing.

## Do not

- **Do not** paraphrase the error message. "You got a NullPointerException" is not an explanation.
- **Do not** explain what a `NullPointerException` *is*. Explain why *this one* happened.
- **Do not** suggest wrapping the line in `try/except` as a fix. That's a symptom suppressor.
- **Do not** stop at the crash line when the table above says the bug is upstream.

## Output format

```
## What broke
<one sentence — the real error, unwrapped>

## Where to look
<file>:<line>  — <why this frame, not the one the error points at>

## Likely cause
<ranked list, 1–3 items, most likely first>

## Verify with
<one concrete diagnostic step — a print, a breakpoint, a query>
```
