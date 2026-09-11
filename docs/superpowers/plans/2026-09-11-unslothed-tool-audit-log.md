# Unslothed Tool Audit Log Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Record every tool invocation — what ran, with what arguments, what came back, whether it ran sandboxed — with enough fidelity to answer "what did the AI do to this machine" after the fact.

**Architecture:** A single fork-owned `around()` wrapper installed over `execute_tool` by a definition-time shadow at the end of `tools.py`. It writes a row on entry (`outcome='running'`) and updates it on exit. All recording is guarded so a failure can never break a tool call. A read-only API and a `/tool-audit` panel expose the records.

**Tech Stack:** Python 3.12, FastAPI, SQLite (`sqlite3` stdlib), pytest; React 19 + TanStack Router + TypeScript on the frontend.

**Spec:** `C:\Users\Admin\odysseus\docs\superpowers\specs\2026-09-11-unslothed-tool-audit-log-design.md`

## Global Constraints

Every task's requirements implicitly include this section.

- **Repo:** `C:\Users\Admin\unsloth`, branch `unslothed-release`. This is a fork of `unslothai/unsloth`. The remote `origin` is UPSTREAM and **must never be pushed to**; the fork remote is `fork`.
- **Additive-only in upstream files.** `studio/backend/core/inference/tools.py` currently stands at **47 insertions / 0 deletions** against upstream. This plan adds ~8 more. **Zero deletions, zero re-indentation.** Verify with:
  `git diff --numstat $(git merge-base origin/main HEAD) -- studio/backend/core/inference/tools.py`
- **Do-not-edit files:** `core/inference/llama_cpp.py`, `routes/inference.py`, `pyproject.toml`.
- **`studio/backend/main.py` is additive-only, not untouchable.** It stands at 2 insertions / 0 deletions (the draft-model router). Task 5 takes it to 4. Never modify or delete an existing line there.
- **NEVER run the full backend test suite.** Upstream fixtures fabricate GGUF files up to 40 GB and filled the disk once. Run only the specific test files named in each task.
- **Test runner:** `C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest <file> -v -p no:cacheprovider` run from `C:\Users\Admin\unsloth\studio\backend`.
- **Every guard gets a negative control demonstrated to fail before its fix.** Eight controls in this project turned out inert, one caught mid-run. "The test passes" is not evidence. Where a step says "watch it fail", actually run it and read the failure.
- **Style:** this codebase writes keyword arguments with spaces around `=` (`session_id = session_id`). Match it.
- **Frontend gates:** `npx tsc -b --force --noEmit` and `npm run i18n:check:strict` must both pass, run from `studio/frontend`. Use `--force`; stale incremental state has produced phantom diagnostics here five times.
- **Commit trailer:** end every commit message with
  `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`

## File Structure

**Created:**

| Path | Responsibility |
|---|---|
| `studio/backend/storage/tool_audit_db.py` | table DDL, insert/finish, query, prune. Owns its schema; does not touch `studio_db.py` |
| `studio/backend/core/inference/tool_audit/__init__.py` | `around()` — the wrapper, plus the degraded counter |
| `studio/backend/core/inference/tool_audit/redaction.py` | secret scrubbing for arguments and result text |
| `studio/backend/routes/tool_audit.py` | read-only API |
| `studio/backend/tests/test_tool_audit_db.py` | storage tests |
| `studio/backend/tests/test_tool_audit_redaction.py` | redaction tests |
| `studio/backend/tests/test_tool_audit_recorder.py` | recorder + never-raises tests |
| `studio/backend/tests/test_tool_audit_seam.py` | AST + signature-preservation tests |
| `studio/frontend/src/features/tool-audit/tool-audit-page.tsx` | the panel |
| `studio/frontend/src/features/tool-audit/index.ts` | named export for the lazy route |
| `studio/frontend/src/app/routes/tool-audit.tsx` | route definition |

**Modified:**

| Path | Change |
|---|---|
| `studio/backend/core/inference/tools.py` | +8 lines appended after `execute_tool` ends (line 10189) |
| `studio/backend/main.py` | +2 lines (import, `include_router`) |
| `studio/frontend/src/app/router.tsx` | +2 lines (import, routeTree child) |
| `studio/frontend/src/i18n/locales/en.ts` | + a `toolAudit` section |
| `studio/frontend/src/i18n/locales/{ar,de,es,fr,hi,it,ja,ko,pt-br,ru,zh-CN}.ts` | + translated `toolAudit` section (11 files) |

---

### Task 1: Storage layer

**Files:**
- Create: `studio/backend/storage/tool_audit_db.py`
- Test: `studio/backend/tests/test_tool_audit_db.py`

**Interfaces:**
- Consumes: `storage.studio_db.get_connection()` (existing, `studio_db.py:1115`)
- Produces:
  - `record_start(*, tool_name: str, arguments_json: str, paths_json: str, redacted: bool, session_id: str | None, thread_id: str | None, disable_sandbox: bool) -> int` — returns row id
  - `record_finish(row_id: int, *, outcome: str, duration_ms: int, result_head: str, result_tail: str, result_bytes: int, result_sha256: str, error_text: str | None) -> None`
  - `query_entries(*, limit: int = 100, offset: int = 0, tool_name: str | None = None, session_id: str | None = None) -> list[dict]`
  - `get_entry(row_id: int) -> dict | None`
  - `prune(*, max_age_days: int = 365, max_rows: int = 250_000) -> int` — returns rows deleted

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_tool_audit_db.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Storage for the tool audit log.

Owns its own table and schema rather than extending studio_db's _ensure_schema,
so the fork adds a file instead of editing one.
"""

from __future__ import annotations

import time

import pytest

from storage import tool_audit_db


@pytest.fixture(autouse = True)
def _isolated_db(tmp_path, monkeypatch):
    """Point the studio data root at a temp dir so tests never touch real data."""
    monkeypatch.setenv("UNSLOTH_STUDIO_HOME", str(tmp_path))
    tool_audit_db.reset_for_tests()
    yield


def _start(**over):
    kw = dict(
        tool_name = "terminal",
        arguments_json = '{"command": "ls"}',
        paths_json = "[]",
        redacted = False,
        session_id = "sess-1",
        thread_id = "thread-1",
        disable_sandbox = False,
    )
    kw.update(over)
    return tool_audit_db.record_start(**kw)


def test_record_start_creates_a_running_row():
    row_id = _start()
    entry = tool_audit_db.get_entry(row_id)
    assert entry is not None
    assert entry["outcome"] == "running"
    assert entry["tool_name"] == "terminal"
    assert entry["duration_ms"] is None


def test_record_finish_completes_the_row():
    row_id = _start()
    tool_audit_db.record_finish(
        row_id,
        outcome = "ok",
        duration_ms = 12,
        result_head = "total 0",
        result_tail = "",
        result_bytes = 7,
        result_sha256 = "abc",
        error_text = None,
    )
    entry = tool_audit_db.get_entry(row_id)
    assert entry["outcome"] == "ok"
    assert entry["duration_ms"] == 12
    assert entry["result_bytes"] == 7


def test_error_text_is_stored_whole():
    """A truncated traceback is useless; this column is deliberately uncapped."""
    row_id = _start()
    long_tb = "Traceback\n" + ("frame\n" * 5000)
    tool_audit_db.record_finish(
        row_id,
        outcome = "error",
        duration_ms = 1,
        result_head = "",
        result_tail = "",
        result_bytes = 0,
        result_sha256 = "",
        error_text = long_tb,
    )
    assert tool_audit_db.get_entry(row_id)["error_text"] == long_tb


def test_query_filters_by_tool_and_session():
    _start(tool_name = "terminal", session_id = "a")
    _start(tool_name = "python", session_id = "a")
    _start(tool_name = "terminal", session_id = "b")
    assert len(tool_audit_db.query_entries(tool_name = "terminal")) == 2
    assert len(tool_audit_db.query_entries(session_id = "a")) == 2
    assert len(tool_audit_db.query_entries(tool_name = "terminal", session_id = "b")) == 1


def test_prune_by_row_cap_writes_its_own_row():
    """A forensic log that silently drops history is worse than no log: 'it never
    happened' and 'it scrolled off' must stay distinguishable."""
    for _ in range(10):
        _start()
    deleted = tool_audit_db.prune(max_rows = 4)
    assert deleted > 0
    remaining = tool_audit_db.query_entries(limit = 100)
    markers = [e for e in remaining if e["tool_name"] == "__prune__"]
    assert len(markers) == 1, "prune must record itself"
    assert str(deleted) in markers[0]["arguments_json"]


