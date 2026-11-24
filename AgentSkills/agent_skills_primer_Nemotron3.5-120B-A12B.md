# Open WebUI Skills Primer (v2)

## 1. What a Skill Is

Skills are reusable, markdown-based instruction sets that you attach to models or invoke on-the-fly in chat. Unlike Tools (executable Python scripts), Skills are plain-text instructions: code review guidelines, writing style rules, troubleshooting playbooks, data analysis workflows. The model reads them and follows them [1].

A skill is a small folder: `SKILL.md` at its root, plus any supporting files the instructions refer to, such as scripts in `scripts/`, detailed docs in `references/`, reusable outputs in `templates/`, and images or other assets in `assets/` [1]. `SKILL.md` stays at the root and must be UTF-8 text [1].

Key characteristics:
- Teach your AI how to approach a task with plain-text instructions [1].
- No Python, no API calls, no deployment required to write one [1].
- Mention a skill with `$` in chat, or bind it to a model so every new chat starts with it selected [1].
- Every other skill you can read stays available, and the model loads it when needed [1].
- Reusable across models: create one skill and attach it to every coding model; update the skill once, and every model gets the new version [1].
- Composable with tools: pair a skill with Open Terminal or any tool server. The skill teaches the model *how* to use the tool, while the tool provides the *capability* [1].

### Augmentation: Cross-platform context on what a skill is

The open Agent Skills standard (agentskills.io) defines a skill as a folder containing a `SKILL.md` file with metadata (`name` and `description`) and instructions, optionally bundled with scripts, references, templates, and other resources [5]. Anthropic describes skills as reusable, filesystem-based resources that give Claude domain-specific expertise, distinct from one-off conversation-level prompts [2]. These descriptions are conceptually compatible with the OWUI baseline: both treat skills as portable, reusable instruction packages that live alongside the work they describe, rather than as ephemeral chat prompts. The OWUI "small folder" model (`SKILL.md` plus supporting files) aligns with this cross-platform convention [5][2].

## 2. How Skills Are Delivered to the Model

### Delivery mechanisms

- **User-selected skills ($ mention):** Type `$` in the chat input, or use the `/` menu, and pick a skill. Its instructions go into the system prompt with every message. A message that is only a `$` mention is sent with the skill names as its text [1].
- **Per-chat skills (Integrations menu):** The **+** menu > **Skills** turns skills on for the current chat. A badge shows how many are on. Selected skills send their instructions with every message [1].
- **Model-attached skills:** A new chat with a single model starts with the model's skills selected [1].

### Skill tools

With native function calling, the Builtin Tools capability and the model's Skills category on, the model gets these tools. `/skills:create` needs them too [1].

| Tool | What it does |
| :--- | :--- |
| `view_skill` | Loads a skill's `SKILL.md` and the list of its files, up to the read limit. |
| `read_skill_file` | Reads one file of a skill from a given offset, up to the read limit per call. For a binary file it returns the size and a link to the skill editor. |
| `grep_skill_files` | Searches a skill's text files for a word, phrase or regular expression and returns up to 10 excerpts. |
| `update_skill_files` | Saves file changes as a new version of a skill the user can edit. Refused with a conflict when someone saved the skill in the meantime. |
| `create_skill` | Creates a new private skill in the workspace from the `SKILL.md` text and optional text files under `scripts/`, `references/`, `templates/` or `assets/`. Offered to admins and to users with the Skills Access permission. |

All five follow the skill's access grants. A user with write access to an inactive skill can still have the model load and read it [1].

### Lazy loading and the manifest

Skills that are not selected use lazy loading. With the skill tools, every other active skill you can read is listed in a manifest by name and description. The model loads one with `view_skill` when it needs it, so many skills cost little context. A skill you switch off in the chat stays in the manifest [1].

Without the skill tools there is no manifest. Selected, mentioned and model skills are injected up to the first 100,000 characters of `SKILL.md`, and model skills are injected even when switched off in the chat [1].

