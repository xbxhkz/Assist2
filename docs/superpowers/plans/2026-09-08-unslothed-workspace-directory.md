# Unslothed User-Chosen Workspace Directory Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let the user choose the directory the AI's tools operate in — globally, and per project — with warnings that state what each choice exposes.

**Architecture:** `_get_workdir(session_id)` is the single resolution point every tool already flows through. The resolution order becomes project `rootPath` → global default → per-session sandbox. The per-project half unblocks a `root_path` column that already exists and is already read; the global half adds one small additive hunk to the fork's merge seam.

**Tech Stack:** Python 3.12 / FastAPI / SQLite / pytest (backend); React + TypeScript / Vite (frontend).

**Spec:** `docs/superpowers/specs/2026-09-08-unslothed-workspace-directory-design.md`

## Global Constraints

- **Merge seam.** The ONLY permitted edit to `studio/backend/core/inference/tools.py` is a single additive hunk of ~4 lines in `_get_workdir` (Task 3). That file currently holds 17 insertions across 3 hunks with **zero deletions**; it must still have zero deletions when this plan is done. Do NOT edit `routes/inference.py`, `core/inference/llama_cpp.py`, `pyproject.toml`, or `studio/backend/main.py`.
- **Warn only, never refuse.** No path is ever blocked. The classifier is advisory. A guard that returns "refuse" is a defect against this spec.
- **Opt-in by construction.** With no project root and no global default, `_get_workdir` must return byte-identically what it returns today. This is the invariant protecting every existing chat.
- **NEVER run the full backend test suite.** Upstream fixtures fabricate GGUF files up to 40 GB and have filled this machine's disk once. Run only the named test files.
- Python style: spaces around `=` in keyword arguments (`foo(bar = 1)`).
- SPDX header on every new file:
  ```
  # SPDX-License-Identifier: AGPL-3.0-only
  # Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0
  ```
- Every guard gets a negative control **proven to fail** before its fix. Eight controls in this project turned out inert; an unproven control is worthless.

## File Structure

| File | Responsibility |
|---|---|
| `studio/backend/utils/workspace_root.py` | **New.** Warning classification + the global-default accessor. Pure logic plus two DB calls; no HTTP. |
| `studio/backend/storage/studio_db.py` | **Modify.** Honour a caller-supplied project `rootPath` on create; allow `rootPath` in the patch map. |
| `studio/backend/core/inference/tools.py` | **Modify, ~4 lines.** The global fallback in `_get_workdir`. The entire seam cost. |
| `studio/backend/routes/settings.py` | **Modify.** `GET`/`PUT /workspace-root` and `POST /workspace-root/preview`. |
| `studio/backend/routes/chat_history.py` | **Modify.** `ChatProjectPatch` gains `rootPath`. |
| `studio/backend/tests/test_workspace_root.py` | **New.** Classifier + resolver tests and their controls. |
| `studio/backend/tests/test_workspace_root_routes.py` | **New.** HTTP contract. |
| `studio/frontend/src/features/settings/workspace-root-setting.tsx` | **New.** The global setting UI. |

---

### Task 1: Warning classification

**Files:**
- Create: `studio/backend/utils/workspace_root.py`
- Test: `studio/backend/tests/test_workspace_root.py`

**Interfaces:**
- Produces: `classify_root(path: str) -> list[RootWarning]`, where `RootWarning` is a dataclass with `code: str` and `message: str`. Codes: `WARN_STUDIO_DATA`, `WARN_INSTALL_DIR`, `WARN_SYSTEM_DIR`, `WARN_BROAD`. An unremarkable path returns `[]`.

**Why warnings and not blocks:** the owner chose warn-only. A classifier that refuses is a spec violation. Each message must name what is at stake — under warn-only, a warning that does not say what it is warning about is decoration.

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_workspace_root.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""Advisory classification of a user-chosen workspace root.

