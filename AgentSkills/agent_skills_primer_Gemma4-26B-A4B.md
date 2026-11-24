## 1. What a Skill Is
Skills are reusable, markdown-based instruction sets that can be attached to models or invoked on-the-fly in chat [1]. Unlike Tools (which are executable Python scripts), Skills consist of plain-text instructions such as code review guidelines, writing style rules, troubleshooting playbooks, or data analysis workflows [1]. The model reads and follows these instructions [1].

## 2. How Skills Are Delivered to the Model
Skills are delivered through several mechanisms [1]:
- **User-selected skills ($ mention):** Typing `$` in the chat input or using the `/` menu allows a user to pick a skill. Its instructions are injected into the system prompt with every message [1]. A message consisting only of a `$` mention is sent with the skill names as its text [1].
- **Per-chat skills (Integrations menu):** Selecting skills via the **+** menu > **Skills** turns them on for the current chat. A badge indicates how many are active, and selected skills send their instructions with every message [1].
- **Model-attached skills:** Skills can be bound to a model so that every new chat starts with them selected [1].

**Lazy Loading and Skill Tools**
Skills not actively selected are listed in a manifest by name and description. If "Load selected skills on demand" is enabled in **Admin Settings > General** (`ENABLE_SKILL_LAZY_LOADING`), these skills load only when needed [1]. This requires native function calling and the **Builtin Tools** capability to be active [1]. The model can load a skill from the manifest using `view_skill` [1].

**Skill Tools**
| Tool | What it does |
| :--- | :--- |
| `view_skill` | Loads a skill's `SKILL.md` and the list of its files, up to the read limit [1]. |
| `read_skill_file` | Reads one file of a skill from a given offset, up to the read limit per call. For a binary file, it returns the size and a link to the skill editor [1]. |
| `grep_skill_files` | Searches a skill's text files for a word, phrase, or regular expression and returns up to 10 excerpts [1]. |
| `update_skill_files` | Saves file changes as a new version of a skill the user can edit. Refused with a conflict if someone saved the skill in the meantime [1]. |
| `create_skill` | Creates a new private skill in the workspace from the `SKILL.md` text and optional text files. Offered to admins and users with **Skills Access** permission [1]. |

**Read Limits**
The following settings in **Admin Settings > General** control how much of a skill the model receives [1]:
- **Skill Read Limit:** Default 20,000 characters (the most characters the model gets in one read) [1].
- **Skill Preview Length:** Default 5,000 characters (what the model gets first if `SKILL.md` is longer than the read limit) [1].
- **Injection Limit:** Selected, mentioned, and model skills are injected up to the first 100,000 characters of `SKILL.md` [1].

These can be configured via `SKILL_CONTENT_MAX_CHARS` and `SKILL_PREVIEW_MAX_CHARS` [1].

### Augmentation: Activation Strategies [2]
Implementation typically follows one of two patterns:
- **File-read activation:** The model uses its existing file-reading capabilities to read `SKILL.md` directly from a path in the catalog.
- **Dedicated tool activation:** A specialized tool is provided to return content. This allows for better control, such as wrapping content in structured tags, listing bundled resources, or enforcing permissions before returning data.

### Augmentation: Progressive Disclosure [2]
Skill delivery can be viewed as a form of progressive disclosure: the model first sees a catalog (manifest) of names and descriptions, and only upon selection does it receive the full instructional content.

### Augmentation: Catalog Integrity [2]
To prevent the model from wasting turns attempting to load unavailable skills, invalid or disabled skills should be filtered out of the manifest (catalog) entirely rather than being listed and blocked at activation time.

## 3. Skills from a Terminal Server
A connected Open Terminal server can carry its own skills, which are kept in the workspace next to the files they describe [1]. They appear in the `$` picker with IDs formatted as `terminal:<url-encoded name>` [1]. They behave like other skills: `$` mention injects content; otherwise, they sit in a manifest and load via `view_skill` [1].

Terminal skills support `read_skill_file` for paging through `SKILL.md` and `grep_skill_files` for searching [1]. Other files remain on the terminal, and the model opens them using the terminal's own file tools [1].