A `SKILL.md` longer than the read limit arrives as a preview, and the model reads on with `read_skill_file` [1].

### Load selected skills on demand

The setting **Load selected skills on demand** in Admin Settings > General (`ENABLE_SKILL_LAZY_LOADING`, off by default) lists selected and model skills in the manifest too, so they load only when needed. A skill mentioned with `$` in the chat is still sent with every message. The setting needs the skill tools [1].

### Read limits

Two settings in Admin Settings > General set how much of a skill the model gets at once [1]:

| Setting | Default | What it sets |
| :--- | :--- | :--- |
| Skill Read Limit | 20,000 | Most characters the model gets in one read |
| Skill Preview Length | 5,000 | Characters the model gets first when `SKILL.md` is longer than the read limit |

Leave a field empty for the default, or set `SKILL_CONTENT_MAX_CHARS` and `SKILL_PREVIEW_MAX_CHARS` [1].

### Augmentation: Progressive disclosure and context budgets

The open Agent Skills standard describes "progressive disclosure" as a three-tier loading strategy: (1) a catalog of name + description at session start, (2) full `SKILL.md` instructions when a skill is activated, and (3) resources (scripts, references, assets) only when referenced [4][5]. This maps directly onto OWUI's manifest → preview → read limit model: the manifest is the catalog, the preview is the first slice of instructions, and `read_skill_file` pages through the rest [1][4].

Anthropic's Claude skills use the same tiered model: Level 1 metadata (`name` and `description` from YAML frontmatter) is always loaded at startup (~100 tokens per skill), Level 2 instructions (`SKILL.md` body) load when triggered (under 5k tokens), and Level 3 resources load as needed [2]. This reinforces OWUI's design where the description drives whether the model loads the full instructions [1].

OpenAI Codex applies a context budget to the initial skills list: at most 2% of the model's context window, or 8,000 characters when the context window is unknown, configurable via `skills.max_context_tokens` up to 10,000 tokens [6]. This is conceptually compatible with OWUI's read limits (20,000 characters default, 5,000 preview) [1], though the mechanisms differ (token budget vs. character limit).

## 3. Skills from a Terminal Server

A connected Open Terminal server can carry its own skills, kept in the workspace next to the files they describe. Open WebUI reads them from the server and offers them in the `$` picker beside your workspace skills, under ids of the form `terminal:<url-encoded name>` [1]. They behave like any other skill: `$` mention injects the content, otherwise they sit in the same manifest and load through `view_skill` [1]. What gets injected carries the skill's directory and the files it ships with, so the model knows where to read them from in the workspace. `read_skill_file` pages through a terminal skill's `SKILL.md` and `grep_skill_files` searches it; its other files stay on the terminal, and the model opens them with the terminal's own file tools. The skill editor, version history and `update_skill_files` work on workspace skills [1].

Writing a terminal server of your own means answering two endpoints [1]:

| Endpoint | Returns |
| :--- | :--- |
| `GET /skills` | A JSON array, one entry per skill, each with `id` (prefixed `terminal:`, the name url-encoded), `name`, `description` and `location` (or `path`). |
| `GET /skills/read?name=<name>` | One skill: `name`, `description`, `content` (the Markdown instructions), `location` (or `path`) and `resources`, an array of file paths shipped with the skill. |

Open Terminal serves both from v0.13.0 [1].

A terminal an admin configured is called by the Open WebUI server, with the connection's auth, `Accept: application/json`, `X-User-Id`, `X-Session-Id` (the chat id) and `X-Terminal-Context-Id`, the same identity headers as every other terminal call, so a server that scopes skills per user or per chat has what it needs. Redirects are not followed. A terminal a user added in their own settings is called from that user's browser, with the bearer key and `X-Session-Id`. It only has to be reachable from the browser, so a terminal on the user's own machine, such as `http://127.0.0.1:8000`, works with Open WebUI hosted elsewhere [1].

No compatible external augmentation found for this section. OWUI's terminal-server skill model is specific to Open WebUI and has no direct counterpart in the reviewed external documentation.

