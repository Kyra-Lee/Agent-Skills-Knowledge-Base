# Open WebUI Skills — A Primer

> **Purpose:** This document lays a foundation of how *Skills* work in the Open WebUI (OWUI) environment, so it can be reviewed by a human or passed to another LLM as shared context.
> **Status:** v1 — distilled from the official OWUI Skills documentation. A follow-up section will enrich this primer with how other LLM companies implement skills, converging into a set of best practices for OWUI specifically.
>
> Source: OWUI Skills documentation (`OWUI-skills.md`), cited as [1] below.

---

## 1. What a Skill Is

A **Skill** is a reusable, Markdown-based instruction set that teaches a model *how* to approach a task — written in plain text, not code [1]. Key distinctions:

- **Skills are instructions, Tools are capability.** Tools are executable Python scripts; Skills are guidelines the model reads and follows (e.g., code review checklists, writing style rules, troubleshooting playbooks, data analysis workflows) [1].
- **No deployment required.** If you can write a document, you can create a skill — no Python, API calls, or hosting [1].
- **Reusable across models.** One skill (e.g., "Code Review Guidelines") can be attached to every coding model; updating the skill updates the instruction for all models at once [1].
- **Composable with tools.** A skill teaches the model *how* to use a tool (check exit codes, stream long-running output), while the tool provides the *capability* [1].

A skill is physically **a small folder**: `SKILL.md` at the root (required, UTF-8 Markdown) plus optional supporting files — scripts in `scripts/`, detailed docs in `references/`, reusable outputs in `templates/`, and binary/visual assets in `assets/` [1].

### File limits [1]

| Limit | Value |
| :--- | :--- |
| Size of one file | 10 MiB |
| Total size of all files in one skill | 50 MiB |
| Number of files per skill | 1,000 |

---

## 2. How Skills Are Delivered to the Model

There are three ways a skill's instructions reach the model, plus a lazy-loading mechanism for everything else [1]:

| Mechanism | How it works |
| :--- | :--- |
| **`$` mention** | Type `$` in chat (or use the `/` menu) and pick a skill. Its instructions are injected into the system prompt with **every** message. A message containing only a `$` mention is sent with the skill name(s) as its text. |
| **Per-chat toggle** | The **+** menu → **Skills** turns skills on for just the current chat — no model edit needed. A badge shows the count; selected skills send their instructions with every message. |
| **Model binding** | Bind skills to a model under **Workspace > Models**; every new chat with that model starts with them selected. |
| **Lazy loading (manifest)** | Unselected skills are listed by name + description in a manifest. With the skill tools enabled, the model loads one via `view_skill` only when it needs it, so many skills cost little context. Without the skill tools, there is **no manifest**; selected/mentioned/model skills are injected up to the first 100,000 characters of `SKILL.md`, and model-bound skills are injected even when switched off in the chat. |

### Skill tools

With native function calling, the **Builtin Tools** capability, and the model's **Skills** category enabled, the model gets five tools [1]:

| Tool | Purpose |
| :--- | :--- |
| `view_skill` | Loads a skill's `SKILL.md` and its file list, up to the read limit. |
| `read_skill_file` | Reads one file of a skill from a given offset, up to the read limit per call (binary files return size + a link to the editor). |
| `grep_skill_files` | Searches a skill's text files for a word, phrase, or regex; returns up to 10 excerpts. |
| `update_skill_files` | Saves file changes as a **new version** of an editable skill; returns a conflict if the skill changed in the meantime. |
| `create_skill` | Creates a new private workspace skill from `SKILL.md` text plus optional text files. Offered to admins and users with **Skills Access**. |

All five respect the skill's access grants.

### Read limits

Two Admin Settings (General) control how much of a skill the model sees at once [1]:

| Setting | Default | Meaning |
| :--- | :--- | :--- |
| **Skill Read Limit** (`SKILL_CONTENT_MAX_CHARS`) | 20,000 | Max characters returned in one read |
| **Skill Preview Length** (`SKILL_PREVIEW_MAX_CHARS`) | 5,000 | Characters shown first when `SKILL.md` exceeds the read limit |

A `SKILL.md` longer than the read limit arrives as a preview; the model pages through the rest with `read_skill_file`.

### Optional: lazy loading of *selected* skills

The **Load selected skills on demand** admin setting (`ENABLE_SKILL_LAZY_LOADING`, off by default) also puts selected and model-bound skills in the manifest so they load only when needed. A `$`-mentioned skill is still always sent with the message. This requires the skill tools [1].

---

## 3. Skills from a Terminal Server

A connected **Open Terminal** server can carry its own skills, kept in the workspace next to the files they describe. OWUI reads them and offers them in the `$` picker alongside workspace skills, under ids of the form `terminal:<url-encoded name>` [1]:

- `$` mention injects the content (including the skill's directory and shipped file paths); otherwise they sit in the manifest and load via `view_skill`.
- `read_skill_file` pages through the terminal skill's `SKILL.md`; `grep_skill_files` searches it. Other files stay on the terminal, where the model opens them with the terminal's own file tools.
- The skill editor, version history, and `update_skill_files` work only on **workspace** skills.

A terminal server exposes two endpoints [1]:

| Endpoint | Returns |
| :--- | :--- |
| `GET /skills` | JSON array of `{id (prefixed terminal:, url-encoded name), name, description, location/path}` |
| `GET /skills/read?name=<name>` | One skill: `name`, `description`, `content` (the Markdown), `location/path`, and `resources` (array of shipped file paths) |

Auth notes: admin-configured terminals are called by the OWUI server with the connection's auth plus identity headers (`X-User-Id`, `X-Session-Id` = chat id, `X-Terminal-Context-Id`); user-added terminals are called from the user's browser with the bearer key and `X-Session-Id` (so a localhost terminal works with a hosted OWUI). Redirects are not followed [1].

---

## 4. Creating and Managing Skills

### Creation path

**Workspace > Skills → Create**. Fields: **Name** (human-readable), **Skill ID** (unique slug, auto-generated, editable only during creation), **Description** (shown in the manifest; the model uses it to decide whether to load the skill lazily), and **Files** (`SKILL.md` + supporting files). YAML frontmatter typed into `SKILL.md` auto-fills an empty Name/Description. A "Describe this change" note labels the version a save creates [1].

### Importing

- **Import** (upload): one or more `.zip`, `.skill`, `.json`, or `.md` files, ≤ 200 MiB. A ZIP may hold one skill (`SKILL.md` at root) or several skill folders; a `.md` becomes a skill with that file as its `SKILL.md`. Needs **Skills Access** + **Import Skills**.
- **Import from URL** (admins only): GitHub repo, folder, or `SKILL.md` links, or direct links to `.zip`/`.skill`/`.json`/`.md`. Links are checked against local-web-fetch rules; up to 5 redirects, 60 s timeout, 200 MiB cap.
- **Import folder**: a skill folder or a folder of skill folders from the user's machine.

Every import opens a **preview** with per-skill decisions: **Create**, **Create copy**, **Replace** (saves as a new version of an existing editable skill, sharing unchanged), or **Skip**. Skills with an existing ID/name start on Skip [1].

### Authoring from chat: `/skills:create`

Type `/skills:create` in a chat that already has content; the model distills the conversation into a standard `SKILL.md` (plus supporting text files) and saves it via `create_skill` as a **private** skill. Requires: **Skills Access** permission, a model with Builtin Tools + Skills category on, and a non-empty chat — the server checks all three [1]. Anything written after the command is treated as authoring guidance.

### Management actions (ellipsis menu on an editable skill)

**Edit**, **Access** (read/write grants per user/group), **Clone** (private copy with random-suffixed name/ID), **Export .skill / JSON / ZIP**, **Share** (publish to openwebui.com; ≤ 15 MiB), **Delete** (removes skill and full version history). Bulk export via the chevron menu beside Create. An **Active/Inactive toggle** removes a skill from manifests and `$` mentions without deleting it (users with write access can still have the model load it) [1].

### Version history

Every save that changes name, description, translations, or files stores a **complete snapshot**. The version menu lists **Current** plus prior versions (newest first, 20/page, with author and change note). Picking an earlier version opens it read-only with **Compare to current** (line-by-line diffs for text files) and **Set as Production** (needs write access; refuses if the skill changed since). Concurrent edits are handled with conflict detection (HTTP `409`): a refused save keeps the draft and offers Reload latest / Save as copy / Keep editing [1].

---

## 5. Access Control

Skills use the standard OWUI RBAC system [1]:

- **Private by default** — only the creator sees a new skill.
- **Share with users or groups** — grant `read` or `write` via the **Access** button (non-admins need the **Skills Sharing** permission to see sharing UI).
- **Read-only** — the editor shows a "Read Only" badge.
- **Model binding does NOT bypass access control.** When a user chats with a model, OWUI checks that user's read access to each attached skill; inaccessible skills are silently excluded. (Example from docs: an admin's private skill attached to a shared model is invisible to users without read access.)

