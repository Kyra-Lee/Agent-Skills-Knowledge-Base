# Open WebUI Skills Primer

**Version:** 2.0  
**Target Audience:** LLM agents and human reviewers requiring accurate baseline knowledge of the OWUI Skills feature  
**Sources:** Open WebUI official documentation [1], Agent Skills Documentation Knowledge Base [2]

---

## 1. What a Skill Is

A **Skill** is a reusable, markdown-based instruction set that you attach to models or invoke on-the-fly in chat. Unlike Tools (executable Python scripts), Skills are plain-text instructions: code review guidelines, writing style rules, troubleshooting playbooks, data analysis workflows. The model reads them and follows them [1].

**Key characteristics:**
- Markdown content with optional supporting files
- No code execution capability (requires separate tools like Open Terminal)
- Composable with tools (skill teaches the model *how* to use a tool)
- Persistent across chats when bound to models

---

## 2. How Skills Are Delivered to the Model

### Delivery Mechanisms

| Mechanism | Description |
| :--- | :--- |
| **User-selected ($ mention)** | Type `$` in chat input or use the `/` menu to inject a skill's instructions into the current message |
| **Per-chat toggle** | The **+** menu > **Skills** turns skills on for the current chat; badge shows how many are active |
| **Model-attached skills** | A new chat with a single model starts with the model's skills selected |

### Lazy Loading and Manifest

With the skill tools, every other active skill that is not selected is listed in a manifest by name and description. The model loads one with `view_skill` when it needs it, so many skills cost little context [1].

---

### Augmentation: Progressive Disclosure Pattern

The lazy-loading mechanism in OWUI aligns with the **progressive disclosure** pattern used across the Agent Skills ecosystem:

| Tier | What's loaded | When | Token cost |
| :--- | :--- | :--- | :--- |
| 1. Catalog | Name + description | Session start | ~50-100 tokens per skill |
| 2. Instructions | Full `SKILL.md` body | When the skill is activated | <5,000 tokens (recommended) |
| 3. Resources | Scripts, references, assets | When the instructions reference them | Varies |

OWUI implements Tiers 1 and 2 via its manifest and lazy-loading system. Tier 3 (bundled resources) is supported through `read_skill_file`, which allows the model to load supporting files on demand [2].

### Augmentation: Context Budget Management

Some implementations (e.g., OpenAI Codex) apply automatic budget management when many skills are installed:
- Initial skill list uses at most 2% of the model's context window, or 8,000 characters when the context window is unknown
- When many skills exist, descriptions are shortened first
- For very large skill sets, some skills may be omitted from the initial list with a warning [2]

OWUI does not implement automatic shortening or omission; all discoverable skills appear in the manifest. Users should monitor context usage when attaching many skills to models [1].

---

**Enabling lazy loading for selected skills:**  
The setting **Load selected skills on demand** in Admin Settings > General (`ENABLE_SKILL_LAZY_LOADING`, off by default) lists selected and model skills in the manifest too, so they load only when needed. This setting requires the skill tools [1].

### Skill Tools

With native function calling, the **Builtin Tools** capability and the model's **Skills** category on, the model gets these tools:

| Tool | Function |
| :--- | :--- |
| `view_skill` | Loads a skill's `SKILL.md` and the list of its files, up to the read limit |
| `read_skill_file` | Reads one file of a skill from a given offset, up to the read limit per call. For a binary file it returns the size and a link to the skill editor |
| `grep_skill_files` | Searches a skill's text files for a word, phrase or regular expression and returns up to 10 excerpts |
| `update_skill_files` | Saves file changes as a new version of a skill the user can edit. Refused with a conflict when someone saved the skill in the meantime |
| `create_skill` | Creates a new private skill in the workspace from the `SKILL.md` text and optional text files. Offered to admins and to users with the **Skills Access** permission |

All five follow the skill's access grants. A user with write access to an inactive skill can still have the model load and read it [1].

### Read Limits

Two settings in Admin Settings > General control how much of a skill the model gets at once:

| Setting | Default | What it sets |
| :--- | :--- | :--- |
| **Skill Read Limit** | 20,000 | Most characters the model gets in one read |
| **Skill Preview Length** | 5,000 | Characters the model gets first when `SKILL.md` is longer than the read limit |

