---
name: legacy-code-summarizer
description: Summarizes undocumented legacy code by inferring intent from structure, naming, data flow, and calling context — explicitly flagging what's inferred vs. what's certain. Use when onboarding to inherited code, when documentation is missing or wrong, or when deciding whether legacy code is safe to change.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-summarizer, code-comment-generator, pseudocode-extractor"
---

# Legacy Code Summarizer

Legacy code has no docs, misleading names, and commit history that says `fix`. The summary you produce is **archaeology** — reconstructing intent from artifacts. Be honest about what's deduction vs. what's guesswork.

Differs from → `code-summarizer`: that one assumes the code is readable. This one assumes it isn't.

## Evidence sources — in order of reliability

| Source                          | Reliability | What it tells you                                    |
| ------------------------------- | ----------- | ---------------------------------------------------- |
| What the code *does* (dataflow) | High        | Actual behavior. Can't lie.                          |
| How it's called (grep callers)  | High        | What callers pass in, what they do with the result   |
| Tests (if any)                  | High        | Someone wrote down an expectation                    |
| Variable/function names         | Medium      | Might be right. Might be copy-pasted from elsewhere. |
| Comments                        | Low         | Often stale. Trust the code over the comment.        |
| Commit messages                 | Low         | `"fix bug"`, `"wip"`, `"asdf"`                       |

## Step 1 — Trace the data, not the names

Start from inputs, follow to outputs. Ignore what things are *called*; watch what *happens* to them.

```c
int proc_rec(char *buf, int n) {       // name tells you nothing
    int k = 0;
    for (int i = 0; i < n; i++) {
        if (buf[i] == 0x1F) k++;        // 0x1F = ASCII unit separator
    }
    return k + 1;
}
```

**Data trace:** counts `0x1F` bytes in `buf[0..n]`, returns count + 1. That's "count fields in a unit-separator-delimited record." The `+1` is fence-post: N separators → N+1 fields. Now you know what `proc_rec` means: "process record" → actually "count fields in record."

## Step 2 — Look at callers

```c
int nf = proc_rec(line, len);
char **fields = malloc(nf * sizeof(char*));
split_on(line, len, 0x1F, fields);
```

Caller uses the result to size an allocation for `split_on` with the same delimiter. Confirms: `proc_rec` is pre-counting fields so the split can allocate exactly. The abstraction is "two-pass field split: count pass, then split pass."

## Step 3 — Mark confidence explicitly

Every claim in the summary gets a confidence tag:

- **[observed]** — this is what the code does, period.
- **[inferred]** — this is probably why, based on callers/structure.
- **[guess]** — name/comment suggests this, unverified.
- **[unknown]** — can't tell. Flag for a human who knows the history.

## Worked example output

**Code:** 400 lines of Perl, `sub munge_data`, no comments, written 2009.

**Summary:**

```
## munge_data($ref, $mode)

### Behavior [observed]
- Walks a nested hashref `$ref` in-place.
- In mode 'C': lowercases all string leaf values, trims whitespace.
- In mode 'V': validates leaf values against regexes in the package-global
  %PATTERNS (keyed by parent hash key). Dies on mismatch.
- In any other mode: no-op (falls through the if/elsif chain).
- Mutates `$ref` directly. Returns nothing useful ($ref, but caller ignores it).

### Intent [inferred]
- 'C' = "canonicalize", 'V' = "validate" — based on the operations, not the letters.
- Called in sequence: `munge_data($d, 'C'); munge_data($d, 'V');` in 3 of 4 call
  sites. The 4th calls only 'V' — that input is apparently pre-canonicalized.
- %PATTERNS is populated from `config/field_rules.txt` at module load.

### Hazards [observed]
- Any mode other than 'C'/'V' silently does nothing — no error. Typo in mode
  string = silent skip. [→ seen in git log: commit a3f891 "fix: was passing 'c'
  not 'C'" — this bit someone]
- Deep recursion, no depth limit. Cyclic `$ref` → infinite loop.
- %PATTERNS global — if two threads load different configs, races.

### Unknowns
- [unknown] Why mode 'V' dies instead of returning an error list. Every caller
  wraps it in eval{} — suggests dying was a mistake, worked around.
- [guess] The 4th call site (V-only, in `import_legacy.pl`) might be handling
  data that's already canonical from an upstream system. Or might be a bug.
```

## Archaeology tools

| Question                               | Command                                                   |
| -------------------------------------- | --------------------------------------------------------- |
| Who calls this?                        | `rg -w funcname` across the repo                          |
| When was this line last touched?       | `git blame -w <file>`                                     |
| What did it look like before?          | `git log -p --follow -- <file>`                           |
| Who wrote it originally?               | `git log --diff-filter=A -- <file>` (first commit adding) |
| Was there ever a comment here?         | `git log -p -L <line>,<line>:<file>`                      |
| What else changed in the same commit?  | `git show <sha>` — context for why                        |

## Do not

- **Do not** trust names over behavior. `validate()` might mutate. `get_x()` might have side effects. Read the body.
- **Do not** write a summary that sounds confident about guesses. "This function validates input" [guess] is very different from "[observed]." Tag it.
- **Do not** summarize in isolation. The callers are half the story — they tell you which inputs actually occur and what's done with outputs.
- **Do not** clean up the code while summarizing. Understand first, change later. Changing what you don't understand is how legacy code gets worse.

## Output format

```
## <function/module name>

### Behavior [observed]
<what it does — traced from code, no interpretation>

### Intent [inferred]
<why — based on callers, structure, git history>

### Hazards [observed]
<footguns, silent failures, global state, thread-unsafety>

### Unknowns
<[unknown] and [guess] items — what a human with context should confirm>

### Evidence
<callers found, git commits consulted, tests referenced>
```