def test_prune_by_age():
    old_id = _start()
    with tool_audit_db._connect() as conn:
        conn.execute(
            "UPDATE tool_audit SET ts = ? WHERE id = ?",
            (time.time() - 400 * 86400, old_id),
        )
        conn.commit()
    fresh_id = _start()
    tool_audit_db.prune(max_age_days = 365)
    assert tool_audit_db.get_entry(old_id) is None
    assert tool_audit_db.get_entry(fresh_id) is not None
```

- [ ] **Step 2: Run it and watch it fail**

Run from `C:\Users\Admin\unsloth\studio\backend`:
```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_db.py -v -p no:cacheprovider
```
Expected: `ModuleNotFoundError: No module named 'storage.tool_audit_db'` — every test errors at collection.

- [ ] **Step 3: Implement the storage module**

Create `studio/backend/storage/tool_audit_db.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Persistent record of every tool invocation.

Owns its own table and its own schema check rather than extending studio_db's
_ensure_schema: the fork adds a file instead of editing one, which keeps the
upstream storage module at a zero-line diff.

The table is deliberately append-mostly. Rows are written twice -- once on entry
with outcome='running', once on exit -- so a crash mid-tool leaves visible
evidence rather than nothing at all.
"""

from __future__ import annotations

import json
import sqlite3
import time
from contextlib import contextmanager
from typing import Any, Iterator, Optional

from storage.studio_db import get_connection

_SCHEMA_READY = False

_DDL = """
CREATE TABLE IF NOT EXISTS tool_audit (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    ts               REAL    NOT NULL,
    duration_ms      INTEGER,
    session_id       TEXT,
    thread_id        TEXT,
    tool_name        TEXT    NOT NULL,
    arguments_json   TEXT    NOT NULL,
    paths_json       TEXT,
    redacted         INTEGER NOT NULL DEFAULT 0,
    disable_sandbox  INTEGER NOT NULL DEFAULT 0,
    outcome          TEXT    NOT NULL,
    result_head      TEXT,
    result_tail      TEXT,
    result_bytes     INTEGER,
    result_sha256    TEXT,
    error_text       TEXT
);
CREATE INDEX IF NOT EXISTS idx_tool_audit_ts ON tool_audit (ts DESC);
CREATE INDEX IF NOT EXISTS idx_tool_audit_tool ON tool_audit (tool_name);
CREATE INDEX IF NOT EXISTS idx_tool_audit_session ON tool_audit (session_id);
"""

# Written as a normal row so the log can always explain its own gaps.
PRUNE_MARKER_TOOL = "__prune__"


def reset_for_tests() -> None:
    """Forget that the schema was created; used by tests that swap data roots."""
    global _SCHEMA_READY
    _SCHEMA_READY = False


@contextmanager
def _connect() -> Iterator[sqlite3.Connection]:
    conn = get_connection()
    try:
        global _SCHEMA_READY
        if not _SCHEMA_READY:
            conn.executescript(_DDL)
            conn.commit()
            _SCHEMA_READY = True
        yield conn
    finally:
        conn.close()


def record_start(
    *,
    tool_name: str,
    arguments_json: str,
    paths_json: str,
    redacted: bool,
    session_id: Optional[str],
    thread_id: Optional[str],
    disable_sandbox: bool,
) -> int:
    with _connect() as conn:
        cur = conn.execute(
            """
            INSERT INTO tool_audit
                (ts, session_id, thread_id, tool_name, arguments_json, paths_json,
                 redacted, disable_sandbox, outcome)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'running')
            """,
            (
                time.time(),
                session_id,
                thread_id,
                tool_name,
                arguments_json,
                paths_json,
                1 if redacted else 0,
                1 if disable_sandbox else 0,
            ),
        )
        conn.commit()
        return int(cur.lastrowid)


def record_finish(
    row_id: int,
    *,
    outcome: str,
    duration_ms: int,
    result_head: str,
    result_tail: str,
    result_bytes: int,
    result_sha256: str,
    error_text: Optional[str],
) -> None:
    with _connect() as conn:
        conn.execute(
            """
            UPDATE tool_audit
               SET outcome = ?, duration_ms = ?, result_head = ?, result_tail = ?,
                   result_bytes = ?, result_sha256 = ?, error_text = ?
             WHERE id = ?
            """,
            (
                outcome,
                duration_ms,
                result_head,
                result_tail,
                result_bytes,
                result_sha256,
                error_text,
                row_id,
            ),
        )
        conn.commit()


def _row_to_dict(row: sqlite3.Row) -> dict[str, Any]:
    d = dict(row)
    d["redacted"] = bool(d.get("redacted"))
    d["disable_sandbox"] = bool(d.get("disable_sandbox"))
    return d


def get_entry(row_id: int) -> Optional[dict[str, Any]]:
    with _connect() as conn:
        conn.row_factory = sqlite3.Row
        row = conn.execute("SELECT * FROM tool_audit WHERE id = ?", (row_id,)).fetchone()
        return _row_to_dict(row) if row else None


def query_entries(
    *,
    limit: int = 100,
    offset: int = 0,
    tool_name: Optional[str] = None,
    session_id: Optional[str] = None,
) -> list[dict[str, Any]]:
    clauses: list[str] = []
    params: list[Any] = []
    if tool_name:
        clauses.append("tool_name = ?")
        params.append(tool_name)
    if session_id:
        clauses.append("session_id = ?")
        params.append(session_id)
    where = (" WHERE " + " AND ".join(clauses)) if clauses else ""
    params.extend([max(1, min(limit, 1000)), max(0, offset)])
    with _connect() as conn:
        conn.row_factory = sqlite3.Row
        rows = conn.execute(
            f"SELECT * FROM tool_audit{where} ORDER BY ts DESC, id DESC LIMIT ? OFFSET ?",
            params,
        ).fetchall()
        return [_row_to_dict(r) for r in rows]


def prune(*, max_age_days: int = 365, max_rows: int = 250_000) -> int:
    """Delete old rows, then excess rows. Records what it did as a row of its own."""
    cutoff = time.time() - max_age_days * 86400
    with _connect() as conn:
        deleted = conn.execute(
            "DELETE FROM tool_audit WHERE ts < ? AND tool_name != ?",
            (cutoff, PRUNE_MARKER_TOOL),
        ).rowcount
        over = conn.execute("SELECT COUNT(*) FROM tool_audit").fetchone()[0] - max_rows
        if over > 0:
            deleted += conn.execute(
                """
                DELETE FROM tool_audit WHERE id IN (
                    SELECT id FROM tool_audit
                     WHERE tool_name != ?
                     ORDER BY ts ASC, id ASC
                     LIMIT ?
                )
                """,
                (PRUNE_MARKER_TOOL, over),
            ).rowcount
        if deleted:
            conn.execute(
                """
                INSERT INTO tool_audit
                    (ts, tool_name, arguments_json, paths_json, redacted,
                     disable_sandbox, outcome, duration_ms)
                VALUES (?, ?, ?, '[]', 0, 0, 'ok', 0)
                """,
                (
                    time.time(),
                    PRUNE_MARKER_TOOL,
                    json.dumps(
                        {
                            "deleted": deleted,
                            "max_age_days": max_age_days,
                            "max_rows": max_rows,
                        }
                    ),
                ),
            )
        conn.commit()
        return deleted
```

- [ ] **Step 4: Run the tests and watch them pass**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_db.py -v -p no:cacheprovider
```
Expected: 6 passed.

- [ ] **Step 5: Prove the prune-marker control is live**

Temporarily delete the `if deleted:` INSERT block in `prune()`, re-run, and confirm
`test_prune_by_row_cap_writes_its_own_row` FAILS with "prune must record itself". Restore the block
and confirm it passes again. Do not skip this: eight controls in this project have turned out inert.

- [ ] **Step 6: Commit**

```bash
git add studio/backend/storage/tool_audit_db.py studio/backend/tests/test_tool_audit_db.py
git commit -m "feat(audit): storage layer for the tool audit log

Owns its own table and schema check rather than extending studio_db's
_ensure_schema, so the fork adds a file instead of editing one.

Two-phase by design: record_start writes outcome='running', record_finish
completes it. A crash mid-tool therefore leaves a visible running row instead of
no evidence.

prune() records itself as a row. A forensic log that silently drops history is
worse than no log, because 'it never happened' and 'it scrolled off' become
indistinguishable.

error_text is deliberately uncapped -- a truncated traceback is useless.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: Redaction

**Files:**
- Create: `studio/backend/core/inference/tool_audit/__init__.py` (empty package marker for now — Task 3 fills it)
- Create: `studio/backend/core/inference/tool_audit/redaction.py`
- Test: `studio/backend/tests/test_tool_audit_redaction.py`

**Interfaces:**
- Consumes: nothing from earlier tasks
- Produces:
  - `redact_arguments(arguments: dict) -> tuple[str, bool]` — `(json_string, was_redacted)`
  - `redact_text(text: str) -> tuple[str, bool]`
  - `extract_paths(arguments: dict) -> str` — JSON list of path-ish argument values
  - `REDACTED` — the literal replacement marker, `"[redacted]"`

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_tool_audit_redaction.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Redaction for the tool audit log.

The log must not become the highest-value file on disk. It must also not
over-redact: a log that scrubs ordinary arguments is quietly useless, which is
why every positive case here has a negative partner.
"""