## 4. Creating and Managing Skills

### Creating a skill

Navigate to **Workspace > Skills** and click **Create** in the Workspace header [1].

| Field | Description |
| :--- | :--- |
| Name | Human-readable display name (e.g., "Code Review Guidelines") |
| Skill ID | Unique slug, auto-generated from the name. Editable during creation, read-only afterwards |
| Description | Short summary shown in the manifest. For model-attached skills, the model uses this to decide whether to load the full instructions |
| Files | `SKILL.md` with the full skill instructions in Markdown, plus any supporting files |

The globe selector beside the name translates the Name and Description per language, so a skill reads in the user's own language wherever it is listed [1].

While you create a skill, YAML frontmatter typed or pasted into `SKILL.md` fills in an empty Name and Description: `name: code-review-guidelines` becomes the name "Code Review Guidelines". The frontmatter is parsed as real YAML, so quoted and multi-line values come through as written [1].

Add a short note in **Describe this change** beside **Save** to label the version the save creates [1].

### Supporting files

Limits [1]:

| Limit | Value |
| :--- | :--- |
| Size of one file | 10 MiB |
| Size of all files in one skill | 50 MiB |
| Files per skill | 1,000 |

Text files are stored as text and any other file as binary. All file changes land together when you click **Save** [1]. The file panel in the editor lists files as a tree. Its **...** menu offers New File, New Folder, Upload and Upload Folder. Entries can be renamed, deleted or dragged into another folder, and the open file has a Download button. A text file also has Copy, which copies its content, and, when you can edit the skill, Compare revision [1].

### Importing skills

The chevron menu beside **Create** has three import entries [1]:

- **Import** takes one or more `.zip`, `.skill`, `.json` or `.md` files, up to 200 MiB per import. A `.skill` file is a ZIP and is read the same way. A ZIP holds a single skill with `SKILL.md` at its root, or several skill folders that each contain a `SKILL.md`. A `.md` file becomes a skill with that file as its `SKILL.md`. A JSON file is a skill export from Open WebUI, older exports with a single `content` field included. A ZIP or `.skill` file that contains a symbolic link or another special file is refused with `Archives cannot contain links or special files`. Needs the Skills Access and Import Skills permissions [1].
- **Import from URL** (admins only) downloads skills from a link [1].
- **Import folder** takes a skill folder from your computer, or a folder holding several skill folders, each with its own `SKILL.md`. Needs the Skills Access and Import Skills permissions [1].

Every import opens a preview listing each skill with its name, ID and files. For each one, choose Create, Create copy, Replace or Skip. A skill whose ID or name already exists starts on Skip; give it a unique ID and name to create it as a copy. Replace appears for existing skills you can edit, and saves the import as a new version of that skill with its sharing unchanged. New skills and copies start private [1].

### Importing from Markdown

Skills imported from a ZIP, a folder or a `.md` file take their name and description from the `SKILL.md` frontmatter, and the ID from the name; a JSON export carries its own [1]:

```yaml
---
name: code-review-guidelines
description: Step-by-step instructions for thorough code reviews
---

# Code Review Guidelines

1. Check for correctness...
```

### Importing from a URL

**Import from URL** in the chevron menu beside **Create** loads skills straight from the web. It is available to administrators [1]. Paste one of these links [1]:

| Link | What gets imported |
| :--- | :--- |
| A GitHub repository, such as `https://github.com/owner/repo` | Every skill in the repository: each folder that holds a `SKILL.md`, or the whole repository when `SKILL.md` sits at its root |
| A GitHub folder, `https://github.com/owner/repo/tree/<branch>/<folder>` | The skills in that folder |
| A GitHub `SKILL.md`, `https://github.com/owner/repo/blob/<branch>/<folder>/SKILL.md` | The skill in that `SKILL.md`'s folder, with all its files |
| A direct link to a `.zip`, `.skill`, `.json` or `.md` file | The skills in that file, read the same way as an upload |