The policy is warn-only: nothing here may refuse a path. These tests pin both
halves of that -- that the dangerous cases DO warn, and that warning is all
they do.
"""

from __future__ import annotations

import os
import sys
from pathlib import Path

import pytest

_BACKEND_DIR = str(Path(__file__).resolve().parent.parent)
if _BACKEND_DIR not in sys.path:
    sys.path.insert(0, _BACKEND_DIR)

from utils.workspace_root import (
    WARN_BROAD,
    WARN_INSTALL_DIR,
    WARN_STUDIO_DATA,
    WARN_SYSTEM_DIR,
    classify_root,
)


def _codes(path):
    return {w.code for w in classify_root(str(path))}


class TestDangerousRoots:
    def test_studio_data_dir_warns_about_auth_and_models(self, tmp_path, monkeypatch):
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_studio_root", lambda: str(tmp_path / "studio"))
        warnings = classify_root(str(tmp_path / "studio"))
        assert WARN_STUDIO_DATA in {w.code for w in warnings}
        msg = " ".join(w.message for w in warnings).lower()
        assert "auth" in msg, "the warning must say WHAT is at stake, not just that there is risk"

    def test_a_subdirectory_of_the_studio_data_dir_also_warns(self, tmp_path, monkeypatch):
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_studio_root", lambda: str(tmp_path / "studio"))
        assert WARN_STUDIO_DATA in _codes(tmp_path / "studio" / "cache")

    def test_the_install_directory_warns(self, tmp_path, monkeypatch):
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_install_root", lambda: str(tmp_path / "app"))
        assert WARN_INSTALL_DIR in _codes(tmp_path / "app")

    def test_a_filesystem_root_warns_as_a_system_directory(self):
        root = os.path.abspath(os.sep)
        assert WARN_SYSTEM_DIR in _codes(root)


class TestBroadRoots:
    def test_the_home_directory_warns_as_broad(self, tmp_path, monkeypatch):
        monkeypatch.setenv("USERPROFILE", str(tmp_path / "me"))
        monkeypatch.setenv("HOME", str(tmp_path / "me"))
        assert WARN_BROAD in _codes(tmp_path / "me")


class TestOrdinaryRoots:
    def test_an_ordinary_project_folder_warns_about_nothing(self, tmp_path, monkeypatch):
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_studio_root", lambda: str(tmp_path / "studio"))
        monkeypatch.setattr(mod, "_install_root", lambda: str(tmp_path / "app"))
        monkeypatch.setenv("USERPROFILE", str(tmp_path / "me"))
        monkeypatch.setenv("HOME", str(tmp_path / "me"))
        assert classify_root(str(tmp_path / "me" / "code" / "my-project")) == []


class TestWarnOnlyIsStructural:
    def test_classify_never_refuses(self, tmp_path, monkeypatch):
        """Every classification returns warnings. There is no rejection channel
        at all -- the policy is enforced by the return type, not by discipline."""
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_studio_root", lambda: str(tmp_path / "studio"))
        for candidate in (os.path.abspath(os.sep), str(tmp_path / "studio"), str(tmp_path / "x")):
            result = classify_root(candidate)
            assert isinstance(result, list)
            assert all(hasattr(w, "code") and hasattr(w, "message") for w in result)

    # --- negative controls ------------------------------------------------
    # Each must FAIL if its own rule is deleted, and must NOT fire on a
    # neighbour's path. Verified in Step 2.

    def test_control_studio_rule_does_not_fire_on_a_sibling_named_similarly(
        self, tmp_path, monkeypatch
    ):
        """A string-prefix check would match `studio-notes` against `studio`."""
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_studio_root", lambda: str(tmp_path / "studio"))
        assert WARN_STUDIO_DATA not in _codes(tmp_path / "studio-notes")

    def test_control_broad_does_not_fire_on_a_folder_inside_home(
        self, tmp_path, monkeypatch
    ):
        """Only home ITSELF is broad. If a nested folder warns, the rule is
        matching a prefix rather than the directory."""
        monkeypatch.setenv("USERPROFILE", str(tmp_path / "me"))
        monkeypatch.setenv("HOME", str(tmp_path / "me"))
        assert WARN_BROAD not in _codes(tmp_path / "me" / "code")
```

- [ ] **Step 2: Run the tests and confirm they fail**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v
```

Expected: `ModuleNotFoundError: No module named 'utils.workspace_root'`.

**Prove the two controls after Step 3 passes.** Replace `_is_within` with the naive version below, re-run, and confirm `test_control_studio_rule_does_not_fire_on_a_sibling_named_similarly` FAILS. Then revert. If it passes, the control is inert — say so.

```python
# NAIVE VERSION — for the control check only, do not keep
def _is_within(path, root):
    return os.path.realpath(path).startswith(os.path.realpath(root))
```

- [ ] **Step 3: Write the implementation**

Create `studio/backend/utils/workspace_root.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""The directory the AI's tools work in, when the user has chosen one.

Studio resolves a chat's working directory through core/inference/tools.py's
_get_workdir. This module supplies the two inputs that can override the
per-session sandbox -- a global default, and (via storage/studio_db.py) a
per-project root -- plus the advisory classification the UI shows before a
choice is committed.

The policy is WARN ONLY. Nothing here refuses a path, and there is no rejection
channel in the return type to refuse through. That is deliberate: the owner
chose full read/write access to any directory, and a classifier that quietly
declined would be a different feature wearing this one's name.

The consequence is worth stating where someone changing this file will read it:
whatever root is chosen becomes what upstream's `terminal` tool runs inside.
The ten tools this fork adds confine themselves to the workdir; `terminal` does
not confine, it inhabits. The warnings below exist because that is the whole
exposure.
"""

from __future__ import annotations

import os
from dataclasses import dataclass

WARN_STUDIO_DATA = "studio_data"
WARN_INSTALL_DIR = "install_dir"
WARN_SYSTEM_DIR = "system_dir"
WARN_BROAD = "broad"


@dataclass(frozen = True)
class RootWarning:
    code: str
    message: str


def _real(path: str) -> str:
    """Fully resolved, so a symlink or `..` cannot dodge a rule by spelling."""
    return os.path.realpath(os.path.expanduser(str(path)))


def _is_within(path: str, root: str) -> bool:
    """Whether ``path`` is ``root`` or sits inside it.

    Compared as path components, never as a string prefix: `studio-notes`
    starts with `studio` and is a different directory.
    """
    p, r = _real(path), _real(root)
    if p == r:
        return True
    return p.startswith(r.rstrip(os.sep) + os.sep)