from __future__ import annotations

import json

from core.inference.tool_audit import redaction


def test_key_named_secret_is_redacted():
    out, hit = redaction.redact_arguments({"api_key": "hunter2", "path": "/tmp/x"})
    assert hit is True
    data = json.loads(out)
    assert data["api_key"] == redaction.REDACTED
    assert data["path"] == "/tmp/x", "only the secret-named key should change"


def test_ordinary_arguments_are_NOT_redacted():
    """The control that stops over-redaction gutting the log."""
    out, hit = redaction.redact_arguments({"command": "ls -la /var/log", "timeout": 30})
    assert hit is False
    assert json.loads(out) == {"command": "ls -la /var/log", "timeout": 30}


def test_token_shaped_values_are_redacted_by_pattern():
    for secret in (
        "sk-abcdefghijklmnopqrstuvwxyz0123",
        "ghp_abcdefghijklmnopqrstuvwxyz0123456789",
        "AKIAIOSFODNN7EXAMPLE",
    ):
        out, hit = redaction.redact_text(f"export TOKEN={secret}")
        assert hit is True, f"{secret!r} should have been redacted"
        assert secret not in out


def test_plain_prose_is_NOT_redacted():
    """Second over-redaction control, on the text path."""
    text = "Compiled 42 files in 3.2 seconds with no errors."
    out, hit = redaction.redact_text(text)
    assert hit is False
    assert out == text


def test_nested_values_are_reached():
    out, hit = redaction.redact_arguments({"env": {"AUTHORIZATION": "Bearer abc123def456"}})
    assert hit is True
    assert "abc123def456" not in out


def test_extract_paths_finds_path_arguments():
    paths = json.loads(redaction.extract_paths({"path": "/a/b.txt", "count": 3}))
    assert paths == ["/a/b.txt"]


def test_extract_paths_is_empty_when_there_are_none():
    assert json.loads(redaction.extract_paths({"query": "hello"})) == []
```

- [ ] **Step 2: Run it and watch it fail**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_redaction.py -v -p no:cacheprovider
```
Expected: `ModuleNotFoundError: No module named 'core.inference.tool_audit'`.

- [ ] **Step 3: Create the package marker**

Create `studio/backend/core/inference/tool_audit/__init__.py` containing only:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Tool audit log. Task 3 adds around(); this file is the package marker."""
```

- [ ] **Step 4: Implement redaction**

Create `studio/backend/core/inference/tool_audit/redaction.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Scrub secrets before anything reaches the audit log.

Two failure directions matter equally. Missing a secret puts a credential on
disk. Over-redacting mangles ordinary arguments and quietly makes the log
useless -- which is why the test file pairs every positive case with a negative
one.

Known gap, accepted for v1 (see the design spec): a secret matching no pattern
and sitting under an innocuous key -- a bare passphrase inside a terminal
command -- still gets through. Closing it would mean decrypting the credential
store to scan every record, widening exposure in order to reduce it.
"""

from __future__ import annotations

import json
import re
from typing import Any

REDACTED = "[redacted]"

# Keys whose VALUE is secret regardless of what it looks like.
_SECRET_KEY_RE = re.compile(
    r"(?i)(pass(word|wd|phrase)?|token|secret|api[_-]?key|authorization|credential|private[_-]?key)"
)

# Values that are secret regardless of the key they sit under.
_SECRET_VALUE_RES = (
    re.compile(r"sk-[A-Za-z0-9]{16,}"),                  # OpenAI-style
    re.compile(r"gh[pousr]_[A-Za-z0-9]{20,}"),           # GitHub
    re.compile(r"github_pat_[A-Za-z0-9_]{20,}"),
    re.compile(r"AKIA[0-9A-Z]{16}"),                     # AWS access key id
    re.compile(r"(?i)bearer\s+[A-Za-z0-9._\-]{12,}"),
    re.compile(r"xox[baprs]-[A-Za-z0-9-]{10,}"),         # Slack
)

# Argument names that carry a filesystem path, for the searchable paths_json column.
_PATH_KEYS = frozenset({"path", "file", "file_path", "filepath", "target", "directory", "dir", "cwd"})


def redact_text(text: str) -> tuple[str, bool]:
    """Replace secret-shaped runs in free text. Returns (text, was_redacted)."""
    if not isinstance(text, str) or not text:
        return text if isinstance(text, str) else "", False
    hit = False
    out = text
    for rx in _SECRET_VALUE_RES:
        out, n = rx.subn(REDACTED, out)
        if n:
            hit = True
    return out, hit


def _walk(value: Any, key: str | None) -> tuple[Any, bool]:
    if key is not None and isinstance(key, str) and _SECRET_KEY_RE.search(key):
        return REDACTED, True
    if isinstance(value, str):
        return redact_text(value)
    if isinstance(value, dict):
        hit = False
        out: dict[Any, Any] = {}
        for k, v in value.items():
            out[k], h = _walk(v, k if isinstance(k, str) else None)
            hit = hit or h
        return out, hit
    if isinstance(value, list):
        hit = False
        out_list = []
        for item in value:
            red, h = _walk(item, None)
            out_list.append(red)
            hit = hit or h
        return out_list, hit
    return value, False