Leave a field empty for the default, or set `SKILL_CONTENT_MAX_CHARS` and `SKILL_PREVIEW_MAX_CHARS` environment variables [1].

A `SKILL.md` longer than the read limit arrives as a preview, and the model reads on with `read_skill_file` [1].

---

## 3. Skills from a Terminal Server

A connected **Open Terminal** server can carry its own skills, kept in the workspace next to the files they describe. Open WebUI reads them from the server and offers them in the `$` picker beside your workspace skills, under ids of the form `terminal:<url-encoded name>` [1].

**Terminal skill behavior:**
- What gets injected carries the skill's directory and the files it ships with, so the model knows where to read them from in the workspace
- `read_skill_file` pages through a terminal skill's `SKILL.md` and `grep_skill_files` searches it; its other files stay on the terminal, and the model opens them with the terminal's own file tools
- The skill editor, version history and `update_skill_files` work on workspace skills only

### Terminal Server API Contract

Writing a terminal server of your own means answering two endpoints:

| Endpoint | Returns |
| :--- | :--- |
| `GET /skills` | A JSON array, one entry per skill, each with `id` (prefixed `terminal:`, the name url-encoded), `name`, `description` and `location` (or `path`) |
| `GET /skills/read?name=<name>` | One skill: `name`, `description`, `content` (the Markdown instructions), `location` (or `path`) and `resources`, an array of file paths shipped with the skill |

Open Terminal serves both from v0.13.0 [1].

**Auth headers:** A terminal an admin configured is called by the Open WebUI server, with the connection's auth, `Accept: application/json`, `X-User-Id`, `X-Session-Id` (the chat id) and `X-Terminal-Context-Id`. Redirects are not followed. A terminal a user added in their own settings is called from that user's browser, with the bearer key and `X-Session-Id` [1].

---

### Augmentation: Cross-Platform Skill Discovery

While OWUI uses a central workspace for skill management, the broader Agent Skills ecosystem has converged on filesystem-based discovery conventions:

| Scope | Path Convention | Purpose |
| :--- | :--- | :--- |
| Project | `<project>/.agents/skills/` | Cross-client interoperability; skills specific to a repository |
| User | `~/.agents/skills/` | Cross-client interoperability; user-available skills |
| User | `~/.<client>/skills/` | Client-specific location |

OWUI does not currently implement filesystem scanning for external skills; all skills are managed through the Workspace UI. However, OWUI's terminal server API (`GET /skills`) provides a similar remote discovery mechanism for connected servers [1, 2].

### Augmentation: Trust Considerations for Project-Level Skills

For locally-scanned skill systems, some implementations consider gating project-level skill loading on a trust check—only loading skills from repositories the user has marked as trusted. This prevents untrusted repositories from silently injecting instructions into the agent's context [2].

OWUI's explicit import workflow (requiring preview and user confirmation) provides similar protection against untrusted skill content [1].

## 4. Creating and Managing Skills

### Manual Creation

Navigate to **Workspace > Skills** and click **Create** in the Workspace header.

| Field | Description |
| :--- | :--- |
| **Name** | Human-readable display name (e.g., "Code Review Guidelines") |
| **Skill ID** | Unique slug, auto-generated from the name. Editable during creation, read-only afterwards |
| **Description** | Short summary shown in the manifest. For model-attached skills, the model uses this to decide whether to load the full instructions |
| **Files** | `SKILL.md` with the full skill instructions in Markdown, plus any supporting files |

The globe selector beside the name translates the **Name** and **Description** per language [1].

---

### Augmentation: Frontmatter Leniency for Cross-Platform Compatibility

Skills authored for other clients may contain technically invalid YAML that their parsers accept. A common issue is unquoted values containing colons:

```yaml
description: Use this skill when: the user asks about PDFs  # Technically invalid
```

Robust implementations use fallback parsing (wrapping such values in quotes or converting to block scalars) to improve cross-platform compatibility [2].

OWUI parses frontmatter as "real YAML" and may reject malformed input during creation, requiring authors to ensure valid YAML syntax [1].