GitHub links are downloaded as a ZIP of the repository at that branch, tag or commit, so a skill arrives with its supporting files. A branch name containing `/` has to be URL-encoded (`%2F`), or use a commit link. A ZIP or `.skill` file downloaded from any link skips symbolic links and other special files outside the skill folders, so a repository with such links elsewhere still imports. A symbolic link or special file inside a skill folder refuses the import with `Archives cannot contain links or special files`. The skills found open in the same import preview as an upload, where you choose what to create [1].

The link is checked against the same address rules as web fetching before the download starts and again at every redirect, so with `ENABLE_LOCAL_WEB_FETCH` off a link that leads into your private network is refused. It follows up to 5 redirects, gives up after 60 seconds and stops at 200 MiB. A link that returns a web page is refused with a hint to use a raw file or a ZIP download link; on GitHub, the repository, folder and `SKILL.md` links above work as they are [1].

### From a chat with `/skills:create`

A workflow you just worked through in a chat can be turned into a skill without writing it out yourself. Type `/skills:create` in that chat, optionally followed by what the skill should cover, and the model gathers the material, authors one `SKILL.md` to the standard below and saves it to your Workspace > Skills with the `create_skill` tool, along with any supporting text files under `scripts/`, `references/`, `templates/` or `assets/`. The new skill is private. The model reports the name, the location and a one-line summary when it is done, and can revise an existing skill you can edit with `update_skill_files` [1].

Anything you write after the command is treated as authoring guidance, all of it. Sources to pull from (paths, URLs, "what we just did", pasted notes) and requirements that shape the result (focus, exclusions, naming, style) can be mixed freely in one request [1].

The command needs [1]:
- the Skills Access workspace permission (admins always have it),
- a model with built-in tools active and its Skills category switched on under Builtin Tools,
- a chat that already has content, since the chat is the raw material.

The server checks all three before the model may save anything. The `/` menu offers the command once the chat has content and you hold the permission [1].

### Augmentation: Skill creation from a demonstrated workflow

OpenAI Codex offers a comparable creation path: a built-in `@skill-creator` (ChatGPT Work) or `$skill-creator` (Codex) command that asks what the skill does, when it should trigger, and whether it should include scripts, then drafts a `SKILL.md` [6]. OpenAI also offers a "Record & Replay" feature that captures a demonstrated workflow and drafts a reusable skill from it [6]. These are conceptually compatible with OWUI's `/skills:create`, which similarly turns a worked-through chat into a `SKILL.md` using `create_skill` [1]. The OWUI baseline does not include record-and-replay; that mechanism is excluded because OWUI has no equivalent feature.

### Augmentation: Cross-client skill directory conventions

The open Agent Skills standard and its adoption guide describe a `.agents/skills/` directory convention for cross-client skill sharing: skills installed by one compliant client are visible to another when both scan `.agents/skills/` at project or user level [4][5][7]. OWUI's workspace skills are stored in its own workspace rather than on the local filesystem, so this convention does not map to an OWUI feature. It is noted here only as context: OWUI skills are managed through its Workspace UI and API, not through `.agents/skills/` scanning [1].

### Management actions

From the Skills workspace list, use the ellipsis menu (**...**) on a skill you can edit [1]:

| Action | Description |
| :--- | :--- |
| Edit | Open the skill in the editor |
| Access | Open the skill's access settings straight from the list. Changes save as you make them. Shown to non-admins only with the Skills Sharing permission |
| Clone | Create a private copy of the current version, with all its files, and open it in the editor. The copy's name and ID get a short random suffix |
| Export .skill | Download the skill with all its files as one `.skill` file, a ZIP holding the skill's folder. On import, the name and description come from the `SKILL.md` frontmatter and the ID from that name; tags stay behind. Use Export JSON to keep the ID, name, description and tags. Needs the Skills Access and Export Skills permissions |
| Export JSON / Export ZIP | Download the skill with all its files. Needs the Skills Access and Export Skills permissions |
| Share | Publish the skill with all its files on openwebui.com. Needs Community Sharing on and the Export Skills permission |
| Delete | Permanently remove the skill and its whole version history (Shift+Click for quick deletion) |