def redact_arguments(arguments: Any) -> tuple[str, bool]:
    """Redact an argument mapping and serialise it. Returns (json, was_redacted)."""
    try:
        redacted, hit = _walk(arguments, None)
        return json.dumps(redacted, default = str, ensure_ascii = False), hit
    except Exception:
        # Never let redaction failure escape; an unserialisable argument must not
        # cost the caller its tool call.
        return json.dumps({"_unserialisable": True}), True


def extract_paths(arguments: Any) -> str:
    """Best-effort list of path-valued arguments, for search only.

    Deliberately shallow and name-based. A general mechanism would mean teaching
    this module every tool's semantics -- the coupling a registry exists to
    remove. Full arguments are stored regardless, so nothing is lost; only the
    indexed search is approximate.
    """
    found: list[str] = []
    try:
        if isinstance(arguments, dict):
            for k, v in arguments.items():
                if isinstance(k, str) and k.lower() in _PATH_KEYS and isinstance(v, str) and v:
                    found.append(v)
    except Exception:
        pass
    return json.dumps(found)
```

- [ ] **Step 5: Run the tests and watch them pass**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_redaction.py -v -p no:cacheprovider
```
Expected: 7 passed.

- [ ] **Step 6: Prove the over-redaction controls are live**

Temporarily change `_SECRET_VALUE_RES` to include `re.compile(r"[A-Za-z]+")`. Re-run and confirm
`test_ordinary_arguments_are_NOT_redacted` and `test_plain_prose_is_NOT_redacted` FAIL. Restore.
These two are the only thing standing between a useful log and one that redacts everything.

- [ ] **Step 7: Commit**

```bash
git add studio/backend/core/inference/tool_audit/ studio/backend/tests/test_tool_audit_redaction.py
git commit -m "feat(audit): secret redaction for arguments and result text

Key-name matching for values that are secret regardless of shape, plus
pattern matching for values that are secret regardless of key. Walks nested
dicts and lists.

Every positive case has a negative partner in the tests. Over-redaction is the
quieter failure: a log that scrubs ordinary arguments still passes a
'secrets are removed' test while being useless.

Known gap, accepted for v1 and documented in the spec: a secret matching no
pattern under an innocuous key still gets through. Closing it would mean
decrypting the credential store to scan every record.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: The recorder

**Files:**
- Modify: `studio/backend/core/inference/tool_audit/__init__.py`
- Test: `studio/backend/tests/test_tool_audit_recorder.py`

**Interfaces:**
- Consumes: `storage.tool_audit_db` (Task 1), `core.inference.tool_audit.redaction` (Task 2)
- Produces:
  - `around(fn, *args, **kwargs) -> str` — calls `fn`, records the invocation, returns `fn`'s result unchanged
  - `degraded_count() -> int` — how many times recording failed
  - `reset_degraded_for_tests() -> None`
  - `RESULT_CAP_BYTES` — `4096`, the per-side cap

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_tool_audit_recorder.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""The audit recorder wrapping execute_tool.

The invariant that matters most is negative: auditing must never break a tool
call. A test asserting that is the difference between an audit feature and an
outage.
"""

from __future__ import annotations

import pytest

from core.inference import tool_audit
from storage import tool_audit_db


@pytest.fixture(autouse = True)
def _isolated(tmp_path, monkeypatch):
    monkeypatch.setenv("UNSLOTH_STUDIO_HOME", str(tmp_path))
    tool_audit_db.reset_for_tests()
    tool_audit.reset_degraded_for_tests()
    yield


def _fake_tool(name, arguments, **kwargs):
    return "tool output"


def test_successful_call_is_recorded_and_result_passes_through():
    out = tool_audit.around(_fake_tool, "terminal", {"command": "ls"}, session_id = "s1")
    assert out == "tool output"
    entries = tool_audit_db.query_entries()
    assert len(entries) == 1
    assert entries[0]["tool_name"] == "terminal"
    assert entries[0]["outcome"] == "ok"
    assert entries[0]["session_id"] == "s1"


def test_raising_tool_is_recorded_as_error_and_the_exception_propagates():
    def boom(name, arguments, **kwargs):
        raise RuntimeError("kaboom")

    with pytest.raises(RuntimeError, match = "kaboom"):
        tool_audit.around(boom, "python", {"code": "1/0"})
    entry = tool_audit_db.query_entries()[0]
    assert entry["outcome"] == "error"
    assert "kaboom" in entry["error_text"]


def test_a_raising_RECORDER_does_not_break_the_tool_call(monkeypatch):
    """THE critical invariant. If auditing fails, the tool still works."""
    def explode(**kwargs):
        raise OSError("disk full")

    monkeypatch.setattr(tool_audit_db, "record_start", explode)
    out = tool_audit.around(_fake_tool, "terminal", {"command": "ls"})
    assert out == "tool output", "a failed audit write must not cost the caller its result"
    assert tool_audit.degraded_count() >= 1, "and the failure must be counted, not silent"


def test_arguments_are_redacted_before_storage():
    tool_audit.around(_fake_tool, "terminal", {"api_key": "hunter2"})
    entry = tool_audit_db.query_entries()[0]
    assert "hunter2" not in entry["arguments_json"]
    assert entry["redacted"] is True


def test_large_results_are_capped_but_length_is_true():
    big = "x" * 50_000

    def big_tool(name, arguments, **kwargs):
        return big

    tool_audit.around(big_tool, "python", {})
    entry = tool_audit_db.query_entries()[0]
    assert entry["result_bytes"] == 50_000, "the TRUE length must survive truncation"
    assert len(entry["result_head"]) <= tool_audit.RESULT_CAP_BYTES
    assert len(entry["result_tail"]) <= tool_audit.RESULT_CAP_BYTES


def test_disable_sandbox_is_recorded():
    tool_audit.around(_fake_tool, "terminal", {"command": "ls"}, disable_sandbox = True)
    assert tool_audit_db.query_entries()[0]["disable_sandbox"] is True
```