While you create a skill, YAML frontmatter typed or pasted into `SKILL.md` fills in an empty **Name** and **Description**: `name: code-review-guidelines` becomes the name "Code Review Guidelines". The frontmatter is parsed as real YAML, so quoted and multi-line values come through as written [1].

### Supporting Files

A skill is a small folder: `SKILL.md` at its root, plus any files the instructions refer to, such as scripts in `scripts/`, detailed docs in `references/`, reusable outputs in `templates/` and images or other assets in `assets/` [1].

| Limit | Value |
| :--- | :--- |
| Size of one file | 10 MiB |
| Size of all files in one skill | 50 MiB |
| Files per skill | 1,000 |

Text files are stored as text and any other file as binary. `SKILL.md` stays at the root and must be UTF-8 text [1].

### Importing Skills

The chevron menu beside **Create** has three import entries:

- **Import**: Takes one or more `.zip`, `.skill`, `.json` or `.md` files, up to 200 MiB per import. A `.skill` file is a ZIP and is read the same way. A ZIP holds a single skill with `SKILL.md` at its root, or several skill folders that each contain a `SKILL.md`. A `.md` file becomes a skill with that file as its `SKILL.md`. A JSON file is a skill export from Open WebUI, older exports with a single `content` field included. Needs the **Skills Access** and **Import Skills** permissions [1].
- **Import from URL** (admins only): Downloads skills from a link [1].
- **Import folder**: Takes a skill folder from your computer, or a folder holding several skill folders, each with its own `SKILL.md`. Needs the **Skills Access** and **Import Skills** permissions [1].

Every import opens a preview listing each skill with its name, ID and files. For each one, choose **Create**, **Create copy**, **Replace** or **Skip** [1].

#### Importing from a URL

**Import from URL** in the chevron menu beside **Create** loads skills straight from the web. It is available to administrators. Paste one of these links:

| Link | What gets imported |
| :--- | :--- |
| A GitHub repository, such as `https://github.com/owner/repo` | Every skill in the repository: each folder that holds a `SKILL.md`, or the whole repository when `SKILL.md` sits at its root |
| A GitHub folder, `https://github.com/owner/repo/tree/<branch>/<folder>` | The skills in that folder |
| A GitHub `SKILL.md`, `https://github.com/owner/repo/blob/<branch>/<folder>/SKILL.md` | The skill in that `SKILL.md`'s folder, with all its files |
| A direct link to a `.zip`, `.skill`, `.json` or `.md` file | The skills in that file, read the same way as an upload |

GitHub links are downloaded as a ZIP of the repository at that branch, tag or commit. The link follows up to 5 redirects, gives up after 60 seconds and stops at 200 MiB [1].

### Creating Skills from Chat (`/skills:create`)

A workflow you just worked through in a chat can be turned into a skill without writing it out yourself. Type `/skills:create` in that chat, optionally followed by what the skill should cover, and the model gathers the material, authors one `SKILL.md` to the standard below and saves it to your **Workspace > Skills** with the `create_skill` tool, along with any supporting text files under `scripts/`, `references/`, `templates/` or `assets/` [1].

The command needs:
- the **Skills Access** workspace permission (admins always have it)
- a model with built-in tools active and its **Skills** category switched on under **Builtin Tools**
- a chat that **already has content**, since the chat is the raw material

The `/` menu offers the command once the chat has content and you hold the permission [1].

### Skill Management Actions

From the Skills workspace list, use the ellipsis menu (**...**) on a skill you can edit:

| Action | Description |
| :--- | :--- |
| **Edit** | Open the skill in the editor |
| **Access** | Open the skill's access settings straight from the list. Changes save as you make them. Shown to non-admins only with the **Skills Sharing** permission |
| **Clone** | Create a private copy of the current version, with all its files, and open it in the editor. The copy's name and ID get a short random suffix |
| **Export .skill** | Download the skill with all its files as one `.skill` file, a ZIP holding the skill's folder |
| **Export JSON** / **Export ZIP** | Download the skill with all its files |
| **Share** | Publish the skill with all its files on openwebui.com. A skill can be shared up to 15 MiB in total |
| **Delete** | Permanently remove the skill and its whole version history (Shift+Click for quick deletion) |