def _studio_root() -> str:
    """Studio's own data directory. Patched in tests."""
    from utils.paths import studio_root
    return str(studio_root())


def _install_root() -> str:
    """Where the app itself lives. Frozen, that is the directory holding the
    exe; from source it is the repo checkout. Patched in tests."""
    import sys
    if getattr(sys, "frozen", False):
        return os.path.dirname(os.path.abspath(sys.executable))
    return os.path.dirname(os.path.dirname(os.path.abspath(__file__)))


def _home() -> str:
    return os.path.expanduser("~")


def _system_roots() -> list[str]:
    roots = [os.path.abspath(os.sep)]
    for var in ("SystemRoot", "ProgramFiles", "ProgramFiles(x86)"):
        value = (os.environ.get(var) or "").strip()
        if value:
            roots.append(value)
    return roots


def classify_root(path: str) -> list[RootWarning]:
    """Advisory warnings for a candidate workspace root. Never refuses."""
    out: list[RootWarning] = []
    if not path:
        return out

    try:
        if _is_within(path, _studio_root()):
            out.append(RootWarning(
                WARN_STUDIO_DATA,
                "This is Studio's own data directory. The AI will be able to modify its "
                "auth database, saved models and chat history.",
            ))
    except Exception:
        pass

    try:
        if _is_within(path, _install_root()):
            out.append(RootWarning(
                WARN_INSTALL_DIR,
                "This is where Unslothed is installed. The AI will be able to modify the "
                "application itself.",
            ))
    except Exception:
        pass

    for root in _system_roots():
        try:
            if _real(path) == _real(root):
                out.append(RootWarning(
                    WARN_SYSTEM_DIR,
                    "This is a system directory. Tools have full read and write access here.",
                ))
                break
        except Exception:
            continue

    # Only the directory ITSELF is "broad" -- a folder inside home is an
    # ordinary choice and warning about it would train the user to click past
    # every warning, including the ones above that matter.
    broad = [_home()]
    for name in ("Documents", "Desktop", "Downloads"):
        broad.append(os.path.join(_home(), name))
    for root in broad:
        try:
            if _real(path) == _real(root):
                out.append(RootWarning(
                    WARN_BROAD,
                    "This is a broad location — the AI will see everything inside it.",
                ))
                break
        except Exception:
            continue

    return out
```

- [ ] **Step 4: Run the tests and verify they pass**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v
```

Expected: all PASS. Then run the control proof from Step 2 and record the observed result.

- [ ] **Step 5: Commit**

```bash
git add studio/backend/utils/workspace_root.py studio/backend/tests/test_workspace_root.py
git commit -m "feat(workspace): advisory classification for a chosen workspace root"
```

---

### Task 2: The global default setting

**Files:**
- Modify: `studio/backend/utils/workspace_root.py`
- Test: `studio/backend/tests/test_workspace_root.py`

**Interfaces:**
- Consumes: `get_app_setting(key, fallback)` and `upsert_app_settings(dict)` from `storage.studio_db` (signatures confirmed at `storage/studio_db.py:3735` and `:3746`).
- Produces: `WORKSPACE_ROOT_KEY = "workspace_root"`; `get_global_root() -> str | None`; `set_global_root(path: str | None) -> str | None`.

- [ ] **Step 1: Write the failing test**

Append to `studio/backend/tests/test_workspace_root.py`:

```python
from utils.workspace_root import WORKSPACE_ROOT_KEY, get_global_root, set_global_root


class TestGlobalDefault:
    def test_unset_reads_as_none(self, monkeypatch):
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: fallback)
        assert get_global_root() is None

    def test_a_stored_value_is_returned_expanded(self, monkeypatch, tmp_path):
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: str(tmp_path))
        assert get_global_root() == os.path.realpath(str(tmp_path))

    def test_a_blank_stored_value_reads_as_none(self, monkeypatch):
        """An empty string must not become a workdir of "" -- that would
        resolve to the process's cwd, which is not a directory the user chose."""
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: "   ")
        assert get_global_root() is None

    def test_a_missing_directory_reads_as_none(self, monkeypatch, tmp_path):
        """A root that no longer exists must not be returned: _get_workdir would
        then makedirs() it and silently recreate a folder the user deleted."""
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: str(tmp_path / "gone"))
        assert get_global_root() is None

    def test_setting_writes_under_the_documented_key(self, monkeypatch, tmp_path):
        seen = {}
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_write_setting", lambda mapping: seen.update(mapping))
        set_global_root(str(tmp_path))
        assert seen == {WORKSPACE_ROOT_KEY: os.path.realpath(str(tmp_path))}

    def test_clearing_writes_none(self, monkeypatch):
        seen = {}
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_write_setting", lambda mapping: seen.update(mapping))
        set_global_root(None)
        assert seen == {WORKSPACE_ROOT_KEY: None}
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v -k Global
```

Expected: `ImportError: cannot import name 'WORKSPACE_ROOT_KEY'`.

- [ ] **Step 3: Write the implementation**

Append to `studio/backend/utils/workspace_root.py`:

```python
WORKSPACE_ROOT_KEY = "workspace_root"


def _read_setting(key: str, fallback = None):
    """Indirection so tests can substitute storage without a database."""
    from storage.studio_db import get_app_setting
    return get_app_setting(key, fallback)


def _write_setting(mapping: dict) -> None:
    from storage.studio_db import upsert_app_settings
    upsert_app_settings(mapping)


def get_global_root() -> "str | None":
    """The global default workspace root, or None when unset or unusable.

    Returns None rather than a path in three cases that would each be worse
    than falling through to the sandbox:
      * unset -- the feature is opt-in, so nothing set means today's behaviour
      * blank -- an empty string would resolve to the process cwd
      * missing on disk -- _get_workdir calls makedirs() on whatever it returns,
        so a stale value would silently recreate a folder the user deleted
    """
    raw = _read_setting(WORKSPACE_ROOT_KEY, None)
    if not isinstance(raw, str) or not raw.strip():
        return None
    resolved = _real(raw.strip())
    if not os.path.isdir(resolved):
        return None
    return resolved


def set_global_root(path: "str | None") -> "str | None":
    """Store the global default. ``None`` or blank clears it. Never refuses a
    path -- classification is advisory and belongs to the caller."""
    if path is None or not str(path).strip():
        _write_setting({WORKSPACE_ROOT_KEY: None})
        return None
    resolved = _real(str(path).strip())
    _write_setting({WORKSPACE_ROOT_KEY: resolved})
    return resolved
```

- [ ] **Step 4: Run the tests and verify they pass**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add studio/backend/utils/workspace_root.py studio/backend/tests/test_workspace_root.py
git commit -m "feat(workspace): store and read the global default workspace root"
```

---

### Task 3: The seam — global fallback in `_get_workdir`

**Files:**
- Modify: `studio/backend/core/inference/tools.py` (~4 lines, ONE additive hunk)
- Test: `studio/backend/tests/test_workspace_root.py`

**Interfaces:**
- Consumes: `get_global_root()` from Task 2.
- Produces: no new exports. `_get_workdir(session_id)` gains one fallback branch.

**This is the entire merge-seam cost of the sub-project.** `tools.py` currently holds 17 insertions across 3 hunks with zero deletions. After this task it must hold ~21 insertions across 4 hunks and **still zero deletions**. Verify with `git diff --stat origin/main..HEAD -- studio/backend/core/inference/tools.py` and report the exact numbers.

- [ ] **Step 1: Write the failing test**

Append to `studio/backend/tests/test_workspace_root.py`:

```python
class TestResolutionOrder:
    """The three-way order: project root, then global default, then sandbox."""

    def test_the_global_default_is_used_when_no_project_applies(self, monkeypatch, tmp_path):
        from core.inference import tools
        chosen = tmp_path / "chosen"
        chosen.mkdir()
        monkeypatch.setattr(tools, "_project_workdir_for", lambda sid: None)
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: str(chosen))
        assert os.path.realpath(tools._get_workdir("sess-global")) == os.path.realpath(str(chosen))

    def test_a_project_root_beats_the_global_default(self, monkeypatch, tmp_path):
        from core.inference import tools
        proj = tmp_path / "proj"; proj.mkdir()
        glob = tmp_path / "glob"; glob.mkdir()
        monkeypatch.setattr(tools, "_project_workdir_for", lambda sid: str(proj))
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: str(glob))
        assert os.path.realpath(tools._get_workdir("sess-both")) == os.path.realpath(str(proj))

    # --- the control that protects every existing chat --------------------
    def test_control_with_nothing_set_the_sandbox_is_unchanged(self, monkeypatch):
        """The opt-in invariant. If this fails, the feature has changed
        behaviour for users who never asked for it."""
        from core.inference import tools
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: None)
        monkeypatch.setattr(tools, "_project_workdir_for", lambda sid: None)
        first = tools._get_workdir("sess-untouched")
        tools._workdirs.pop("sess-untouched", None)
        second = tools._get_workdir("sess-untouched")
        assert first == second
        assert "sandbox" in first.lower() or tools._contained_in_root(first, tools.sandbox_root())
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v -k Resolution
```

Expected: `test_the_global_default_is_used_when_no_project_applies` FAILS — `_get_workdir` returns a sandbox path, not the chosen directory. The other two should already pass (they describe existing behaviour), which is itself the check that the test file is wired correctly.

- [ ] **Step 3: Write the implementation**

In `studio/backend/core/inference/tools.py`, find this existing block in `_get_workdir` (around line 8432):

```python
        project_workdir = _project_workdir_for(session_id)
        if project_workdir:
            workdir = project_workdir
        elif session_id:
```

Insert the global fallback between the project branch and the session branch, so the block becomes:

```python
        project_workdir = _project_workdir_for(session_id)
        # Resolved once, not called twice in the branch below: it reads the
        # database, and two calls could disagree if the setting changed between
        # them -- the test would still pass and the workdir would be whichever
        # answer arrived second.
        global_workdir = None if project_workdir else _global_workspace_root()
        if project_workdir:
            workdir = project_workdir
        elif global_workdir:
            # A directory the user chose in Settings. Second in the order: a
            # project's own root is more specific and wins. Falls through to
            # the sandbox when unset, so this is opt-in.
            workdir = global_workdir
        elif session_id:
```

and add this helper next to the other module-level helpers in the same file:

```python
def _global_workspace_root() -> "str | None":
    """The user's chosen global workspace root, or None. Imported lazily so a
    storage failure cannot break tool dispatch."""
    try:
        from utils.workspace_root import get_global_root
        return get_global_root()
    except Exception:
        return None
```

**Do not** change the `_claim_sandbox`, `chmod`, or marker logic below it — a chosen root is a directory the user already owns and must not be re-permissioned. Note the existing guard `if not project_workdir and not session_id:` already skips claiming for a project root; confirm by reading that a global root does not reach `_claim_sandbox` either, and if it does, extend that condition rather than adding a new one.

- [ ] **Step 4: Run the tests and verify they pass**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v
```

Expected: all PASS.

- [ ] **Step 5: Verify the seam**

```bash
git diff --stat origin/main..HEAD -- studio/backend/core/inference/tools.py
git diff origin/main..HEAD -- studio/backend/core/inference/tools.py | grep -c "^-[^-]"
```

Expected: roughly `21 insertions(+)`, and the deletion count must be **0**. Report both numbers.

- [ ] **Step 6: Commit**

```bash
git add studio/backend/core/inference/tools.py studio/backend/tests/test_workspace_root.py
git commit -m "feat(workspace): fall back to the global root when no project applies"
```

---

### Task 4: Honour a per-project root

**Files:**
- Modify: `studio/backend/storage/studio_db.py:2253-2290`
- Test: `studio/backend/tests/test_workspace_root.py`

**Interfaces:**
- Produces: `upsert_chat_project` honours `project["rootPath"]` at creation; `update_chat_project` accepts `rootPath` in its patch map.

- [ ] **Step 1: Write the failing test**

Append to `studio/backend/tests/test_workspace_root.py`:

```python
class TestProjectRoot:
    def test_a_supplied_root_is_honoured_at_creation(self, tmp_path, monkeypatch):
        from storage import studio_db
        chosen = tmp_path / "mycode"; chosen.mkdir()
        captured = {}
        monkeypatch.setattr(studio_db, "get_chat_project", lambda pid: captured.get(pid))
        monkeypatch.setattr(studio_db, "_ensure_project_workspace", lambda p: os.path.realpath(p))
        resolved = studio_db._resolve_project_root(
            {"id": "p1", "name": "P", "rootPath": str(chosen)}, existing = None
        )
        assert os.path.realpath(resolved) == os.path.realpath(str(chosen))

    def test_an_existing_root_is_kept_when_none_is_supplied(self, tmp_path, monkeypatch):
        from storage import studio_db
        monkeypatch.setattr(studio_db, "_ensure_project_workspace", lambda p: os.path.realpath(p))
        resolved = studio_db._resolve_project_root(
            {"id": "p1", "name": "P"}, existing = {"rootPath": str(tmp_path / "old")}
        )
        assert os.path.realpath(resolved) == os.path.realpath(str(tmp_path / "old"))

    def test_rootpath_is_patchable(self):
        from storage import studio_db
        import inspect
        src = inspect.getsource(studio_db.update_chat_project)
        assert '"rootPath"' in src, "update_chat_project must accept rootPath in its patch map"

    # --- negative control -------------------------------------------------
    def test_control_a_supplied_root_beats_the_default(self, tmp_path, monkeypatch):
        """The whole defect being fixed: the old code computed
        _default_project_root and ignored the caller. If the default wins here,
        nothing has changed."""
        from storage import studio_db
        chosen = tmp_path / "mycode"; chosen.mkdir()
        monkeypatch.setattr(studio_db, "_ensure_project_workspace", lambda p: os.path.realpath(p))
        monkeypatch.setattr(studio_db, "_default_project_root", lambda proj: str(tmp_path / "DEFAULT"))
        resolved = studio_db._resolve_project_root(
            {"id": "p1", "name": "P", "rootPath": str(chosen)}, existing = None
        )
        assert "DEFAULT" not in resolved
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v -k ProjectRoot
```

Expected: `AttributeError: module 'storage.studio_db' has no attribute '_resolve_project_root'`.

- [ ] **Step 3: Write the implementation**

In `studio/backend/storage/studio_db.py`, add this helper immediately above `upsert_chat_project`:

```python
def _resolve_project_root(project: dict, existing: "Optional[dict]") -> str:
    """The root a project should use.

    Precedence: an explicitly supplied rootPath (a new project the user pointed
    somewhere), then the stored one (never silently relocate an established
    project -- its files are already in there), then the generated default.
    """
    supplied = (project.get("rootPath") or "").strip() if project.get("rootPath") else ""
    stored = existing.get("rootPath") if existing else None
    if supplied and not stored:
        return _ensure_project_workspace(supplied)
    if stored:
        return _ensure_project_workspace(stored)
    return _ensure_project_workspace(_default_project_root(project))
```

Replace the first four lines of `upsert_chat_project`:

```python
def upsert_chat_project(project: dict) -> dict:
    existing = get_chat_project(project["id"])
    root_path = _resolve_project_root(project, existing)
```