- [ ] **Step 2: Run it and watch it fail**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_recorder.py -v -p no:cacheprovider
```
Expected: `AttributeError: module 'core.inference.tool_audit' has no attribute 'around'`.

- [ ] **Step 3: Implement the recorder**

Replace `studio/backend/core/inference/tool_audit/__init__.py` with:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Record every tool invocation.

Installed over execute_tool by a definition-time shadow at the end of tools.py,
so all three callers -- studio_tool_loop, safetensors_agentic, and llama_cpp
(which this fork does not edit) -- are covered by one hook.

THE INVARIANT: auditing must never break a tool call. Every recording path is
guarded, and the guard is a BARE except: this project has established that
"never raises" needs one, because OverflowError and unhashable types slip past
`except Exception` in practice.

Silence is its own failure, so failures increment a counter that the API
surfaces as "audit degraded". An audit log that quietly stops working leaves
holes with no indication.
"""

from __future__ import annotations

import hashlib
import logging
import time
from typing import Any, Callable

logger = logging.getLogger(__name__)

# Per side. A 50 KB result stores its first and last 4 KB; result_bytes keeps the
# true length so truncation is always detectable.
RESULT_CAP_BYTES = 4096

_degraded = 0


def degraded_count() -> int:
    return _degraded


def reset_degraded_for_tests() -> None:
    global _degraded
    _degraded = 0


def _note_failure(what: str, exc: BaseException) -> None:
    global _degraded
    _degraded += 1
    logger.warning("tool audit %s failed (%s); tool execution unaffected", what, exc)


def _split_result(text: str) -> tuple[str, str, int, str]:
    """(head, tail, true_byte_length, sha256_of_redacted_text)."""
    from core.inference.tool_audit import redaction

    safe, _ = redaction.redact_text(text)
    raw = text.encode("utf-8", errors = "replace")
    # Hash the REDACTED payload, not the original: hashing the original would let
    # a short secret be confirmed by brute force against the stored digest.
    digest = hashlib.sha256(safe.encode("utf-8", errors = "replace")).hexdigest()
    if len(safe) <= RESULT_CAP_BYTES * 2:
        return safe, "", len(raw), digest
    return safe[:RESULT_CAP_BYTES], safe[-RESULT_CAP_BYTES:], len(raw), digest


def around(fn: Callable[..., str], *args: Any, **kwargs: Any) -> str:
    """Call ``fn`` and record the invocation. Returns ``fn``'s result unchanged."""
    row_id = None
    started = time.monotonic()
    try:
        from core.inference.tool_audit import redaction
        from storage import tool_audit_db

        # All three callers pass name and arguments positionally, then **kwargs.
        name = args[0] if args else kwargs.get("name")
        arguments = args[1] if len(args) > 1 else kwargs.get("arguments")
        args_json, redacted = redaction.redact_arguments(arguments)
        row_id = tool_audit_db.record_start(
            tool_name = str(name),
            arguments_json = args_json,
            paths_json = redaction.extract_paths(arguments),
            redacted = redacted,
            session_id = kwargs.get("session_id"),
            thread_id = kwargs.get("thread_id"),
            disable_sandbox = bool(kwargs.get("disable_sandbox", False)),
        )
    except BaseException as exc:  # noqa: BLE001 - never-raises; see module docstring
        _note_failure("start", exc)

    try:
        result = fn(*args, **kwargs)
    except BaseException as exc:
        _finish(row_id, started, outcome = "error", text = "", error = repr(exc))
        raise
    _finish(row_id, started, outcome = "ok", text = result if isinstance(result, str) else str(result))
    return result


def _finish(row_id, started: float, *, outcome: str, text: str, error: str | None = None) -> None:
    if row_id is None:
        return
    try:
        from storage import tool_audit_db

        head, tail, size, digest = _split_result(text)
        tool_audit_db.record_finish(
            row_id,
            outcome = outcome,
            duration_ms = int((time.monotonic() - started) * 1000),
            result_head = head,
            result_tail = tail,
            result_bytes = size,
            result_sha256 = digest,
            error_text = error,
        )
    except BaseException as exc:  # noqa: BLE001 - never-raises; see module docstring
        _note_failure("finish", exc)
```

- [ ] **Step 4: Run the tests and watch them pass**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_recorder.py -v -p no:cacheprovider
```
Expected: 6 passed.

- [ ] **Step 5: Prove the never-raises control is live**

Temporarily change the `except BaseException` on the `record_start` guard to `except ValueError`.
Re-run and confirm `test_a_raising_RECORDER_does_not_break_the_tool_call` FAILS with `OSError: disk
full` escaping. Restore. This test is the difference between an audit feature and an outage.

- [ ] **Step 6: Commit**

```bash
git add studio/backend/core/inference/tool_audit/__init__.py studio/backend/tests/test_tool_audit_recorder.py
git commit -m "feat(audit): the recorder wrapping execute_tool

around(fn, *args, **kwargs) records an invocation and returns fn's result
unchanged. Two-phase: a running row on entry, completed on exit, so a crash
mid-tool leaves visible evidence.

Every recording path is guarded by a BARE except, not except Exception: this
project has established that 'never raises' needs one, because OverflowError and
unhashable types slip past. A failed audit write must never cost the caller its
tool call -- there is a test for exactly that, and it was verified to fail when
the guard is narrowed.

Failures increment a counter surfaced as 'audit degraded'. Silence is its own
failure mode: a log that quietly stops working leaves holes with no indication.

The result hash covers the REDACTED payload. Hashing the original would let a
short secret be confirmed by brute force against the stored digest.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: The seam

**Files:**
- Modify: `studio/backend/core/inference/tools.py` (append after line 10189, before `_opt_int`)
- Test: `studio/backend/tests/test_tool_audit_seam.py`

**Interfaces:**
- Consumes: `core.inference.tool_audit.around` (Task 3)
- Produces: `core.inference.tools._execute_tool_unaudited` — the original function, for tests and any future caller that must bypass auditing

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_tool_audit_seam.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""The audit hook must live INSIDE tools.py, wrapping execute_tool.

execute_tool has three callers and one of them is core/inference/llama_cpp.py,
which this fork does not edit. A hook at the call sites would therefore miss a
whole agentic path. These tests exist to stop a later 'simplification' moving it
there, and to stop the functools.wraps disappearing.
"""

from __future__ import annotations

import ast
import inspect
from pathlib import Path

from core.inference import tools

_TOOLS_PY = Path(inspect.getfile(tools))


def test_execute_tool_is_wrapped_by_the_audit_shadow():
    assert hasattr(tools, "_execute_tool_unaudited"), (
        "the definition-time shadow is gone; execute_tool is no longer audited"
    )
    assert tools.execute_tool is not tools._execute_tool_unaudited


def test_the_wrapper_preserves_the_original_signature():
    """functools.wraps here is load-bearing, not decoration.

    studio_tool_loop.py gates kwarg forwarding on accepts_kwarg(execute_tool, ...),
    which uses inspect.signature -- and inspect.signature follows __wrapped__. A
    bare wrapper would report (*args, **kwargs) and silently disable
    conversation-branch and budget forwarding.
    """
    params = inspect.signature(tools.execute_tool).parameters
    for expected in ("name", "arguments", "session_id", "thread_id",
                     "disable_sandbox", "conversation_branch", "conversation_budget_tokens"):
        assert expected in params, f"{expected} vanished from the public signature"


def test_accepts_kwarg_still_sees_the_forwarded_kwargs():
    """The real consumer, not a proxy for it."""
    from core.inference.tool_stream_exec import accepts_kwarg

    assert accepts_kwarg(tools.execute_tool, "conversation_branch") is True
    assert accepts_kwarg(tools.execute_tool, "conversation_budget_tokens") is True


def test_the_hook_is_in_tools_py_and_not_at_the_call_sites():
    """AST check: the shadow must be a module-level assignment in tools.py."""
    tree = ast.parse(_TOOLS_PY.read_text(encoding = "utf-8"))
    names = {
        t.id
        for node in tree.body
        if isinstance(node, ast.Assign)
        for t in node.targets
        if isinstance(t, ast.Name)
    }
    assert "_execute_tool_unaudited" in names, (
        "the audit shadow is not a module-level assignment in tools.py -- if the "
        "hook moved to the call sites, the llama_cpp.py path is no longer audited"
    )


def test_the_seam_stays_additive():
    """tools.py must never lose a line to this fork."""
    import subprocess

    repo = _TOOLS_PY.parents[4]
    base = subprocess.run(
        ["git", "merge-base", "origin/main", "HEAD"],
        cwd = repo, capture_output = True, text = True, check = True,
    ).stdout.strip()
    stat = subprocess.run(
        ["git", "diff", "--numstat", base, "--", "studio/backend/core/inference/tools.py"],
        cwd = repo, capture_output = True, text = True, check = True,
    ).stdout.split()
    assert stat, "no diff recorded for tools.py"
    insertions, deletions = int(stat[0]), int(stat[1])
    assert deletions == 0, f"tools.py lost {deletions} line(s) to the fork; the seam must be additive"
    assert insertions <= 70, f"seam grew to {insertions} insertions; budget is ~55"
```

- [ ] **Step 2: Run it and watch it fail**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_seam.py -v -p no:cacheprovider
```
Expected: 4 failures (`_execute_tool_unaudited` missing); `test_the_seam_stays_additive` passes already.

- [ ] **Step 3: Add the shadow**

In `studio/backend/core/inference/tools.py`, find the end of `execute_tool` — the line
`    return f"Unknown tool: {name}"` at **line 10189** — and insert the following immediately after
it, before the blank lines preceding `def _opt_int`. **Do not modify or re-indent any existing
line.**

```python


# --- fork: tool audit -------------------------------------------------------
# Shadowing here, rather than wrapping execute_tool's body in try/finally, is
# what keeps this seam additive: the body is ~160 lines with many returns, so a
# try: would re-indent all of them and every line would read as modified.
#
# This is NOT an external monkeypatch. It runs inside this module's own body,
# before the module finishes executing and therefore before any importer can
# bind the name -- so all three callers (studio_tool_loop, safetensors_agentic,
# and llama_cpp, which this fork does not edit) get the audited version
# deterministically rather than by import order.
#
# functools.wraps is load-bearing: studio_tool_loop gates kwarg forwarding on
# accepts_kwarg(execute_tool, ...), which uses inspect.signature, and that
# follows __wrapped__. Without it the wrapper would report (*args, **kwargs) and
# silently disable conversation-branch and budget forwarding.
_execute_tool_unaudited = execute_tool


@functools.wraps(_execute_tool_unaudited)
def execute_tool(*args, **kwargs):  # noqa: F811 - deliberate shadow, see above
    from core.inference import tool_audit

    return tool_audit.around(_execute_tool_unaudited, *args, **kwargs)
```

- [ ] **Step 4: Run the tests and watch them pass**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_seam.py -v -p no:cacheprovider
```
Expected: 5 passed.

- [ ] **Step 5: Confirm nothing else broke**

Run the fork's other tool-facing suites — these exercise `execute_tool` directly and would catch a
broken wrapper:

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_assist_code_registration.py tests/test_assist_vision_budget.py tests/test_tool_audit_recorder.py -v -p no:cacheprovider
```
Expected: all pass. If any fails with a `TypeError` about keyword arguments, `functools.wraps` is
missing or misplaced.

- [ ] **Step 6: Prove the wraps control is live**

Temporarily delete the `@functools.wraps(...)` line. Re-run
`tests/test_tool_audit_seam.py::test_accepts_kwarg_still_sees_the_forwarded_kwargs` and confirm it
FAILS. Restore it. This is the failure no existing test would have caught.

- [ ] **Step 7: Commit**

```bash
git add studio/backend/core/inference/tools.py studio/backend/tests/test_tool_audit_seam.py
git commit -m "feat(audit): install the recorder over execute_tool

A definition-time shadow appended after execute_tool ends, not a try/finally
around its body: the body is ~160 lines with many returns, so wrapping it would
re-indent ~145 lines and every one would read as modified -- the largest
conflict surface this fork could own, in the most contended upstream file.
Zero deletions, zero re-indentation.

Not an external monkeypatch either. This runs inside tools.py's own module body,
before any importer can bind the name, so all three callers get the audited
version deterministically rather than by import order. That matters because one
caller is core/inference/llama_cpp.py, which this fork does not edit -- a
call-site hook could never have covered it.

functools.wraps is load-bearing: studio_tool_loop gates kwarg forwarding on
accepts_kwarg(execute_tool, ...) via inspect.signature, which follows
__wrapped__. Without it the wrapper reports (*args, **kwargs) and silently
disables conversation-branch and budget forwarding. No existing test covered
that; test_accepts_kwarg_still_sees_the_forwarded_kwargs now does, and was
verified to fail when the decorator is removed.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 5: Read-only API

**Files:**
- Create: `studio/backend/routes/tool_audit.py`
- Modify: `studio/backend/main.py` (+2 lines)
- Test: `studio/backend/tests/test_tool_audit_routes.py`

**Interfaces:**
- Consumes: `storage.tool_audit_db` (Task 1), `core.inference.tool_audit.degraded_count` (Task 3)
- Produces: router mounted at `/api/tool-audit` with
  `GET /entries?limit&offset&tool_name&session_id`, `GET /entries/{entry_id}`, `GET /status`

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_tool_audit_routes.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Read-only API over the tool audit log."""