**Bulk export**: Click **Export ZIP** or **Export JSON** in the chevron menu beside **Create** to export all accessible skills as one file. The ZIP holds one folder per skill [1].

**Active/Inactive toggle**: Inactive skills are left out of manifests and `$` mentions, even when bound to a model. Users with write access can still have the model load them [1].

### Version History

Every save that changes a skill's name, description, translations or files stores a complete snapshot of the skill as a new version. Switching a skill on or off and changing its access leave the history as it is [1].

The version menu at the top of the editor's file panel lists **Current** and every earlier version, newest first and 20 per page, each with its author and the note from **Describe this change** (or a short version ID when there is none). Picking an earlier version opens it read-only with two actions:

- **Compare to current**: Shows what differs between that version and the current one
- **Set as Production**: Makes that version the current one again. Needs write access. If someone saved the skill after you opened the version, the switch is refused and the editor offers to reload the latest state [1].

If someone else saves the skill while you edit it, your save is refused and your draft stays in the editor. A banner offers **Reload latest**, **Save draft as copy** or **Keep editing** [1].

### Binding Skills to a Model

1. Go to **Workspace > Models**
2. Edit a model and open the **Skills** row
3. Pick the skills you want this model to always have access to
4. Click **Save & Update**

Every new chat with the model starts with these skills selected [1].

---

## 5. Access Control

Skills use the same **Access Control** system as other workspace resources:

- **Private by default**: Only the creator can see and edit a new skill
- **Share with users or groups**: Grant `read` or `write` access via the **Access** button in the editor or the **Access** entry in the list menu
- **Read-only access**: Users with read access can view but not edit. The editor shows a "Read Only" badge

:::caution Attached skills still require user access
Attaching a skill to a model does **not** bypass access control. When a user chats with the model, Open WebUI checks whether that user has read access to each attached skill. Skills the user can't access are silently excluded.

**Solution**: Make sure users who need the model's skills also have read access to each skill (via access grants, group permissions, or by making the skill public).
::: [1]

### Required Permissions

| Permission | What it controls |
| :--- | :--- |
| **Workspace > Skills Access** | Access the Skills workspace and create/manage skills |
| **Workspace > Import Skills** | Import skills from ZIP, `.skill`, folder, JSON or Markdown, and open skills from openwebui.com |
| **Workspace > Export Skills** | Export skills and their versions as `.skill`, ZIP or JSON, and share skills to openwebui.com |
| **Sharing > Skills Sharing** | Share skills with individual users or groups |
| **Sharing > Skills Public Sharing** | Make skills publicly accessible |

---

## 6. Skills API (Overview)

The editor's file, version, import and export features are backed by these endpoints under `/api/v1/skills`:

| Endpoint | What it does |
| :--- | :--- |
| `GET /id/{id}/files` | Lists the skill's files with path, size and encoding. `version_id` picks a historical version |
| `GET /id/{id}/files/content` | Downloads one file, given `path` and `version_id` |
| `GET /id/{id}/history` | Lists versions, newest first, 20 per `page` |
| `GET /id/{id}/history/{history_id}` | Returns one version with its file list |
| `GET /id/{id}/history/diff` | Lists what changed between `from_id` and `to_id`: metadata and added, deleted or modified files. `/history/diff/file` with a `path` returns the line diff of one text file |
| `POST /id/{id}/update/version` | Makes a stored version the current one. The body carries `version_id` and `expected_version_id`; when that no longer matches, the call returns `409` |
| `DELETE /id/{id}/history/{history_id}` | Deletes a version other than the current one and links its child versions to its parent |
| `POST /id/{id}/clone` | Copies the skill under a new `id` and `name`, optionally from a given `version_id`. The copy is private |
| `GET /export` | Exports skills as `format=json`, `format=zip` or `format=skill`. Repeat `ids` to pick skills; `format=skill` takes exactly one |
| `POST /load/url` | Admins only. Downloads the skills behind `{"url": "..."}` and returns them as skill packages, without saving anything |
| `POST /import/preview` | Reads uploaded ZIP, `.skill`, JSON or Markdown files (up to 200 MiB in total) and lists the skills found, flagging IDs and names that already exist |
| `POST /import` | Imports the same uploads with a `decisions` list, one entry per skill: `action` (`create`, `copy`, `replace` or `skip`), an optional `id` and `name`, plus `expected_version_id` for `replace` |