**Bulk export:** Click Export ZIP or Export JSON in the chevron menu beside Create to export all accessible skills as one file. The ZIP holds one folder per skill. Both need the Skills Access and Export Skills permissions [1].

**Active/Inactive toggle:** Inactive skills are left out of manifests and `$` mentions, even when bound to a model. Users with write access can still have the model load them [1].

### Version history

Every save that changes a skill's name, description, translations or files stores a complete snapshot of the skill as a new version. Switching a skill on or off and changing its access leave the history as it is. Skills that existed before the upgrade start with one version holding their current content [1].

The version menu at the top of the editor's file panel lists Current and every earlier version, newest first and 20 per page, each with its author and the note from Describe this change (or a short version ID when there is none). Picking an earlier version opens it read-only with two actions [1]:

- Compare to current shows what differs between that version and the current one: name, description and translations, plus every added, deleted or modified file with a line-by-line diff for text files.
- Set as Production makes that version the current one again. It needs write access. If someone saved the skill after you opened the version, the switch is refused and the editor offers to reload the latest state.

The **...** beside each earlier version holds Delete, for users with write access. Versions that were based on a deleted one are linked to its parent, so the chain stays intact. With the Export Skills permission, the file panel's **...** menu also exports the version you are viewing as `.skill`, JSON or ZIP [1].

If someone else saves the skill while you edit it, your save is refused and your draft stays in the editor. A banner offers Reload latest, Save draft as copy or Keep editing [1].

## 5. Access Control

Skills use the same Access Control system as other workspace resources [1]:

- **Private by default:** Only the creator can see and edit a new skill.
- **Share with users or groups:** Grant `read` or `write` access via the Access button in the editor or the Access entry in the list menu. Non-admins see both only with the Skills Sharing permission.
- **Read-only access:** Users with read access can view but not edit. The editor shows a "Read Only" badge.

**Attached skills still require user access:** Attaching a skill to a model does not bypass access control. When a user chats with the model, Open WebUI checks whether that user has read access to each attached skill. Skills the user can't access are silently excluded [1].

Example: An admin creates a private skill and attaches it to a shared model. Regular users chatting with this model will not get the skill because they don't have read access [1].

Solution: Make sure users who need the model's skills also have read access to each skill (via access grants, group permissions, or by making the skill public) [1].

### Required permissions

| Permission | What it controls |
| :--- | :--- |
| Workspace > Skills Access | Access the Skills workspace and create/manage skills |
| Workspace > Import Skills | Import skills from ZIP, `.skill`, folder, JSON or Markdown, and open skills from openwebui.com |
| Workspace > Export Skills | Export skills and their versions as `.skill`, ZIP or JSON, and share skills to openwebui.com |
| Sharing > Skills Sharing | Share skills with individual users or groups |
| Sharing > Skills Public Sharing | Make skills publicly accessible |

No compatible external augmentation found for this section. OWUI's role-based access control model for skills is specific to Open WebUI and has no direct counterpart in the reviewed external documentation.

## 6. Skills API (overview)

The editor's file, version, import and export features are backed by these endpoints under `/api/v1/skills`. Reads need read access to the skill; changing the current version and deleting a version need write access [1].

