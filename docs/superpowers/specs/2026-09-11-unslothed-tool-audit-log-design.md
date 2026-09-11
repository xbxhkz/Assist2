# Unslothed Tool Audit Log — Design Spec

## Context

Sixth sub-project of the initiative to fork [Unsloth](https://github.com/unslothai/unsloth/) and
extend it into a local AI agent platform. Sub-projects 1–5 (vision tools, code intelligence,
packaging, draft-model chooser, workspace directory) are merged and shipping.

The master specification (`Master Specification Prompt — Advanced Local AI Desktop Agent.md`) asks
for three related capabilities: a formal tool registry (§3, §37, §38), a permission model with risk
levels and modes (§25, §26), and an audit log of every tool invocation (§27). The owner chose to
build all three, **audit first**, as separate specs.

Audit first is not only the lowest-risk ordering. Today there is no record of what tools actually
run, with what arguments, how often. Assigning risk levels in the governance piece without that
evidence would be guesswork; with it, the classification follows from observed traffic.

## What already exists, and what is actually missing

Investigation reshaped this sub-project before design started. §9 of the master spec is explicit
about not rebuilding what exists under another name, and considerably more exists than the spec's
framing implies.

**Already built upstream:**

| Capability | Where |
|---|---|
| Permission modes `off` / `ask` / `auto` / `full` | `core/inference/studio_tool_loop.py` |
| Approval handshake — request/wait/resolve/abort, approval IDs, 1-hour timeout | `state/tool_approvals.py` |
| Argument-aware risk classification | `tools.py:6558 is_high_risk_tool_call`, plus `is_potentially_unsafe_tool_call` |
| Read-only tool allowlist | `tools.py:4535 _ALWAYS_SAFE_TOOLS` |
| Tool schemas | `tools.py:9837 ALL_TOOLS` |

Two consequences worth recording now, because they will shape the governance sub-project:

- The existing risk classification is **argument-aware** (`is_high_risk_tool_call(name, arguments)`)
  and fails closed on unknown tools. §25's static per-tool LOW/MEDIUM/HIGH/CRITICAL labels are
  *less* expressive. Implementing §25 naively would be a downgrade, not an upgrade.
- The four existing modes already map closely onto §26's Safe/Assisted/Autonomous/Restricted.

**Genuinely missing, and therefore this sub-project:** there is no record of tool invocations. No
table, no rows, no API. Confirmed by grep across `storage/` — the word "audit" appears in
`tools.py` only in an unrelated allowlist comment and a GraphQL directive.

## Decisions taken by the owner

| Question | Decision |
|---|---|
| Which of the three pieces first | All three, **audit first**, as separate specs |
| What the audit log is for | **Forensics** — "what did it do to my machine", not a live activity ticker |
| How much of each call to keep | **Full arguments** (redacted), **capped results + hash**, **errors whole** |
| Where the hook goes | Inside `execute_tool` (approach A below) |

## Goal

Record every tool invocation with enough fidelity to answer, after the fact, what the AI did to this
machine — without ever breaking a tool call, and without becoming a credential store.

## Architecture

### Why the hook goes inside `execute_tool`

`execute_tool(name, arguments, ...)` at `tools.py:10031` is the single chokepoint. It has **three**
callers:

```
core/inference/llama_cpp.py:24121          <- DO-NOT-EDIT file for this fork
core/inference/safetensors_agentic.py:1337
core/inference/studio_tool_loop.py:1246
```

Recording at the call sites is therefore impossible: it would require editing `llama_cpp.py`, and it
would need three edits instead of one. A single hook inside `execute_tool` covers all three paths,
including the two agentic loops a call-site approach would silently miss.

It also covers **MCP tools**, which dispatch through the same function (`tools.py:10101`). That
matters more than it first appears: MCP tools are arbitrary third-party code, added by the user at
runtime and unknown when this fork was built. They are simultaneously the least predictable thing
the agent can invoke and the surface with no audit trail at all today.

An import-time monkeypatch of `tools.execute_tool` was considered and rejected. All three callers do
`from core.inference.tools import execute_tool`, binding the function object at import; patching the
module attribute afterwards misses them unless load order cooperates. That is the same late-binding
trap as `Depends()` capturing at import time, which has already cost this project once. Too fragile
for a security-relevant record.

### Components

All new code is fork-owned. `tools.py` gains only the call.

```
core/inference/tool_audit/__init__.py    around(fn, *args, **kwargs) — the single entry point
core/inference/tool_audit/redaction.py   secret scrubbing
storage/tool_audit_db.py                 table, writes, queries, pruning
routes/tool_audit.py                     read-only API
studio/frontend/...                      activity panel
```

`main.py` gains two lines — an import and an `include_router` — exactly mirroring what the
draft-model sub-project already added there. That file is additive-only for this fork, not
untouchable: it currently stands at 2 insertions / 0 deletions against upstream, and this takes it
to 4.

### The seam

An earlier draft of this spec said "a `try/finally` around the existing dispatch inside
`execute_tool`". **That is not achievable additively**, and the correction matters enough to record.
`execute_tool` spans `tools.py:10031-10189` — roughly 160 lines of if/elif dispatch with many
`return` statements. Wrapping that body in `try:` means re-indenting ~145 lines, so every one of
them shows as modified. It would be the single largest conflict surface the fork owns, in the most
contended upstream file.

The additive form is a **definition-time shadow**, appended immediately after the function ends
(line 10189, before `_opt_int` at 10192):

```python
# --- fork: tool audit -------------------------------------------------------
_execute_tool_unaudited = execute_tool


@functools.wraps(_execute_tool_unaudited)
def execute_tool(*args, **kwargs):  # noqa: F811 - deliberate shadow, see above
    from core.inference import tool_audit

    return tool_audit.around(_execute_tool_unaudited, *args, **kwargs)
```

~8 insertions, **zero deletions, zero re-indentation**. `functools` is already imported at
`tools.py:10`.

**Why this is not the monkeypatch rejected above.** The rejected approach patched
`tools.execute_tool` from *another* module after import, so whether a caller got the wrapper
depended on import order — and all three callers bind the function object via
`from core.inference.tools import execute_tool`, so a late patch misses them. This shadow runs
inside `tools.py`'s own module body, before that module finishes executing and therefore before any
importer can bind anything. Deterministic, not order-dependent.

**Why `accepts_kwarg` keeps working.** `studio_tool_loop.py:1230-1244` calls
`accepts_kwarg(execute_tool, "conversation_branch")` and friends before forwarding those kwargs, and
`accepts_kwarg` (`core/inference/tool_stream_exec.py:35`) uses `inspect.signature(func)`.
`inspect.signature` follows `__wrapped__`, which `functools.wraps` sets, so it reports the ORIGINAL
parameters rather than `(*args, **kwargs)`. Verified empirically before adopting this approach — a
bare wrapper without `functools.wraps` would have silently disabled conversation-branch and budget
forwarding, which no existing test covers.

The additive-only rule is what makes this worth the care: a purely additive hunk survives an
upstream rewrite of neighbouring lines, where a modified line conflicts the moment upstream touches
it. `tools.py` goes from 47 to ~55 insertions, still zero deletions.

### Two-phase write

Insert on entry with `outcome='running'`; update on exit. The second write costs ~1 ms against tool
calls measured in seconds, and buys two things a single terminal write cannot:

- a crash, kill or power loss mid-`terminal` leaves a visible `running` row rather than no evidence
- in-flight calls are observable

For a forensic log, "started and never finished" is precisely the case most worth recording.

## Schema

One row per invocation, in the existing Studio SQLite database.

| Column | Purpose |
|---|---|
| `id` | primary key |
| `ts`, `duration_ms` | when, how long |
| `session_id`, `thread_id` | correlate to a conversation; both are already parameters of `execute_tool` |
| `tool_name` | what ran |
| `arguments_json` | full, redacted — the field that answers "what did it do" |
| `paths_json` | best-effort extraction of path arguments, for search only |
| `redacted` | 1 when any redaction fired, so scrubbed is distinguishable from clean |
| `outcome` | `running` / `ok` / `error` / `cancelled` |
| `result_head`, `result_tail`, `result_bytes` | first 4 KB and last 4 KB of the result (8 KB stored per row at most); `result_bytes` is the TRUE length, so truncation is always detectable |
| `result_sha256` | prove two runs produced identical output |
| `error_text` | kept **whole** — a truncated traceback is useless |
| `approval_id`, `approved`, `permission_mode` | ties the call to the existing approval handshake and the mode in force at the time |

Indexed on `ts`, `tool_name`, `session_id`.

**The hash covers the redacted payload, not the original.** Hashing the original would let a short
secret be confirmed by brute force against the stored digest. Hashing post-redaction keeps
"same output?" working for ordinary content and closes that avenue.

**`paths_json` is best-effort and advertised as such.** It pulls known path arguments (`edit_file`'s
target and so on) purely so the panel can search by path. A general mechanism would mean teaching the
audit layer every tool's semantics — exactly the coupling a registry is meant to remove. Full
arguments are recorded regardless, so nothing is lost; only indexed search is approximate.

## Redaction

The audit log must not become the highest-value file on disk. Four layers, because no single one
suffices:

1. **Credential-referencing MCP calls.** MCP tools are arbitrary third-party code and are the least
   trustworthy surface reaching `execute_tool`. Upstream already classifies them:
   `_mcp_arguments_reference_sensitive(arguments)` (`tools.py:4234`) is true when an MCP call's
   arguments name a credential path, a secret environment variable, or a cloud-metadata host. Reuse
   that signal — when it fires, the result is **withheld** and the row records that, rather than
   storing whatever the server returned. Reusing upstream's classifier rather than writing a second
   one also means the two cannot drift apart.
2. **Key-name matching.** Argument keys matching `password|token|secret|api_key|authorization|credential`
   have values replaced regardless of shape.
3. **Pattern matching** over arguments and result text: `sk-`, `ghp_`, `github_pat_`, AWS `AKIA`,
   `Bearer `, and long high-entropy hex/base64 runs.
4. **The `redacted` flag**, so a scrubbed record is never mistaken for a clean one.

An earlier draft of this spec named `get_env` as a tool to deny-list. It is not a tool in this
codebase — it appears only inside the docstring above, as an example of what a third-party MCP
server might expose. Recorded because the mistake points at the real shape of the risk: the
dangerous surface here is not a known tool with a known name, it is **whatever an MCP server
chooses to offer**, which is exactly why the reused classifier is the right layer.

### The gap, stated rather than papered over

A secret matching no pattern and sitting under an innocuous key — a bare passphrase inside a
`terminal` command — can still land in the log. Matching against the app's own `credential_secrets`
store would catch those exactly, but would require decrypting every stored credential to scan each
record, widening exposure in order to reduce it. Rejected for v1.

Mitigating fact: this table lives in the Studio data root **alongside the auth database**, which
already holds credentials. The audit log does not create a new trust boundary; it sits inside the
existing one.

## Retention

- Keep **365 days**, hard cap **250 000 rows**, whichever binds first
- Both configurable through the existing `app_settings` table
- **Pruning writes its own audit row**: "pruned N rows older than T"

A forensic log that silently drops history is worse than no log, because "it never happened" and
"it scrolled off" become indistinguishable. With the prune recorded, absence is always explainable.

## Error handling

**Auditing must never break a tool call.** If the write fails — disk full, database locked, a bug in
a redaction regex — the tool still executes and returns normally. The entire record path sits inside
a guard using a **bare** `except`: this project has already established that "never-raises" needs
one, because `OverflowError` and unhashable types slip past `except Exception` in practice.

Silence is its own failure mode. An audit log that quietly stops working leaves holes with no
indication, so failures increment a counter and surface as **"audit degraded"** in the API and
panel. Same principle as pruning recording itself: absence must always be explainable.

## UI

A read-only panel: filter by tool, session and time; expand a row for full arguments, result
head/tail and error text. This matches §28's "expand tool calls and inspect their inputs and
outputs". Exact placement follows the existing panel pattern and is settled in the implementation
plan rather than guessed here.

## Testing

Every guard gets a negative control **demonstrated to fail** before its fix. Eight controls in this
project have turned out inert, one caught mid-run, so "the test passes" is not evidence on its own.

| Test | Control proving it is live |
|---|---|
| a successful call writes `ok` | — |
| a failing call writes `error` with the full traceback | remove the except path; the test must fail |
| **a raising recorder does not break `execute_tool`** | make the recorder raise; assert the tool's result is still correct |
| each redaction pattern fires | **and a non-secret is NOT redacted** — over-redaction would quietly gut the log |
| the hook lives inside `execute_tool`, not at the call sites | AST check; this is what guarantees the `llama_cpp.py` caller is covered |
| pruning writes its own row | — |
| a crash between insert and update leaves a `running` row | — |

Rows three and five are not shippable without. Three is the difference between an audit feature and
an outage. Five is what stops a later "simplification" moving the hook to the call sites and
silently losing the forbidden file's path.

## Risks

**The seam grows.** `tools.py` goes from 47 to ~62 insertions. Still zero deletions, so still
conflict-resistant, but the fork's footprint in the most-contended upstream file is now larger.

**Write volume on chatty sessions.** Two writes per tool call. Negligible per call, but an agentic
loop making hundreds of calls per turn multiplies it. The row cap bounds total size; it does not
bound write rate. If this ever matters, batching the update is the escape hatch — deliberately not
built now.

**Redaction is regex-based and therefore imperfect in both directions.** It can miss a secret
(gap above) and it can over-redact, mangling an argument that merely looked like a token. The
`redacted` flag plus the "non-secret is not redacted" control are the mitigations.

**This log is a disclosure target.** It records what the AI did on this machine, in detail. It
inherits the data root's protection and nothing more.

## What deliberately does not change

No change to how tools execute, what they may do, or when approval is requested. The existing
permission modes, approval handshake and risk classification are untouched. This sub-project only
*observes*. Governance changes land in the next spec, informed by what this one records.

## Out of scope

- Risk levels, declared permissions, permission-mode changes (the governance sub-project)
- `list_tools` / `get_tool_information` discovery (the discovery sub-project)
- Any change to `_ALWAYS_SAFE_TOOLS` or `is_high_risk_tool_call`
- Matching recorded values against the credential store (see the redaction gap)
- Live streaming of in-flight calls to the UI; the panel polls