from __future__ import annotations

import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from routes.tool_audit import router
from storage import tool_audit_db


@pytest.fixture()
def client(tmp_path, monkeypatch):
    monkeypatch.setenv("UNSLOTH_STUDIO_HOME", str(tmp_path))
    tool_audit_db.reset_for_tests()
    app = FastAPI()
    app.include_router(router, prefix = "/api/tool-audit")
    return TestClient(app)


def _seed(tool_name = "terminal", session_id = "s1"):
    return tool_audit_db.record_start(
        tool_name = tool_name,
        arguments_json = '{"command": "ls"}',
        paths_json = "[]",
        redacted = False,
        session_id = session_id,
        thread_id = None,
        disable_sandbox = False,
    )


def test_entries_returns_rows_newest_first(client):
    _seed(tool_name = "terminal")
    _seed(tool_name = "python")
    r = client.get("/api/tool-audit/entries")
    assert r.status_code == 200
    body = r.json()
    assert [e["tool_name"] for e in body["entries"]] == ["python", "terminal"]


def test_entries_filters_by_tool(client):
    _seed(tool_name = "terminal")
    _seed(tool_name = "python")
    r = client.get("/api/tool-audit/entries", params = {"tool_name": "python"})
    assert [e["tool_name"] for e in r.json()["entries"]] == ["python"]


def test_single_entry_and_404(client):
    row_id = _seed()
    assert client.get(f"/api/tool-audit/entries/{row_id}").status_code == 200
    assert client.get("/api/tool-audit/entries/999999").status_code == 404


def test_status_reports_degradation(client):
    from core.inference import tool_audit

    tool_audit.reset_degraded_for_tests()
    body = client.get("/api/tool-audit/status").json()
    assert body["degraded"] is False
    assert body["failed_writes"] == 0
```

- [ ] **Step 2: Run it and watch it fail**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_routes.py -v -p no:cacheprovider
```
Expected: `ModuleNotFoundError: No module named 'routes.tool_audit'`.

- [ ] **Step 3: Implement the router**

Create `studio/backend/routes/tool_audit.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Read-only API over the tool audit log.

Read-only on purpose. Records are evidence; an endpoint that edited or deleted
them would undermine the point of keeping them. Pruning is a retention policy
inside the storage layer, and it records itself.
"""

from __future__ import annotations

from typing import Optional

from fastapi import APIRouter, HTTPException, Query
from pydantic import BaseModel

from core.inference import tool_audit
from storage import tool_audit_db

router = APIRouter()


class AuditStatus(BaseModel):
    degraded: bool
    failed_writes: int


@router.get("/entries")
def list_entries(
    limit: int = Query(100, ge = 1, le = 1000),
    offset: int = Query(0, ge = 0),
    tool_name: Optional[str] = None,
    session_id: Optional[str] = None,
) -> dict:
    entries = tool_audit_db.query_entries(
        limit = limit,
        offset = offset,
        tool_name = tool_name,
        session_id = session_id,
    )
    return {"entries": entries, "count": len(entries)}


@router.get("/entries/{entry_id}")
def get_entry(entry_id: int) -> dict:
    entry = tool_audit_db.get_entry(entry_id)
    if entry is None:
        raise HTTPException(status_code = 404, detail = "No such audit entry")
    return entry


@router.get("/status", response_model = AuditStatus)
def status() -> AuditStatus:
    failed = tool_audit.degraded_count()
    return AuditStatus(degraded = failed > 0, failed_writes = failed)
```

- [ ] **Step 4: Run the tests and watch them pass**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_routes.py -v -p no:cacheprovider
```
Expected: 4 passed.

- [ ] **Step 5: Register the router**

In `studio/backend/main.py`, add the import beside the existing fork import at line 332:

```python
from routes.tool_audit import router as tool_audit_router
```

and the mount beside the existing fork mount at line 1388:

```python
app.include_router(tool_audit_router, prefix = "/api/tool-audit", tags = ["tool-audit"])
```

Then confirm the file is still additive-only:

```bash
git diff --numstat $(git merge-base origin/main HEAD) -- studio/backend/main.py
```
Expected: `4       0       studio/backend/main.py`

- [ ] **Step 6: Commit**

```bash
git add studio/backend/routes/tool_audit.py studio/backend/main.py studio/backend/tests/test_tool_audit_routes.py
git commit -m "feat(audit): read-only API over the tool audit log