| Endpoint | What it does |
| :--- | :--- |
| `GET /id/{id}/files` | Lists the skill's files with path, size and encoding. `version_id` picks a historical version. |
| `GET /id/{id}/files/content` | Downloads one file, given `path` and `version_id`. |
| `GET /id/{id}/history` | Lists versions, newest first, 20 per `page`. |
| `GET /id/{id}/history/{history_id}` | Returns one version with its file list. |
| `GET /id/{id}/history/diff` | Lists what changed between `from_id` and `to_id`: metadata and added, deleted or modified files. `/history/diff/file` with a `path` returns the line diff of one text file. |
| `POST /id/{id}/update/version` | Makes a stored version the current one. The body carries `version_id` and `expected_version_id`, the version you believe is current; when that no longer matches, the call returns `409`. |
| `DELETE /id/{id}/history/{history_id}` | Deletes a version other than the current one and links its child versions to its parent. |
| `POST /id/{id}/clone` | Copies the skill under a new `id` and `name`, optionally from a given `version_id`. The copy is private. |
| `GET /export` | Exports skills as `format=json`, `format=zip` or `format=skill`. Repeat `ids` to pick skills; `format=skill` takes exactly one. With exactly one id, `version_id` exports a historical version. |
| `POST /load/url` | Admins only. Downloads the skills behind `{"url": "..."}` and returns them as skill packages, without saving anything. |
| `POST /import/preview` | Reads uploaded ZIP, `.skill`, JSON or Markdown files (up to 200 MiB in total) and lists the skills found, flagging IDs and names that already exist. |
| `POST /import` | Imports the same uploads with a `decisions` list, one entry per skill: `action` (`create`, `copy`, `replace` or `skip`), an optional `id` and `name`, plus `expected_version_id` for `replace`. |

`POST /create` and `POST /id/{id}/update` take the skill's files as `files`, and an optional `commit_message` that labels the new version. An update can send `operations` (`put`, `move`, `delete`) to change single files. An update that sends `files` or `operations` needs `expected_version_id` and returns `409` when someone saved the skill in the meantime [1].

### Augmentation: Cross-platform skill management APIs

Anthropic's Claude API provides a Skills API with CRUD operations and version management (list, create, versions list) [3]. This is conceptually compatible with OWUI's `/api/v1/skills` endpoints, which also cover file listing, version history, diff, clone, export, and import [1]. The two APIs are independent implementations; OWUI's endpoint paths, parameters, and conflict-handling (`409` on `expected_version_id` mismatch) are specific to Open WebUI and are not mirrored in the external documentation [1][3].

## 7. The OWUI Skill Authoring Standard

This is the standard Open WebUI itself follows when it writes a skill through `/skills:create`, and the one to hold your own skills to [1].

### Frontmatter

| Key | Rule |
| :--- | :--- |
| `name` | Lowercase and hyphenated, no spaces, 64 characters at most. |
| `description` | A clear description, 1024 characters at most, ending in a period. Name the capability, skip the implementation, do not repeat the skill name and leave out words like powerful, comprehensive, seamless, advanced or robust. Count the characters before saving. |
| `platforms` | `[macos]`, `[linux]` or `[windows]`, and only when the skill uses something OS-bound. Omit it for portable skills. |

### Body sections, in this order

1. `# <Human Title>` and a short intro: what it does, what it does not do, what it assumes is installed.
2. `## When to Use`, with concrete trigger phrases.
3. `## Prerequisites`: exact environment variables, credentials and install steps, or `None`.
4. `## How to Run`: the canonical workflow, framed through the tools the model actually has.
5. `## Quick Reference`: flat list of commands, routes, files or APIs.
6. `## Procedure`: numbered steps, copy-paste exact.
7. `## Pitfalls`: known limits and failure modes.
8. `## Verification`: one focused check that proves the skill works.

### Framing the tools

Name tools in backticks, `run_command`, `write_file` and `view_skill` among them, and describe shell work as run through `run_command`. Prefer the read and search tools over raw shell utilities where one exists. Third-party CLIs are fine inside a procedure as long as it is clear the agent invokes them through `run_command` [1].

### Quality bar

Use commands, routes, paths, function names, config keys and error text exactly as they appear in the source; invented flags or APIs are the main way a skill goes wrong. Keep `SKILL.md` scannable, roughly 100 lines for a simple workflow and 200 for a complex one, and put the bulk elsewhere: larger scripts in `scripts/`, detailed docs in `references/`, reusable outputs in `templates/`, binary or visual assets in `assets/`. A skill that only points at other skills is worth nothing; write the one that does the work [1].

### Augmentation: Frontmatter and description guidance from other platforms