### Required permissions

| Permission | Controls |
| :--- | :--- |
| Workspace > Skills Access | Access the Skills workspace; create/manage skills |
| Workspace > Import Skills | Import from ZIP/`.skill`/folder/JSON/Markdown; open openwebui.com skills |
| Workspace > Export Skills | Export skills/versions; share to openwebui.com |
| Sharing > Skills Sharing | Share with individual users or groups |
| Sharing > Skills Public Sharing | Make skills publicly accessible |

---

## 6. Skills API (Overview)

Endpoints live under `/api/v1/skills`. Reads need read access; changing the current version or deleting a version needs write access [1].

| Endpoint | Purpose |
| :--- | :--- |
| `GET /id/{id}/files` | List files (path, size, encoding); `version_id` for history |
| `GET /id/{id}/files/content` | Download one file |
| `GET /id/{id}/history` | List versions (newest first, paged 20) |
| `GET /id/{id}/history/{history_id}` | One version with file list |
| `GET /id/{id}/history/diff` | Changes between `from_id` and `to_id`; `/history/diff/file` for a single file's line diff |
| `POST /id/{id}/update/version` | Restore a stored version as current (optimistic lock via `expected_version_id`; `409` on conflict) |
| `DELETE /id/{id}/history/{history_id}` | Delete a non-current version; children re-link to parent |
| `POST /id/{id}/clone` | Copy under a new id/name, optionally from a version |
| `GET /export` | Export as json/zip/skill (`skill` takes exactly one id; `version_id` for a historical export) |
| `POST /load/url` | Admins only; resolve a URL to skill packages without saving |
| `POST /import/preview` | Parse uploaded archives and list found skills, flagging existing IDs/names |
| `POST /import` | Import with per-skill `decisions` (`create`/`copy`/`replace`/`skip`, optional id/name, `expected_version_id` for replace) |

`POST /create` and `POST /id/{id}/update` take skill files plus an optional `commit_message` version label; updates can send `operations` (`put`, `move`, `delete`) for single-file changes and require `expected_version_id` (409 on conflict) [1].

---

## 7. The OWUI Skill Authoring Standard

This is the standard OWUI itself follows when authoring via `/skills:create`, and the reference bar for hand-written skills [1].

### Frontmatter

| Key | Rule |
| :--- | :--- |
| `name` | Lowercase, hyphenated, no spaces, ≤ 64 chars |
| `description` | ≤ 1024 chars, ends with a period; name the capability, skip implementation detail, don't repeat the skill name, avoid words like "powerful", "comprehensive", "seamless", "advanced", "robust" |
| `platforms` | `[macos]`, `[linux]`, or `[windows]` only for OS-bound skills; omit for portable skills |