GET /api/tool-audit/entries with tool and session filters, /entries/{id}, and
/status reporting whether any audit write has failed.

Read-only on purpose: records are evidence, and an endpoint that edited or
deleted them would undermine keeping them. Retention lives in the storage layer
and records itself.

main.py gains an import and an include_router, mirroring what the draft-model
sub-project already added there; the file stays additive-only against upstream
at 4 insertions / 0 deletions.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 6: Activity panel

**Files:**
- Create: `studio/frontend/src/features/tool-audit/tool-audit-page.tsx`
- Create: `studio/frontend/src/features/tool-audit/index.ts`
- Create: `studio/frontend/src/app/routes/tool-audit.tsx`
- Modify: `studio/frontend/src/app/router.tsx` (+2 lines)
- Modify: `studio/frontend/src/i18n/locales/en.ts` (+ `toolAudit` section)
- Modify: the 11 overlay locales

**Interfaces:**
- Consumes: `GET /api/tool-audit/entries`, `/entries/{id}`, `/status` (Task 5)
- Produces: route at `/tool-audit`

- [ ] **Step 1: Add the English strings**

In `studio/frontend/src/i18n/locales/en.ts`, add a new top-level section at the end of the object,
beside the existing `chat:` section (line 2076):

```typescript
  toolAudit: {
    title: "Tool activity",
    empty: "No tool calls recorded yet.",
    degraded: "Some audit records failed to write ({count}). The log is incomplete.",
    filterTool: "Filter by tool",
    filterAll: "All tools",
    columnTime: "Time",
    columnTool: "Tool",
    columnOutcome: "Outcome",
    columnDuration: "Duration",
    outcomeRunning: "running",
    outcomeOk: "ok",
    outcomeError: "error",
    redactedBadge: "redacted",
    sandboxBypassBadge: "sandbox bypassed",
    argumentsHeading: "Arguments",
    resultHeading: "Result",
    errorHeading: "Error",
    truncatedNote: "Showing the first and last 4 KB of {bytes} bytes.",
    refresh: "Refresh",
  },
```

- [ ] **Step 2: Run the i18n gate and watch it fail**

From `studio/frontend`:
```
npm run i18n:check:strict
```
Expected: FAIL — 11 overlays are missing the `toolAudit` keys.

- [ ] **Step 3: Translate into the 11 overlays**

Add the same `toolAudit` section to each of `ar.ts`, `de.ts`, `es.ts`, `fr.ts`, `hi.ts`, `it.ts`,
`ja.ts`, `ko.ts`, `pt-br.ts`, `ru.ts`, `zh-CN.ts`, **translated**, not copied from English.

This repo's convention is unambiguous: of `ar.ts`'s 1356 string values only 69 are pure ASCII, and
every one of those is a proper noun, acronym or interpolation placeholder — never a sentence.
Pasting English would pass `i18n:check:strict`, which checks key presence and not value
distinctness, and ship English text to every non-English user. That exact mistake already happened
once here and cost a follow-up commit across 11 files.

Keep `{count}` and `{bytes}` placeholders intact in every locale.

- [ ] **Step 4: Run the i18n gate and watch it pass**

```
npm run i18n:check:strict
```
Expected: "All locale overlays pass parity."

- [ ] **Step 5: Create the page component**

Create `studio/frontend/src/features/tool-audit/tool-audit-page.tsx`:

```tsx
// SPDX-License-Identifier: AGPL-3.0-only
// Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

import { type ReactElement, useCallback, useEffect, useState } from "react";
import { useT } from "@/i18n";
import { authFetch } from "@/features/auth";

type AuditEntry = {
  id: number;
  ts: number;
  duration_ms: number | null;
  tool_name: string;
  arguments_json: string;
  outcome: string;
  redacted: boolean;
  disable_sandbox: boolean;
  result_head: string | null;
  result_tail: string | null;
  result_bytes: number | null;
  error_text: string | null;
};

export function ToolAuditPage(): ReactElement {
  const t = useT();
  const [entries, setEntries] = useState<AuditEntry[]>([]);
  const [expanded, setExpanded] = useState<number | null>(null);
  const [failedWrites, setFailedWrites] = useState(0);
  const [toolFilter, setToolFilter] = useState<string>("");

  const load = useCallback(async () => {
    const qs = toolFilter ? `?tool_name=${encodeURIComponent(toolFilter)}` : "";
    const [listRes, statusRes] = await Promise.all([
      authFetch(`/api/tool-audit/entries${qs}`),
      authFetch("/api/tool-audit/status"),
    ]);
    if (listRes.ok) setEntries((await listRes.json()).entries ?? []);
    if (statusRes.ok) setFailedWrites((await statusRes.json()).failed_writes ?? 0);
  }, [toolFilter]);

  useEffect(() => {
    void load();
  }, [load]);

  const tools = Array.from(new Set(entries.map((e) => e.tool_name))).sort();

  return (
    <div className="p-4 space-y-4">
      <div className="flex items-center gap-3">
        <h1 className="text-lg font-semibold">{t("toolAudit.title")}</h1>
        <select
          className="border rounded px-2 py-1 text-sm"
          value={toolFilter}
          onChange={(e) => setToolFilter(e.target.value)}
          aria-label={t("toolAudit.filterTool")}
        >
          <option value="">{t("toolAudit.filterAll")}</option>
          {tools.map((name) => (
            <option key={name} value={name}>{name}</option>
          ))}
        </select>
        <button className="border rounded px-2 py-1 text-sm" onClick={() => void load()}>
          {t("toolAudit.refresh")}
        </button>
      </div>

      {failedWrites > 0 && (
        <div role="alert" className="border border-amber-500 rounded p-2 text-sm">
          {t("toolAudit.degraded", { count: failedWrites })}
        </div>
      )}

      {entries.length === 0 ? (
        <p className="text-sm opacity-70">{t("toolAudit.empty")}</p>
      ) : (
        <table className="w-full text-sm">
          <thead>
            <tr className="text-left">
              <th>{t("toolAudit.columnTime")}</th>
              <th>{t("toolAudit.columnTool")}</th>
              <th>{t("toolAudit.columnOutcome")}</th>
              <th>{t("toolAudit.columnDuration")}</th>
            </tr>
          </thead>
          <tbody>
            {entries.map((e) => (
              <tr key={e.id} className="border-t align-top">
                <td>
                  <button onClick={() => setExpanded(expanded === e.id ? null : e.id)}>
                    {new Date(e.ts * 1000).toLocaleString()}
                  </button>
                  {expanded === e.id && (
                    <div className="my-2 space-y-2">
                      <div>
                        <strong>{t("toolAudit.argumentsHeading")}</strong>
                        <pre className="overflow-x-auto text-xs">{e.arguments_json}</pre>
                      </div>
                      {e.error_text ? (
                        <div>
                          <strong>{t("toolAudit.errorHeading")}</strong>
                          <pre className="overflow-x-auto text-xs">{e.error_text}</pre>
                        </div>
                      ) : (
                        <div>
                          <strong>{t("toolAudit.resultHeading")}</strong>
                          <pre className="overflow-x-auto text-xs">
                            {(e.result_head ?? "") + (e.result_tail ? `\n...\n${e.result_tail}` : "")}
                          </pre>
                          {e.result_tail ? (
                            <p className="text-xs opacity-70">
                              {t("toolAudit.truncatedNote", { bytes: e.result_bytes ?? 0 })}
                            </p>
                          ) : null}
                        </div>
                      )}
                    </div>
                  )}
                </td>
                <td>
                  {e.tool_name}
                  {e.redacted && <span className="ml-1 text-xs">[{t("toolAudit.redactedBadge")}]</span>}
                  {e.disable_sandbox && (
                    <span className="ml-1 text-xs">[{t("toolAudit.sandboxBypassBadge")}]</span>
                  )}
                </td>
                <td>{e.outcome}</td>
                <td>{e.duration_ms == null ? "-" : `${e.duration_ms} ms`}</td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
    </div>
  );
}
```

