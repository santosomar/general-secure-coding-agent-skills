---
name: api-design-assistant
description: Reviews and designs API contracts — function signatures, REST endpoints, library interfaces — for usability, evolvability, and the principle of least surprise. Use when designing a new public interface, when reviewing an API PR, when the user asks whether a signature is well-designed, or when planning a breaking change.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "design-pattern-suggestor, code-refactoring-assistant"
---

# API Design Assistant

An API is a **promise to strangers.** Every decision you make now constrains every user forever — or forces a breaking change. Design for the caller you'll never talk to.

## The three questions

For every public surface, ask:

1. **Can a stranger call this correctly on the first try?** (Usability)
2. **Can you add to it without breaking existing callers?** (Evolvability)
3. **Does it do what the name says — and nothing else?** (Surprise)

## Usability smells

| Smell                             | Why it hurts                                            | Fix                                          |
| --------------------------------- | ------------------------------------------------------- | -------------------------------------------- |
| Boolean parameter                 | `save(true)` — true what? The call site is unreadable.  | Two methods, or an enum: `save(Overwrite.YES)` |
| Positional params > 3             | `create(a, b, c, d, e)` — which is which?               | Named/keyword params, or a config object     |
| Inconsistent naming               | `getUser()` but `fetchOrder()` but `loadProduct()`      | Pick one verb per concept. Grep before naming. |
| Stringly-typed                    | `setMode("fast")` — typo → silent no-op                 | Enum, or fail loudly on unknown strings      |
| Return `null` for "not found"     | Every caller must check; most won't                     | Optional/Maybe, or throw, or a sentinel — be consistent |
| Out parameter                     | `parse(input, &result, &error)`                         | Return a struct/tuple. Out params are C legacy. |
| Required call order               | Must call `init()` before `use()`                       | Make it impossible to get wrong: `use()` inits if needed, or the constructor inits |

## Evolvability — can you change it later?

| Property                          | Makes future changes…                                   |
| --------------------------------- | ------------------------------------------------------- |
| Accepts an options object/struct  | Easy to add fields. Positional args → every addition is breaking. |
| Returns an object, not a tuple    | Can add fields. Tuple → new field breaks destructuring. |
| Versioned (REST: `/v1/`, library: semver) | Possible. Unversioned → every change is scary.   |
| Tolerates unknown input fields    | Old clients can talk to new servers                     |
| Output fields nullable/optional by default | Can add a field old clients don't read           |

**Adding is cheap. Removing is forever.** Every parameter, every field, every endpoint: could you live with it for 5 years?

## Surprise — the hidden contract

The name IS the documentation most callers read. If behavior doesn't match the name:

- `getUser(id)` that *creates* a user if missing → surprise. Call it `getOrCreateUser`.
- `list()` that returns the first 100 → surprise. Call it `listPage` or document loudly.
- `setX(v)` that also triggers a network call → surprise. Setters should be cheap.
- `close()` that can't be called twice → surprise. Idempotent close is the convention.

## Worked example — function review

**Proposed:**

```python
def send_email(to, subject, body, html, cc, bcc, attachments, retry, async_):
    ...
```

**Review:**

| Issue | Severity | Fix |
| --- | --- | --- |
| 9 positional parameters | High — call sites will be unreadable | Required: `to, subject, body`. Rest: keyword-only with defaults. |
| `html` is a boolean | Medium — `send_email(..., True, ...)` means what? | `body_format: Literal["text", "html"] = "text"` |
| `async_` — name with trailing underscore | Low — Python keyword collision, but ugly | `background: bool` or `defer: bool` |
| `attachments` type unclear | Medium — list of paths? bytes? file objects? | Type annotation + docstring. Or a typed `Attachment` class. |
| No way to add headers later without a 10th param | High — evolvability | Wrap the optional stuff: `send_email(to, subject, body, *, options: EmailOptions = None)` |

**Suggested:**

```python
def send_email(
    to: str | list[str],
    subject: str,
    body: str,
    *,
    body_format: Literal["text", "html"] = "text",
    cc: list[str] = (),
    bcc: list[str] = (),
    attachments: list[Attachment] = (),
    retry: int = 0,
) -> EmailResult:
```

Keyword-only after `*`. `async_` dropped — make it a separate `send_email_async()`. Return a result object so you can add `message_id`, `delivered_at`, etc. later.

## REST-specific checklist

- Nouns in paths, verbs in methods: `GET /users/42`, not `GET /getUser?id=42`.
- Status codes carry meaning: 404 for not-found, 400 for bad input, 422 for valid-but-rejected. Don't `200 {"error": "..."}`.
- Pagination from day one. `?limit=&cursor=` — you'll need it once the table grows.
- Error body has a stable shape. `{"error": {"code": "...", "message": "..."}}` — clients pattern-match on it.
- `PATCH` for partial update, `PUT` for full replace — don't make PUT mean PATCH.

## Do not

- **Do not** add a parameter "just in case." Every parameter is a promise to keep working.
- **Do not** overload one endpoint/function to do five things controlled by a flag. Five things = five names.
- **Do not** return different shapes from one endpoint based on a query param. One endpoint, one shape.
- **Do not** break a public API in a minor version. If the change is breaking, the version bump is major — that's what major means.
- **Do not** ship an undocumented API and call it "internal." If it's reachable, someone will use it, and now it's public whether you meant it or not.

## Output format

```
## Usability
| Issue | Severity | Fix |
| ... | ... | ... |

## Evolvability
<what happens when you need to add X — can you, without breaking callers?>

## Surprise
<does behavior match the name? list mismatches>

## Suggested signature / contract
<concrete revision>
```