The `ON CONFLICT` clause keeps `COALESCE(chat_projects.root_path, excluded.root_path)` unchanged — `_resolve_project_root` already decided, and re-pointing is `PATCH`'s job.

In `update_chat_project`, add one entry to the `allowed` map (around line 2291):

```python
        "rootPath": ("root_path", patch.get("rootPath")),
```

- [ ] **Step 4: Run the tests and verify they pass**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root.py -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add studio/backend/storage/studio_db.py studio/backend/tests/test_workspace_root.py
git commit -m "feat(workspace): honour a caller-supplied project root, and allow re-pointing"
```

---

### Task 5: HTTP endpoints

**Files:**
- Modify: `studio/backend/routes/settings.py`
- Modify: `studio/backend/routes/chat_history.py` (`ChatProjectPatch` only)
- Test: `studio/backend/tests/test_workspace_root_routes.py`

**Interfaces:**
- Consumes: `classify_root`, `get_global_root`, `set_global_root`.
- Produces: `GET /api/settings/workspace-root` → `{"path": str|null, "warnings": [{"code","message"}]}`; `PUT /api/settings/workspace-root` body `{"path": str|null}` → same shape; `POST /api/settings/workspace-root/preview` body `{"path": str}` → `{"path", "warnings", "exists": bool}`. `ChatProjectPatch` gains `rootPath: Optional[str] = None`.

- [ ] **Step 1: Write the failing test**

Create `studio/backend/tests/test_workspace_root_routes.py`:

```python
# SPDX-License-Identifier: AGPL-3.0-only
# Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

"""HTTP contract for the chosen workspace root."""

from __future__ import annotations

import sys
from pathlib import Path

import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

_BACKEND_DIR = str(Path(__file__).resolve().parent.parent)
if _BACKEND_DIR not in sys.path:
    sys.path.insert(0, _BACKEND_DIR)


@pytest.fixture
def client(monkeypatch):
    """Auth is replaced via dependency_overrides, NOT monkeypatch: Depends()
    captures the function object at import time, so reassigning the module
    attribute would leave real auth in place and the tests would prove nothing."""
    from auth.authentication import get_current_subject
    from routes.settings import router
    import utils.workspace_root as mod

    store = {}
    monkeypatch.setattr(mod, "_read_setting", lambda key, fallback = None: store.get(key, fallback))
    monkeypatch.setattr(mod, "_write_setting", lambda mapping: store.update(mapping))

    app = FastAPI()
    app.include_router(router, prefix = "/api/settings")
    app.dependency_overrides[get_current_subject] = lambda: "test-subject"
    return TestClient(app)


class TestGlobalRootEndpoints:
    def test_unset_reads_as_null(self, client):
        r = client.get("/api/settings/workspace-root")
        assert r.status_code == 200
        assert r.json()["path"] is None

    def test_put_then_get_round_trips(self, client, tmp_path):
        r = client.put("/api/settings/workspace-root", json = {"path": str(tmp_path)})
        assert r.status_code == 200
        assert client.get("/api/settings/workspace-root").json()["path"] is not None

    def test_clearing_sets_null(self, client, tmp_path):
        client.put("/api/settings/workspace-root", json = {"path": str(tmp_path)})
        client.put("/api/settings/workspace-root", json = {"path": None})
        assert client.get("/api/settings/workspace-root").json()["path"] is None

    def test_a_dangerous_path_is_ACCEPTED_and_warned_about(self, client, tmp_path, monkeypatch):
        """Warn-only, at the HTTP boundary. A 4xx here would be a spec violation."""
        import utils.workspace_root as mod
        monkeypatch.setattr(mod, "_studio_root", lambda: str(tmp_path))
        r = client.put("/api/settings/workspace-root", json = {"path": str(tmp_path)})
        assert r.status_code == 200, "warn-only: a flagged path must still be accepted"
        assert any(w["code"] == "studio_data" for w in r.json()["warnings"])

    def test_preview_does_not_store(self, client, tmp_path):
        r = client.post("/api/settings/workspace-root/preview", json = {"path": str(tmp_path)})
        assert r.status_code == 200
        assert r.json()["exists"] is True
        assert client.get("/api/settings/workspace-root").json()["path"] is None, (
            "preview must not have side effects"
        )

    def test_preview_reports_a_missing_directory(self, client, tmp_path):
        r = client.post("/api/settings/workspace-root/preview",
                        json = {"path": str(tmp_path / "nope")})
        assert r.json()["exists"] is False


class TestProjectPatchModel:
    def test_chatprojectpatch_accepts_rootpath(self):
        from routes.chat_history import ChatProjectPatch
        assert "rootPath" in ChatProjectPatch.model_fields
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root_routes.py -v
```

Expected: 404s on the new routes, and a `KeyError`/assertion on `rootPath`.

- [ ] **Step 3: Write the implementation**

In `studio/backend/routes/settings.py`, add near the other path settings (the `/llama-cpp-path` block around line 968 is the pattern to follow):

```python
class WorkspaceRootBody(BaseModel):
    path: Optional[str] = None


class WorkspaceRootResponse(BaseModel):
    path: Optional[str] = None
    warnings: list[dict] = []


class WorkspaceRootPreviewResponse(WorkspaceRootResponse):
    exists: bool = False