Create `studio/frontend/src/features/tool-audit/index.ts`:

```typescript
// SPDX-License-Identifier: AGPL-3.0-only
// Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

export { ToolAuditPage } from "./tool-audit-page";
```

- [ ] **Step 6: Add the route**

Create `studio/frontend/src/app/routes/tool-audit.tsx`, mirroring `app/routes/api.tsx`:

```tsx
// SPDX-License-Identifier: AGPL-3.0-only
// Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

import { createRoute, lazyRouteComponent } from "@tanstack/react-router";
import { requireAuth } from "../auth-guards";
import { Route as rootRoute } from "./__root";

const ToolAuditPage = lazyRouteComponent(
  () => import("@/features/tool-audit"),
  "ToolAuditPage",
);

export const Route = createRoute({
  getParentRoute: () => rootRoute,
  path: "/tool-audit",
  staticData: { title: "Tool activity" },
  beforeLoad: () => requireAuth(),
  component: ToolAuditPage,
});
```

In `studio/frontend/src/app/router.tsx`, add the import beside the others (near line 9):

```typescript
import { Route as toolAuditRoute } from "./routes/tool-audit";
```

and add `toolAuditRoute,` to the `routeTree` children array (near line 40).

- [ ] **Step 7: Run the frontend gates**

From `studio/frontend`:
```
npx tsc -b --force --noEmit
npm run i18n:check:strict
npm run build
```
Expected: all three succeed. Use `--force`; stale incremental state has produced phantom
diagnostics in this repo five times.

- [ ] **Step 8: Verify the panel against real data**

Start the backend, make any tool call in a chat, then open `/tool-audit`. Confirm the call appears
with its arguments, and that a `terminal` call run with Bypass Permissions shows the
`sandbox bypassed` badge. Reading rows out of SQLite is **not** a substitute — the point is that the
panel renders what the API returns.

- [ ] **Step 9: Commit**

```bash
git add studio/frontend/src/features/tool-audit/ studio/frontend/src/app/routes/tool-audit.tsx studio/frontend/src/app/router.tsx studio/frontend/src/i18n/locales/
git commit -m "feat(audit): tool activity panel

A read-only panel at /tool-audit: filter by tool, expand a row for full
arguments, result head/tail and error text. Surfaces 'audit degraded' when any
write has failed, so an incomplete log announces itself.

Badges the two facts that are easy to miss in a wall of rows: whether anything
was redacted, and whether the call ran with Bypass Permissions -- no sandbox, no
blocklist, no rlimits.

The 11 locale overlays are TRANSLATED, not copied from English.
i18n:check:strict verifies key presence, not value distinctness, so pasting
English passes the gate and still ships English to every non-English user --
which already happened once in this fork and cost a follow-up across 11 files.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 7: Retention wiring

**Files:**
- Modify: `studio/backend/routes/tool_audit.py`
- Test: `studio/backend/tests/test_tool_audit_db.py` (extend)

**Interfaces:**
- Consumes: `tool_audit_db.prune` (Task 1)
- Produces: `maybe_prune()` in `storage/tool_audit_db.py` — prunes at most once per process per hour

- [ ] **Step 1: Write the failing test**

Append to `studio/backend/tests/test_tool_audit_db.py`:

```python
def test_maybe_prune_runs_once_then_backs_off(monkeypatch):
    """Pruning on every write would be wasteful; once per hour is enough."""
    calls = []
    monkeypatch.setattr(tool_audit_db, "prune", lambda **kw: calls.append(kw) or 0)
    tool_audit_db.reset_prune_clock_for_tests()
    tool_audit_db.maybe_prune()
    tool_audit_db.maybe_prune()
    assert len(calls) == 1, "maybe_prune must not prune twice inside the interval"
```

- [ ] **Step 2: Run it and watch it fail**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_db.py::test_maybe_prune_runs_once_then_backs_off -v -p no:cacheprovider
```
Expected: `AttributeError: module 'storage.tool_audit_db' has no attribute 'reset_prune_clock_for_tests'`.

- [ ] **Step 3: Implement**

Append to `studio/backend/storage/tool_audit_db.py`:

```python
_PRUNE_INTERVAL_SECONDS = 3600.0
_last_prune = 0.0


def reset_prune_clock_for_tests() -> None:
    global _last_prune
    _last_prune = 0.0


def maybe_prune(*, max_age_days: int = 365, max_rows: int = 250_000) -> None:
    """Prune at most once an hour per process.

    Called from the read API rather than the write path: pruning is maintenance,
    and a tool call should never wait on it.
    """
    global _last_prune
    now = time.time()
    if now - _last_prune < _PRUNE_INTERVAL_SECONDS:
        return
    _last_prune = now
    try:
        prune(max_age_days = max_age_days, max_rows = max_rows)
    except BaseException:  # noqa: BLE001 - retention must never break a read
        pass
```

- [ ] **Step 4: Call it from the list endpoint**

In `studio/backend/routes/tool_audit.py`, add as the first line of `list_entries`'s body:

```python
    tool_audit_db.maybe_prune()
```

- [ ] **Step 5: Run the tests and watch them pass**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_db.py tests/test_tool_audit_routes.py -v -p no:cacheprovider
```
Expected: all pass.

- [ ] **Step 6: Full-suite sanity for this feature**

```
C:/Users/Admin/.unsloth/studio/unsloth_studio/Scripts/python.exe -m pytest tests/test_tool_audit_db.py tests/test_tool_audit_redaction.py tests/test_tool_audit_recorder.py tests/test_tool_audit_seam.py tests/test_tool_audit_routes.py -v -p no:cacheprovider
```
Expected: all pass. **Do not run the whole backend suite** — upstream fixtures fabricate GGUF files
up to 40 GB.

- [ ] **Step 7: Commit**

```bash
git add studio/backend/storage/tool_audit_db.py studio/backend/routes/tool_audit.py studio/backend/tests/test_tool_audit_db.py
git commit -m "feat(audit): retention, pruned at most hourly from the read path

maybe_prune() runs at most once an hour per process and is called from the list
endpoint rather than the write path: pruning is maintenance, and a tool call
should never wait on it.

Guarded like every other audit path -- retention failing must not break a read.

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Self-Review

**Spec coverage.** Every section of the design maps to a task: architecture and seam → Task 4;
schema → Task 1; redaction → Task 2; retention → Tasks 1 and 7; error handling → Task 3; UI →
Task 6; API → Task 5. The dropped `approval_id`/`approved`/`permission_mode` columns are absent from
the schema, matching the spec's correction. `disable_sandbox` is recorded (Tasks 1, 3) and surfaced
(Task 6).

**Placeholders.** None. Every code step carries the actual code.

**Type consistency.** `record_start` / `record_finish` / `query_entries` / `get_entry` / `prune` /
`maybe_prune` keep identical signatures across Tasks 1, 3, 5 and 7. `around(fn, *args, **kwargs)`
is named identically in Tasks 3 and 4. `RESULT_CAP_BYTES` is defined in Task 3 and used in its own
test. The `AuditEntry` TypeScript type matches the columns Task 1 creates.

**Known ordering constraint.** Task 4 must follow Task 3 — the shadow imports `tool_audit.around`.
Task 6 must follow Task 5. Tasks 1 and 2 are independent and may run in either order.