The open Agent Skills standard requires `name` and `description` in `SKILL.md` YAML frontmatter, where `description` tells the agent when to use the skill [5][7]. Anthropic's Claude skills use the same frontmatter pattern: `name` and `description` in YAML, with the description serving as the match criterion for skill activation [2]. This is directly compatible with OWUI's frontmatter rules, where `name` (lowercase, hyphenated, max 64 chars) and `description` (max 1024 chars, ending in a period) drive manifest listing and model-attached skill selection [1][2][5].

OWUI adds a `platforms` frontmatter field (`[macos]`, `[linux]`, `[windows]`) that is not part of the open Agent Skills standard [1][5]. This field is OWUI-specific and should not be assumed to exist in other platforms' skills.

### Augmentation: Body section alignment with progressive disclosure

OWUI's ordered body sections (`When to Use`, `Prerequisites`, `How to Run`, `Quick Reference`, `Procedure`, `Pitfalls`, `Verification`) operationalize the progressive disclosure model: the `When to Use` section with trigger phrases supports the catalog/activation decision, while `Procedure` and `Verification` form the core instructions loaded on activation [1][4][5]. This aligns with the open standard's three-tier model where the description drives activation and the full body loads only when needed [4][5].

## 8. Use Cases

- **Code review standards:** Write your team's review checklist as a skill: naming conventions, error handling patterns, test coverage requirements. Attach it to your coding models so every review follows the same bar [1].
- **Writing style guide:** Document tone, formatting rules, and terminology in a skill. Attach it to content-writing models. Every draft follows your brand voice [1].
- **Troubleshooting playbooks:** Encode your runbook for common issues: "check logs first, verify config, test connectivity, escalate if X." The model follows the same diagnostic steps your senior engineers would [1].
- **Tool usage instructions:** Pair a skill with Open Terminal to teach the model how to use it well. "Always check exit codes. Use `set -e` in scripts. Stream output for commands that take more than 10 seconds." [1]

### Augmentation: Document-task skills from other platforms

Anthropic provides pre-built Agent Skills for common document tasks: PowerPoint (pptx), Excel (xlsx), Word (docx), and PDF (pdf) [2]. These are conceptually compatible with OWUI's use cases as examples of skills that package domain-specific expertise for repeatable tasks [1][2]. OWUI's baseline does not include pre-built document-task skills; this is noted as cross-platform context only, not as an OWUI feature.

The open skills repository (github.com/anthropics/skills) hosts example skills that can serve as authoring references [7]. This is compatible with OWUI's authoring standard as a source of exemplars, though OWUI has no built-in skill catalog equivalent [1][7].

## 9. Limitations

- **Plain text only:** Open WebUI stores a skill's scripts for the model to read. Running one takes a tool that executes code, such as Open Terminal or the code interpreter. For computation, API calls or system access, use Tools [1].
- **Context window with $ mention:** Selected skills go into the system prompt with every message. A very long skill on a model with a small context window can crowd out the conversation [1].
- **Lazy loading requires function calling:** The manifest and the skill tools need native function calling with Builtin Tools enabled [1].

No compatible external augmentation found for this section. OWUI's limitations are specific to its architecture and are not directly mirrored in the reviewed external documentation.

## 10. Terminology Cheat Sheet

| Term | Meaning |
| :--- | :--- |
| Skill | A reusable, markdown-based instruction set attached to models or invoked on-the-fly in chat; plain-text, not executable code [1]. |
| `SKILL.md` | The root Markdown file of a skill, must be UTF-8 text [1]. |
| Skill tools | Native functions `view_skill`, `read_skill_file`, `grep_skill_files`, `update_skill_files`, `create_skill` [1]. |
| Manifest | A list of active, readable skills by name and description, used for lazy loading [1]. |
| Lazy loading | Loading a skill's instructions only when the model needs them, via `view_skill` [1]. |
| Read limit | Max characters the model gets in one read (default 20,000) [1]. |
| Skill preview length | Characters the model gets first when `SKILL.md` exceeds the read limit (default 5,000) [1]. |
| Terminal skill | A skill served by an Open Terminal server, addressed as `terminal:<url-encoded name>` [1]. |
| `/skills:create` | Chat command that turns a worked-through workflow into a skill using `create_skill` [1]. |
| Version history | Snapshots stored on every save that changes name, description, translations or files [1]. |
| Set as Production | Makes an earlier historical version the current one again [1]. |
| Access control | Private by default; read/write grants via Access button; attached skills still require user read access [1]. |