def _root_payload(path):
    from utils.workspace_root import classify_root
    return {
        "path": path,
        "warnings": [{"code": w.code, "message": w.message} for w in classify_root(path or "")],
    }


@router.get("/workspace-root", response_model = WorkspaceRootResponse)
def get_workspace_root(current_subject: str = Depends(get_current_subject)) -> dict:
    from utils.workspace_root import get_global_root
    return _root_payload(get_global_root())


@router.put("/workspace-root", response_model = WorkspaceRootResponse)
def put_workspace_root(
    payload: WorkspaceRootBody, current_subject: str = Depends(get_current_subject)
) -> dict:
    """Set the global workspace root.

    A flagged path is still stored: the policy is warn-only, so the warnings
    ride along in the response for the UI to show. Returning 4xx here would
    quietly turn this into a blocking feature.
    """
    from utils.workspace_root import set_global_root
    return _root_payload(set_global_root(payload.path))


@router.post("/workspace-root/preview", response_model = WorkspaceRootPreviewResponse)
def preview_workspace_root(
    payload: WorkspaceRootBody, current_subject: str = Depends(get_current_subject)
) -> dict:
    """Classify a candidate WITHOUT storing it, so the UI can warn before the
    user commits."""
    import os
    candidate = (payload.path or "").strip()
    out = _root_payload(os.path.realpath(os.path.expanduser(candidate)) if candidate else None)
    out["exists"] = bool(candidate) and os.path.isdir(os.path.expanduser(candidate))
    return out
```

In `studio/backend/routes/chat_history.py`, add one field to `ChatProjectPatch` (line 287):

```python
    rootPath: Optional[str] = None
```

- [ ] **Step 4: Run the tests and verify they pass**

```bash
cd studio/backend && python -m pytest tests/test_workspace_root_routes.py tests/test_workspace_root.py -v
```

Expected: all PASS.

- [ ] **Step 5: Commit**

```bash
git add studio/backend/routes/settings.py studio/backend/routes/chat_history.py studio/backend/tests/test_workspace_root_routes.py
git commit -m "feat(workspace): endpoints for the global root, plus a non-storing preview"
```

---

### Task 6: The settings UI

**Files:**
- Create: `studio/frontend/src/features/settings/workspace-root-setting.tsx`
- Modify: the settings page that renders the other path settings (find it by locating where `/api/settings/llama-cpp-path` is consumed)

**Interfaces:**
- Consumes: the three endpoints from Task 5.
- Produces: `WorkspaceRootSetting`, no props.

- [ ] **Step 1: Read the conventions before writing**

Find the component that renders the existing llama.cpp path setting:

```bash
cd studio/frontend && grep -rn "llama-cpp-path" src/ | head
```

Read it. Match its layout, its fetch/auth convention (`authFetch` from `@/features/auth` — a raw `fetch` will miss the bearer token and the Tauri URL resolution), and its save/feedback pattern. Also locate the existing server-side folder browser used by the model picker's "Browse for a folder on the server" flow and reuse it rather than building a picker.

- [ ] **Step 2: Write the component**

Substitute the primitives found in Step 1 for the plain elements below; the logic must survive that substitution unchanged.

```tsx
// SPDX-License-Identifier: AGPL-3.0-only
// Copyright 2026-present the Unsloth AI Inc. team. All rights reserved. See /studio/LICENSE.AGPL-3.0

import { useCallback, useEffect, useState } from "react";
import { authFetch } from "@/features/auth";

type RootWarning = { code: string; message: string };