`POST /create` and `POST /id/{id}/update` take the skill's files as `files`, and an optional `commit_message` that labels the new version. An update that sends `files` or `operations` needs `expected_version_id` and returns `409` when someone saved the skill in the meantime [1].

---

## 7. The OWUI Skill Authoring Standard

This is the standard Open WebUI itself follows when it writes a skill through `/skills:create`, and the one to hold your own skills to.

---

### Augmentation: Additional Frontmatter Fields (Cross-Platform)

The broader Agent Skills specification defines additional frontmatter fields beyond OWUI's three:

| Field | Purpose | OWUI Status |
| :--- | :--- | :--- |
| `name` | Lowercase, hyphenated, 64 chars max | Supported [1] |
| `description` | Use-case description, 1024 chars max | Supported [1] |
| `platforms` | OS-specific requirements (`[macos]`, `[linux]`, `[windows]`) | Supported [1] |
| `compatibility` | Environment requirements or version notes | Not implemented in OWUI [2] |

OWUI's implementation covers the core fields (`name`, `description`, `platforms`). The `compatibility` field, used by other platforms for environment requirements, is not currently processed by OWUI [1, 2].

---

### Frontmatter

| Key | Rule |
| :--- | :--- |
| `name` | Lowercase and hyphenated, no spaces, 64 characters at most |
| `description` | A clear description, **1024 characters at most**, ending in a period. Name the capability, skip the implementation, do not repeat the skill name and leave out words like powerful, comprehensive, seamless, advanced or robust. Count the characters before saving |
| `platforms` | `[macos]`, `[linux]` or `[windows]`, and only when the skill uses something OS-bound. Omit it for portable skills |

### Body Sections (In Order)

1. `# <Human Title>` and a short intro: what it does, what it does not do, what it assumes is installed
2. `## When to Use`, with concrete trigger phrases
3. `## Prerequisites`: exact environment variables, credentials and install steps, or `None`
4. `## How to Run`: the canonical workflow, framed through the tools the model actually has
5. `## Quick Reference`: flat list of commands, routes, files or APIs
6. `## Procedure`: numbered steps, copy-paste exact
7. `## Pitfalls`: known limits and failure modes
8. `## Verification`: one focused check that proves the skill works

### Framing the Tools

Name tools in backticks, `run_command`, `write_file` and `view_skill` among them, and describe shell work as run through `run_command`. Prefer the read and search tools over raw shell utilities where one exists. Third-party CLIs are fine inside a procedure as long as it is clear the agent invokes them through `run_command` [1].

### Quality Bar

- Use commands, routes, paths, function names, config keys and error text exactly as they appear in the source; invented flags or APIs are the main way a skill goes wrong
- Keep `SKILL.md` scannable, roughly 100 lines for a simple workflow and 200 for a complex one
- Put the bulk elsewhere: larger scripts in `scripts/`, detailed docs in `references/`, reusable outputs in `templates/`, binary or visual assets in `assets/`
- A skill that only points at other skills is worth nothing; write the one that does the work [1]

---

### Augmentation: Invocation Policy Configuration

Some platforms support an optional `agents/<vendor>.yaml` metadata file to configure invocation policy:

```yaml
policy:
  allow_implicit_invocation: false  # Disable automatic skill selection
```

When set to `false`, the skill cannot be activated automatically based on prompt matching—only explicit invocation (e.g., `$skill-name`) works. OWUI does not implement this configuration; all skills may be auto-selected by the model based on their description [1, 2].

---

## 8. Use Cases

### Code Review Standards

Write your team's review checklist as a skill: naming conventions, error handling patterns, test coverage requirements. Attach it to your coding models so every review follows the same bar [1].

### Writing Style Guide

Document tone, formatting rules, and terminology in a skill. Attach it to content-writing models. Every draft follows your brand voice [1].

### Troubleshooting Playbooks

Encode your runbook for common issues: "check logs first, verify config, test connectivity, escalate if X." The model follows the same diagnostic steps your senior engineers would [1].

### Tool Usage Instructions