### Augmentation: Cross-platform terminology

| Term | Meaning |
| :--- | :--- |
| Progressive disclosure | Loading only skill metadata (name + description) at session start, then the full instructions on activation, then resources as needed [4][5]. |
| Catalog | The initial list of available skills with name and description, shown to the model before any skill is activated [4][5]. |
| Activation | The point at which a model reads a skill's full `SKILL.md` body into context because the task matches the skill's description [4][5]. |
| Tiered loading | The three-stage loading pattern (catalog → instructions → resources) used by skills-compatible agents [4][5]. |

## 11. Planned Enrichment

This primer (v2) establishes a shared, accurate baseline of the OWUI Skills feature, augmented in this pass with cross-platform context from the Agent Skills standard and other LLM companies' implementations [1][2][3][4][5][6][7].

**What was augmented in this pass:**
- Section 1: Cross-platform definition of skills as filesystem-based, reusable instruction packages [2][5].
- Section 2: Progressive disclosure tiers, context budgets, and the `.agents/skills/` convention [2][4][5][6].
- Section 4: Skill creation from a demonstrated workflow (OpenAI `@skill-creator`/`$skill-creator`) and cross-client directory conventions [6][7].
- Section 6: Cross-platform skill management APIs (Anthropic Skills API) [3].
- Section 7: Frontmatter alignment (name/description in YAML) and body section alignment with progressive disclosure [2][5].
- Section 8: Document-task skills (Anthropic pre-built skills) and the open skills repository [2][7].
- Section 10: Cross-platform terminology (progressive disclosure, catalog, activation, tiered loading) [4][5].

**What was deliberately excluded:**
- OpenAI's skill scopes (REPO, USER, ADMIN, SYSTEM) — OWUI uses a workspace-based model, not filesystem scoping [6].
- OpenAI's Record & Replay and `$skill-installer` — OWUI has no equivalent features [6].
- Anthropic's code execution / VM environment for skills — OWUI skills are plain-text instructions, not executable code in a VM [2][3].
- Anthropic's 20-skill-per-request limit and `container` parameter — OWUI has no equivalent API construct [3].
- Claude Code's bundled skills, invocation control, and subagent execution — OWUI-specific features not present in OWUI [8].
- Skill eval frameworks (evals.json, assertions, grading) — outside the scope of this baseline primer [9].

**What remains:**
- A distilled OWUI-specific best-practices checklist, to be developed in a later pass comparing OWUI's authoring standard against other platforms' guidance.

---

## Sources

[1] Open WebUI documentation, "Skills" (OWUI-skills.md). Provided as context for this primer.
[2] Anthropic, "Agent Skills" overview (claude-overview.md). Knowledge base: Agent Skills Documentation.
[3] Anthropic, "Using Agent Skills with the API" (claude-skills-guide.md). Knowledge base: Agent Skills Documentation.
[4] Agent Skills standard, "How to add skills support to your agent" (adding-skills-support.md). Knowledge base: Agent Skills Documentation.
[5] Agent Skills standard, "Agent Skills Overview" (overview.md). Knowledge base: Agent Skills Documentation.
[6] OpenAI, "Build skills" (build-skills.md). Knowledge base: Agent Skills Documentation.
[7] Agent Skills standard, "Quickstart" (quickstart.md). Knowledge base: Agent Skills Documentation.
[8] Anthropic, "Extend Claude with skills" (claude-skills.md). Knowledge base: Agent Skills Documentation.
[9] Agent Skills standard, "Evaluating skill output quality" (evaluating-skills.md). Knowledge base: Agent Skills Documentation.