export function WorkspaceRootSetting() {
  const [path, setPath] = useState<string>("");
  const [warnings, setWarnings] = useState<RootWarning[]>([]);
  const [problem, setProblem] = useState("");
  const [saved, setSaved] = useState(false);

  useEffect(() => {
    let cancelled = false;
    authFetch("/api/settings/workspace-root")
      .then((r) => r.json().catch(() => null))
      .then((b) => {
        if (cancelled || !b) return;
        setPath(b.path ?? "");
        setWarnings(Array.isArray(b.warnings) ? b.warnings : []);
      })
      .catch(() => {
        if (!cancelled) setProblem("could not read the current workspace folder");
      });
    return () => {
      cancelled = true;
    };
  }, []);

  // Preview classifies WITHOUT storing, so the warning appears before the
  // user commits rather than after.
  const preview = useCallback(async (candidate: string) => {
    setProblem("");
    setSaved(false);
    if (!candidate.trim()) {
      setWarnings([]);
      return;
    }
    try {
      const res = await authFetch("/api/settings/workspace-root/preview", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ path: candidate }),
      });
      const b = await res.json().catch(() => null);
      if (!res.ok || !b) {
        setProblem("could not check that folder");
        return;
      }
      setWarnings(b.warnings ?? []);
      if (!b.exists) setProblem("that folder does not exist yet");
    } catch {
      setProblem("could not reach the backend to check that folder");
    }
  }, []);

  const save = useCallback(async () => {
    setProblem("");
    try {
      const res = await authFetch("/api/settings/workspace-root", {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ path: path.trim() || null }),
      });
      const b = await res.json().catch(() => null);
      if (!res.ok || !b) {
        setProblem("could not save the workspace folder");
        return;
      }
      setPath(b.path ?? "");
      setWarnings(b.warnings ?? []);
      setSaved(true);
    } catch {
      setProblem("could not reach the backend to save");
    }
  }, [path]);

  return (
    <div className="space-y-2">
      <label htmlFor="workspace-root">Workspace folder</label>
      <p>
        Chats outside a project will work in this folder instead of their own
        sandbox. Leave it empty to keep the sandbox.
      </p>
      <input
        id="workspace-root"
        value={path}
        placeholder="e.g. C:\\Users\\you\\code"
        onChange={(e) => setPath(e.target.value)}
        onBlur={(e) => preview(e.target.value)}
      />
      <button type="button" onClick={save}>Save</button>
      <button type="button" onClick={() => { setPath(""); setWarnings([]); }}>
        Clear (use the sandbox)
      </button>

      {warnings.map((w) => (
        <p key={w.code} role="alert">{w.message}</p>
      ))}
      {problem && <p role="alert">{problem}</p>}
      {saved && !problem && <p>Saved.</p>}
    </div>
  );
}
```

- [ ] **Step 3: Mount it on the settings page**

Add `<WorkspaceRootSetting />` beside the other path settings in the page found in Step 1. Do not create a new settings section if one already groups path settings.

- [ ] **Step 4: Typecheck, lint, build**

```bash
cd studio/frontend && npx tsc -b --noEmit
cd studio/frontend && npx biome check src/features/settings/workspace-root-setting.tsx
cd studio/frontend && npm run build
```

Expected: clean, and `dist/index.html` emitted.

- [ ] **Step 5: Commit**

```bash
git add studio/frontend/src/features/settings/workspace-root-setting.tsx
git commit -m "feat(workspace): settings UI for the global workspace folder"
```

---

### Task 7: The project folder UI

**Files:**
- Modify: the project settings surface (find it by locating where `PATCH /api/chat/projects/` is called from the frontend)
- Test: manual, plus typecheck and build

**Interfaces:**
- Consumes: `PATCH /api/chat/projects/{id}` with `{"rootPath": "..."}` (Task 5), and the preview endpoint from Task 5 for warnings.
- Produces: no new exports.

**Why this task exists:** the spec calls for the folder picker in **two** places — the global setting and a project's settings. Task 6 covers only the global one. Without this, a project root can be set by API but never by a person, which is the same defect this whole sub-project exists to fix, one level down.

- [ ] **Step 1: Find the project settings surface**

```bash
cd studio/frontend && grep -rn "chat/projects/" src/ | grep -i "patch\|update" | head
```

Read the component that issues the project update. Confirm whether it already has a settings/edit panel; if a project has no edit surface at all, say so in your report before building one — that would be a larger scope than this task assumes, and I want to rule on it rather than have you invent a panel.

- [ ] **Step 2: Add the folder row**

Reuse `WorkspaceRootSetting`'s shape from Task 6 rather than duplicating it: extract the shared parts into a small presentational component if that is clean, or import and parameterise it. The behavioural requirements are identical, with three differences:

- It reads and writes the project's `rootPath` via `PATCH /api/chat/projects/{id}`, not the settings endpoint.
- Its help text says the project's chats use this folder, and that it overrides the global setting.
- Changing it must warn that **existing files are not moved**: they stay in the old folder. This is the spec's stated behaviour, and a user who is not told will assume a move happened.

Use the same preview endpoint for warnings — it classifies a path, and nothing about it is specific to the global setting.

- [ ] **Step 3: Typecheck, lint, build**

```bash
cd studio/frontend && npx tsc -b --noEmit
cd studio/frontend && npm run build
```

- [ ] **Step 4: Manual verification against a running backend**

Start the backend and confirm, stating which you observed rather than asserting all of them:

1. A project's folder can be set, and a chat in that project then works in it.
2. The "existing files are not moved" warning appears when changing an already-populated project.
3. A project root overrides a set global default.
4. Clearing the global default returns non-project chats to their sandbox.

- [ ] **Step 5: Commit**

```bash
git add studio/frontend/src
git commit -m "feat(workspace): choose a project's folder from its settings"
```

---

## Final verification

- [ ] Run all workspace test files (never the full suite):

```bash
cd studio/backend
python -m pytest tests/test_workspace_root.py tests/test_workspace_root_routes.py -v
```

- [ ] Confirm the seam is one additive hunk and nothing else:

```bash
git diff --stat origin/main..HEAD -- studio/backend/core/inference/tools.py
git diff origin/main..HEAD -- studio/backend/core/inference/tools.py | grep -c "^-[^-]"
```

Expected: ~21 insertions, deletion count **0**.

- [ ] Confirm no forbidden file was touched:

```bash
git diff --name-only origin/main..HEAD -- studio/backend/routes/inference.py \
    studio/backend/core/inference/llama_cpp.py pyproject.toml studio/backend/main.py
```

Expected: `studio/backend/main.py` may appear (it carries sub-project 4's two lines); the other three must not.

- [ ] Confirm the opt-in invariant one final time: with no global root and no project root, `_get_workdir` returns a sandbox path.

- [ ] State explicitly which negative controls were proven to fail before their fix, and which — if any — turned out inert.
