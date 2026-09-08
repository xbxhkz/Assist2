# Unslothed User-Chosen Workspace Directory — Design Spec

## Context

Fifth sub-project of the initiative to fork [Unsloth](https://github.com/unslothai/unsloth/) and
extend it into a local AI agent platform. Sub-projects 1 and 2 (vision tools, code intelligence)
and 4 (draft-model chooser) are merged; sub-project 3 (branding, Windows installer, Docker) ships
the installer that carries them.

The owner asked to "give the AI model access to a folder, path, or directory I choose, so it
doesn't have to only use the files inside the chat's working directory", suggesting names in the
shape of `WORK_DIR` / `WORKSPACE_DIR` / `BASE_PATH` / `ROOT_PATH` / `PROJECT_ROOT` / `ALLOWED_DIR`.

## What already exists, and what is actually missing

Investigation reshaped the work before design started.

**The hard part is built.** Studio has a *projects* concept, and a chat inside a project already
resolves its tool working directory to that project's `rootPath` rather than the per-session
sandbox (`core/inference/tools.py:8432-8434`). `_get_project_workdir` even validates that the
sandbox sits inside the root. Pointing tools at a directory other than the sandbox is therefore a
proven path, not a new one.

**The control is what is missing.** `POST /projects` takes a `ChatProject` payload that *contains*
`rootPath`, so the API looks like it accepts one. `upsert_chat_project` discards it:

```python
root_path = existing.get("rootPath") if existing else None
if not root_path:
    root_path = _default_project_root(project)      # the caller's value is never read
```

and the conflict clause is `COALESCE(chat_projects.root_path, excluded.root_path)`, which keeps
whatever is already stored. `ChatProjectPatch` has no `rootPath` field at all. So every project is
forced into `Documents\Unsloth Studio\Projects\<slug>-<id>`, and no API or UI path can change it.

This sub-project supplies the control, not the mechanism.

## Decisions taken by the owner

| Question | Decision |
|---|---|
| What owns the setting | **Both** a global default and a per-project override |
| Level of access | **Full read/write** — the AI works on the files as it does in the sandbox |
| Guardrails | **Warn only, never refuse** — no path is blocked |

The owner was told that "both" costs roughly the sum of the two work items and both risk surfaces,
and chose it deliberately. Recorded so it is not re-litigated.

## Goal

Let the user choose the directory the AI's tools operate in — globally, and per project — with
warnings that state what a given choice exposes.

## Architecture

### Resolution order

```
1. Project rootPath      (the chat is in a project that has one)   -- exists, unblocked here
2. Global default root   (set in Settings)                          -- new
3. Per-session sandbox   (today's behaviour)                        -- unchanged
```

**The global default does not apply to project chats, and that is deliberate.** Every project has a
`root_path` populated the first time its workspace is ensured — auto-generated if the user never
chose one — so rule 1 always matches for a chat inside a project. The global default therefore
governs chats *outside* any project. Stated explicitly because the alternative reading ("the global
default is a fallback that existing projects will pick up") is plausible and wrong: an existing
project keeps its current folder until the user re-points it via `PATCH`, which is the safe
behaviour, since silently relocating an established project's working directory would strand the
files already in it.

`_get_workdir(session_id)` is the single resolution point and **everything downstream follows it
automatically**: the five vision tools, the five code tools (via `assist_code/paths.py`'s
`workspace_for`, which calls `_get_workdir` and uses its answer as both the fallback *and* the
ceiling), and upstream's `terminal`, `python` and `edit_file`. No tool needs to know this feature
exists. That single point is what keeps the change small enough to be safe.

### Storage

- **Per-project:** the `chat_projects.root_path` column already exists and is already read. Nothing
  is added; the code stops ignoring the caller's value.
- **Global:** the existing `app_settings` key/value table via `upsert_app_settings`
  (`storage/studio_db.py:3746`). This mirrors how `/llama-cpp-path` and `/hugging-face-cache`
  already persist path settings.

### API

| Endpoint | Change |
|---|---|
| `POST /projects` | Honour a caller-supplied `rootPath` **at creation**. On an upsert against an existing project, an omitted `rootPath` still keeps the stored value — only `PATCH` re-points |
| `ChatProjectPatch` | Gains `rootPath`, the single way to re-point an existing project |
| `GET/PUT /api/settings/workspace-root` | The global default, following the `/llama-cpp-path` pattern |
| `POST /api/settings/workspace-root/preview` | Returns the warnings for a candidate path **without** setting it, so the UI can show them before the user commits |

### UI

Studio already ships a server-side folder browser ("Browse for a folder on the server", used by the
model picker's custom-folder flow). The picker is reused rather than rebuilt, in two places: the
global setting, and a project's settings.

## The merge seam — the one place this costs the fork

The fork's footprint in `core/inference/tools.py` is currently **17 insertions across 3 hunks, zero
deletions**. The global fallback adds roughly **4 more lines in a fourth additive hunk**, taking it
to ~21 insertions.

The split is worth stating plainly:

- The **per-project** half costs **zero** seam. The resolution already exists, and
  `upsert_chat_project` lives in `storage/studio_db.py`, which is not a seam file.
- The **global** half is what buys the fourth hunk.

That is the concrete price of choosing "both". It is small and additive, and the owner accepted it.

## Warnings

No path is refused. `POST /api/settings/workspace-root/preview` classifies a candidate and the UI
renders the result. Each warning names what is at stake rather than asking "are you sure?" — under
a warn-only policy, a warning that does not say what it is warning about is decoration.

| Path | Warning |
|---|---|
| `~/.unsloth/studio` | This is Studio's own data directory. The AI will be able to modify its auth database, saved models and chat history. |
| The install directory | This is where Unslothed is installed. The AI will be able to modify the application itself. |
| Filesystem root, `C:\Windows`, `Program Files` | This is a system directory. Tools have full read/write access here. |
| Home, Documents, Desktop | This is a broad location — the AI will see everything inside it. |

Classification is advisory and never blocks. A path matching nothing is accepted silently.

## What deliberately does not change

A chat with no project root and no global default keeps today's behaviour exactly: the per-session
sandbox, its ownership markers, symlink rejection, containment checks and `0o700` permissions. The
feature is **opt-in by construction** — set nothing and the resolution is byte-identical to today.

## Risks

**`_ensure_project_workspace` becomes the security boundary.** It currently does
`ensure_dir(root).resolve()` with no validation, which is safe only because the server generates
the value. Once a user supplies it, that function is the gate, and under warn-only it is
deliberately a permissive one. This is the intended behaviour, not an oversight.

**Re-pointing a project does not move its files.** `_get_project_workdir` requires the sandbox to
sit inside the root, so changing `rootPath` on a project that already has files leaves them at the
old location. The design keeps them in place and says so in the UI rather than moving data the user
did not ask to move.

**`terminal` is the real blast radius, not our tools.** The ten tools this fork adds confine
themselves to the workdir; upstream's `terminal` runs *in* it. Granting a root grants shell access
to that root. This follows directly from the full-read/write decision.

**Two paths can break the product that hosts the feature** — Studio's data directory and the
install directory. Warn-only keeps both selectable; the warnings above are the mitigation.

## Testing

Every guard gets a negative control that is demonstrated to fail before its fix. Eight controls in
this project have turned out inert, one of them caught mid-run, so "the test passes" is not
evidence on its own.

- A project whose `rootPath` is honoured actually changes `_get_workdir`'s answer — not merely the
  database row.
- The global default is used **only** when no project root applies, and the project root wins when
  both are set.
- With neither set, `_get_workdir`'s result is unchanged from today. This is the control that
  protects every existing chat.
- Each warning class fires on its own path and not on its neighbours, so a passing warning test
  cannot be a neighbouring classifier misfiring.
- `workspace_for` follows the new root, so the code tools confine to the chosen directory rather
  than the sandbox.

## Out of scope

- Read-only or per-folder permission modes (the owner chose full read/write)
- Blocking or refusing any path (the owner chose warn-only)
- Moving existing project files when a root is re-pointed
- Per-chat roots for chats outside a project — the global default covers that case
- Any change to the sandbox's own ownership, marker or permission logic