Pair a skill with Open Terminal to teach the model *how* to use it well. "Always check exit codes. Use `set -e` in scripts. Stream output for commands that take more than 10 seconds" [1].

---

## 9. Limitations

### Plain Text Only

Open WebUI stores a skill's scripts for the model to read. Running one takes a tool that executes code, such as **Open Terminal** or the code interpreter. For computation, API calls or system access, use **Tools** (executable Python scripts) [1].

### Context Window with $ Mention

Selected skills go into the system prompt with every message. A very long skill on a model with a small context window can crowd out the conversation [1].

---

### Augmentation: Context Protection and Deduplication

Cross-platform best practices for managing skill context include:

- **Protect skill content from context compaction**: If the agent truncates older messages, exempt skill content from pruning. Skill instructions are durable behavioral guidance—losing them silently degrades performance [2].
- **Deduplicate activations**: Track which skills have been activated in the session. If the model or user attempts to load a skill already in context, skip re-injection to avoid duplicate instructions [2].

OWUI does not implement automatic context protection; selected skills consume context with every message. The `$` mention mechanism injects full skill content into the system prompt regardless of prior activation [1].

---

### Lazy Loading Requires Function Calling

The manifest and the skill tools need **native function calling** with **Builtin Tools** enabled [1].

---

### Augmentation: Permission Allowlisting for Skill Resources

Some implementations implement permission allowlisting for skill directories, so the model can read bundled resources (scripts, references) without triggering user confirmation prompts. Without this, every reference to a bundled file results in a permission dialog [2].

OWUI's access control system applies at the skill level rather than per-file; once a user has read access to a skill, `read_skill_file` can access all files in that skill's directory [1].

---

## 10. Terminology Cheat Sheet

| Term | Definition |
| :--- | :--- |
| **Skill** | A reusable, markdown-based instruction set that the model reads and follows |
| **Tool** | An executable Python script that performs actions (distinct from Skills) |
| **SKILL.md** | The primary Markdown file containing a skill's instructions |
| **Manifest** | A list of available skills by name and description used for lazy loading |
| **Lazy loading** | On-demand loading of skill content when the model needs it via `view_skill` |
| **Terminal skill** | A skill served from a connected terminal server, with id prefixed `terminal:` |
| **Model-attached skill** | A skill bound to a model so new chats start with it selected |
| **/$ mention** | Chat input `$` prefix that injects a skill's instructions into the system prompt |
| **Version history** | Snapshot of a skill's name, description, translations and files on each save |
| **Production version** | The current active version of a skill that can be set from any stored version |

---

## 11. Planned Enrichment

This document provides the baseline OWUI Skills specification.

---

### Augmentation Summary (v2.0)

**Sections augmented:**
- **Section 2**: Added progressive disclosure pattern table and context budget management practices from cross-platform implementations
- **Section 3**: Added cross-platform discovery conventions and trust considerations for project-level skills
- **Section 4**: Added frontmatter leniency guidance for authoring cross-platform compatible skills
- **Section 7**: Added additional frontmatter fields from the broader specification and invocation policy configuration
- **Section 9**: Added context protection/deduplication best practices and permission allowlisting considerations

**Deliberately excluded:**
- Filesystem-based skill scanning (OWUI uses central workspace management)
- Automatic description shortening or skill omission (OWUI lists all skills in manifest)
- Environment requirements metadata (`compatibility` field) not processed by OWUI
- `agents/<vendor>.yaml` configuration files (not supported by OWUI)
- Permission allowlisting at per-file level (OWUI uses skill-level access control)

These excluded features depend on mechanisms OWUI lacks or replace OWUI's explicit user-centric workflows (e.g., import preview, workspace-based management) that prioritize user control over automation.

---

**Remaining work:**

1. Distill best practices specific to OWUI's architecture
2. Document real-world patterns for skill composition, lazy loading strategies, and terminal server integrations
3. Add examples of complex multi-file skills and versioning workflows

---

## Sources

[1] Open WebUI Documentation: "Skills" feature (`OWUI-skills.md`, attached)

[2] Agent Skills Documentation Knowledge Base: Multiple files including `overview.md`, `adding-skills-support.md`, `claude-overview.md`, and `build-skills.md` from the Agent Skills ecosystem documentation