Writing a terminal server requires implementing these endpoints [1]:
| Endpoint | Returns |
| :--- | :--- |
| `GET /skills` | A JSON array of skills, each with `id` (prefixed `terminal:`), `name`, `description`, and `location` (or `path`) [1]. |
| `GET /skills/read?name=<name>` | One skill containing `name`, `description`, `content`, `location` (or `path`), and `resources` (an array of file paths) [1]. |

## 4. Creating and Managing Skills
**Creation**
Navigate to **Workspace > Skills** and click **Create** [1].
- **Name:** Human-readable display name [1].
- **Skill ID:** Unique slug, auto-generated from the name; editable during creation, read-only afterwards [1].
- **Description:** Short summary for the manifest [1].
- **Files:** `SKILL.md` (Markdown instructions) and any supporting files [1].

**Importing Skills**
The chevron menu beside **Create** provides three import options [1]:
- **Import:** Takes `.zip`, `.skill`, `.json`, or `.md` files (up to 200 MiB per import). A `.skill` file is a ZIP. A `.md` file becomes a skill where the file is the `SKILL.md`. JSON is a skill export [1].
- **Import from URL:** (Admins only) Downloads skills from a link (GitHub repos, folders, specific `SKILL.md`, or direct links to `.zip`/`.skill`/`.json`/`.md`) [1].
- **Import folder:** Takes a skill folder or a folder of skill folders from a computer [1].

**Creating via Chat (/skills:create)**
Typing `/skills:create` in a chat (optionally with authoring guidance) allows the model to gather material from the chat and save it as a new private skill using the `create_skill` tool [1]. This requires **Skills Access** permission, a model with built-in tools active and its **Skills** category switched on, and a chat that already has content [1].

**Management Actions**
From the ellipsis menu (**...**) on a skill you can edit [1]:
| Action | Description |
| :--- | :--- |
| **Edit** | Open the skill in the editor [1]. |
| **Access** | Open the skill's access settings [1]. |
| **Clone** | Create a private copy with a random suffix to the name and ID [1]. |
| **Export .skill** | Download as a `.skill` file (a ZIP) [1]. |
| **Export JSON / Export ZIP** | Download all files as JSON or ZIP [1]. |
| **Share** | Publish the skill on openwebui.com [1]. |
| **Delete** | Permanently remove the skill and version history [1]. |

Bulk export is available via the chevron menu beside **Create** [1].

**Version History**
Every save that changes a skill's name, description, translations, or files creates a new version [1]. The version menu allows you to:
- **Compare to current:** See differences (metadata and line-by-line diff for text) [1].
- **Set as Production:** Make an earlier version the current one [1].

## 5. Access Control
Skills use the standard Open WebUI access control system [1]:
- **Private by default:** Only the creator can see/edit [1].
- **Sharing:** Users can be granted `read` or `write` access via the **Access** button or list menu [1].
- **Read-only access:** Users with read access can view but not edit [1].

**Note:** Attaching a skill to a model does **not** bypass access control. If a user lacks read access to an attached skill, it is silently excluded from their chat [1].

**Required Permissions**
| Permission | What it controls |
| :--- | :--- |
| **Workspace > Skills Access** | Access the Skills workspace and create/manage skills [1]. |
| **Workspace > Import Skills** | Import skills and open skills from openwebui.com [1]. |
| **Workspace > Export Skills** | Export skills/versions and share to openwebui.com [1]. |
| **Sharing > Skills Sharing** | Share skills with individual users or groups [1]. |
| **Sharing > Skills Public Sharing** | Make skills publicly accessible [1]. |

### Augmentation: Proactive Filtering [2]
A key best practice is to hide filtered skills entirely from the catalog. This prevents the model from wasting turns attempting to load skills it cannot access or that have been disabled in settings.

## 6. Skills API
The editor's features are backed by endpoints under `/api/v1/skills` [1]:
- `GET /id/{id}/files`: Lists files (path, size, encoding) [1].
- `GET /id/{id}/files/content`: Downloads a file [1].
- `GET /id/{id}/history`: Lists versions [1].
- `GET /id/{id}/history/{history_id}`: Returns one version's file list [1].
- `GET /id/{id}/history/diff`: Lists changes between versions [1].
- `POST /id/{id}/update/version`: Makes a stored version current [1].
- `DELETE /id/{id}/history/{history_id}`: Deletes a version [1].
- `POST /id/{id}/clone`: Copies a skill [1].
- `GET /export`: Exports skills (JSON, ZIP, or `.skill`) [1].
- `POST /load/url`: (Admins only) Downloads skills from a URL without saving [1].
- `POST /import/preview`: Reads uploads and lists found skills [1].
- `POST /import`: Imports uploads with decisions (create, copy, replace, skip) [1].