### Body sections (in this order)

1. `# <Human Title>` + short intro (what it does / doesn't do; what it assumes is installed)
2. `## When to Use` — concrete trigger phrases
3. `## Prerequisites` — exact env vars, credentials, install steps, or `None`
4. `## How to Run` — the canonical workflow, framed through the tools the model actually has
5. `## Quick Reference` — flat list of commands, routes, files, APIs
6. `## Procedure` — numbered, copy-paste-exact steps
7. `## Pitfalls` — known limits and failure modes
8. `## Verification` — one focused check that proves the skill works

### Framing the tools

Name tools in backticks (`run_command`, `write_file`, `view_skill`); describe shell work as run through `run_command`; prefer the read/search tools over raw shell utilities where one exists. Third-party CLIs are fine if the procedure is clear the agent invokes them via `run_command`.

### Quality bar

- Use commands, routes, paths, function names, config keys, and error text **exactly as they appear in the source** — invented flags or APIs are the main way a skill goes wrong.
- Keep `SKILL.md` scannable: roughly **100 lines** for a simple workflow, **200** for a complex one.
- Put the bulk elsewhere: larger scripts → `scripts/`, detailed docs → `references/`, reusable outputs → `templates/`, binary/visual assets → `assets/`.
- A skill that only points at other skills is worth nothing — write the one that does the work.

### Example frontmatter

```yaml
---
name: code-review-guidelines
description: Step-by-step instructions for thorough code reviews
---
```

---

## 8. Use Cases (from the docs)

- **Code review standards** — team checklist as a skill attached to coding models.
- **Writing style guide** — tone, formatting, terminology for content models.
- **Troubleshooting playbooks** — encoded runbooks the model follows like a senior engineer would.
- **Tool usage instructions** — pair with Open Terminal: check exit codes, `set -e`, stream output for long commands.

---

## 9. Limitations

- **Plain text only.** OWUI stores a skill's scripts for the model to *read*; running them requires an execution tool (Open Terminal, code interpreter). For computation, API calls, or system access, use Tools [1].
- **Context window pressure.** `$`-mentioned/selected skills go into the system prompt with every message; a very long skill on a small-context model can crowd out the conversation [1].
- **Lazy loading needs function calling.** The manifest and skill tools require native function calling with Builtin Tools enabled [1].

---

## 10. Terminology Cheat Sheet

| Term | Meaning |
| :--- | :--- |
| `SKILL.md` | The required root Markdown instruction file of a skill |
| Frontmatter | YAML block at the top of `SKILL.md` (`name`, `description`, optional `platforms`) |
| Skill ID | Unique slug identifying a skill; editable only at creation |
| Manifest | Name+description list of available skills the model can load lazily |
| Model binding | Attaching skills to a model so new chats start with them selected |
| Lazy loading | On-demand loading of unselected skills via `view_skill` |
| Version | Immutable snapshot created on each meaningful save |
| Set as Production | Restoring a historical version as the current one |
| `.skill` file | A ZIP holding one skill's folder; on import, name/description come from frontmatter |
| `terminal:<name>` | ID prefix for skills served by an Open Terminal server |

---

## 11. Planned Enrichment (to be added)

- How other LLM companies / agent platforms structure and load skills (Anthropic Agent Skills and peers).
- Comparative analysis of authoring standards, progressive disclosure / lazy loading, and tool framing.
- A distilled **best-practices checklist for OWUI skills** built from the comparison plus the OWUI constraints in §7 and §9.

---

### Source

[1] OWUI Skills documentation (`OWUI-skills.md`) — official Open WebUI feature documentation for Skills: creation, delivery, skill tools, terminal-server skills, import/export, versioning, access control, API, authoring standard, use cases, and limitations.