## 7. The OWUI Skill Authoring Standard
**Frontmatter Rules**
- `name`: Lowercase, hyphenated, no spaces, max 64 characters [1].
- `description`: Clear description, max 1024 characters, ends in a period [1].
- `platforms`: `[macos]`, `[linux]`, or `[windows]` (omit if portable) [1].

**Body Section Order**
1. `# <Human Title>` and a short intro [1].
2. `## When to Use` (with trigger phrases) [1].
3. `## Prerequisites` (environment variables, credentials, etc.) [1].
4. `## How to Run` (canonical workflow) [1].
5. `## Quick Reference` (commands, routes, files, or APIs) [1].
6. `## Procedure` (numbered steps) [1].
7. `## Pitfalls` (limits and failure modes) [1].
8. `## Verification` (one focused check) [1].

**Quality Bar**
- Use exact names for commands, routes, paths, function names, config keys, and error text [1].
- Keep `SKILL.md` scannable (roughly 100 lines for simple workflows, 200 for complex) [1].
- Put larger content elsewhere (e.g., `scripts/`, `references/`, `templates/`, `assets/`) [1].
- Frame tools in backticks (e.g., `run_command`, `write_file`, `view_skill`) [1].

## 8. Use Cases
- **Code review standards:** Attach checklists to coding models [1].
- **Writing style guide:** Attach tone and formatting rules to content-writing models [1].
- **Troubleshooting playbooks:** Encode diagnostic runbooks [1].
- **Tool usage instructions:** Teach models how to use tools like Open Terminal (e.g., \"Always check exit codes\") [1].

## 9. Limitations
- **Plain text only:** Skills are instructions; execution requires tools like Open Terminal or the code interpreter [1].
- **Context window crowding:** Long skills injected via `$` can crowd out conversation in models with small context windows [1].
- **Lazy loading requirement:** Requires native function calling and **Builtin Tools** [1].

### Augmentation: Presence Awareness [2]
When no skills are available, it is better to omit the catalog and behavioral instructions entirely. Registering a skill tool with no valid options or providing an empty `<available_skills/>` block can confuse the model.

## 10. Terminology Cheat Sheet
| Term | Meaning |
| :--- | :--- |
| **Skill** | A reusable, markdown-based instruction set [1]. |
| **Tool** | An executable Python script [1]. |
| **$ Mention** | Injecting a skill's instructions into the current message [1]. |
| **Model Binding** | Attaching skills to a model for all new chats [1]. |
| **Lazy Loading** | Loading skills from a manifest only when needed [1]. |
| **Skill Tools** | Functions (`view_skill`, etc.) used by the model to interact with skills [1]. |
| **SKILL.md** | The required root Markdown file for a skill containing instructions [1]. |

## 11. Planned Enrichment
**Document Version: v2**

This document serves as the foundational baseline for the OWUI Skills feature. This version (v2) has been augmented with conceptual reinforcements from the Agent Skills specification [2].

**Augmentations in this pass:**
- **Delivery & Activation:** Added patterns for file-read vs. dedicated tool activation and the concept of progressive disclosure.
- **Catalog Management:** Added concepts of proactive filtering and catalog integrity to prevent model confusion.
- **Access Control:** Reinforced the practice of hiding inaccessible skills from the manifest.

**Deliberately excluded:**
- Platform-specific tool names (e.g., `activate_skill`) used as examples in external docs, to avoid confusion with OWUI's specific tools.
- Implementation details for specific host environments (local vs cloud discovery) that do not change the conceptual model of skill interaction.

**Remaining work:**
- A final pass will distill a set of OWUI-specific best practices by comparing these findings with other major LLM implementations.

Sources
[1] OWUI-skills.md
[2] adding-skills-support.md
