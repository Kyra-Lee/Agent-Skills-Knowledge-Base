# Open WebUI Skills: Design and Authoring Guide

> Compiled 2026-11-08. Written to be read as context by an LLM that is helping a person design Skills for Open WebUI (OWUI). It is also meant to be readable by a human reviewer. Every claim carries a status tag (section 0) so you can tell documented fact from inference.

---

## 0. How to use this guide

**Operating rules for the assistant reading this**

1. Run the intake in section 9 before drafting. Ask only what you cannot infer, and batch your questions.
2. Draft with the standard in section 4 and the template in section 11.
3. Check the draft against the checklist in section 12 before presenting it.
4. Never invent commands, flags, paths, tool names, config keys, endpoints or error text. OWUI's own standard says invented flags and APIs are the main way a skill goes wrong. If the user has not supplied the exact text, leave a marked placeholder and ask.
5. Report status honestly. Say what is documented, what you inferred, and what is unverified (section 14). Never present a `[DERIVED]` or `[OPEN]` item as established fact.
6. Prefer fewer, better instructions over exhaustive ones.

**Status tags**

| Tag | Meaning |
| :--- | :--- |
| `[OWUI]` | Stated in the OWUI Skills documentation this guide is built on. |
| `[OWUI-WEB]` | Stated on other docs.openwebui.com pages (Tools, Open Terminal, Advanced Workflows with Skills), fetched 2026-11-08. |
| `[SPEC]` | From the open Agent Skills specification or its authoring guides (agentskills.io). Applies to OWUI by analogy. |
| `[EXT]` | From other vendors' guidance (Anthropic, OpenAI, Nous Research Hermes). Applies to OWUI by analogy only. |
| `[DERIVED]` | Inferred by this guide's compiler from the sources above. Reasoned, but not stated anywhere. |
| `[OPEN]` | Unknown or undocumented. Verify in a live OWUI instance (section 14). |

**Precedence when sources disagree:** `[OWUI]` and `[OWUI-WEB]` win, then `[SPEC]`, then `[EXT]`. Conflicts are listed in section 14.2.

---

## 1. Rules at a glance

**Design**
1. Make a skill when the need is a procedure or knowledge the model lacks, expressible as instructions plus tools it already has. Use a Tool for custom code, auth handling or deterministic computation (section 2).
2. One skill, one coherent job (section 6).
3. Decide how the skill will be activated first: bound to a model, lazy-loaded by description, or `$`-mentioned (sections 3 and 9).

**Frontmatter**
4. `name`: lowercase, hyphenated, 64 characters at most. Use only letters, digits and hyphens, with no leading, trailing or doubled hyphens. Choose it before the first save, because the Skill ID cannot be edited afterwards (section 4).
5. `description`: 1024 characters at most, ends in a period. It states the capability, the trigger contexts and, where useful, a boundary. No first or second person, no filler adjectives, do not repeat the name (section 5).
6. Put trigger contexts in the description, not only in the body (section 5).
7. Quote YAML values that contain a colon followed by a space. After saving, confirm the Name and Description fields are populated and match the frontmatter (section 4).

**Body**
8. Use the eight-section order. Aim for roughly 100 lines for a simple skill and 200 for a complex one (section 4).
9. Write only what the model would not already know. Cut explanations of common knowledge (section 6).
10. Use commands, paths, flags, function names and error text exactly as they appear in the source (sections 4 and 6).
11. Match firmness to fragility. Give a default, not a menu (section 6).
12. Use numbered steps, state the expected output, and give a template for fixed formats (section 6).
13. Pitfalls are concrete and environment-specific: a rule plus a one-clause reason, with no incident stories (section 6).
14. Name tools in backticks, and only tools the target model actually has (section 4).

**Files**
15. Keep `SKILL.md` as the overview. Link each supporting file directly from it, with a condition for when to load it. Keep links one level deep (section 7).
16. Keep each reference file under the 20,000-character read limit, or give it a contents list (section 7).
17. If the target model may lack native function calling, make `SKILL.md` self-contained (sections 3 and 7).

**Scripts**
18. Prefer instructions to scripts unless you need determinism or external tooling. Running anything needs Open Terminal or the code interpreter (section 8).
19. How a workspace skill's script reaches an execution environment is undocumented. Do not assume it works; test it (sections 8 and 14).
20. Scripts must be non-interactive, document `--help`, fail with clear messages, print structured output, and bound their output size (section 8).

**Safety and testing**
21. No secrets in skills. Name required environment variables in Prerequisites. Treat imported skills as untrusted text (section 3.4).
22. Test triggering (should and should not fire) and output quality (with and without the skill), starting with the weakest model you intend to use (section 10).
23. Use "Describe this change" to record what each version changed and why (section 10).
24. Do not assume features from other platforms exist in OWUI (section 14.3).

---

## 2. What a skill is, and is not

- A skill is a reusable, markdown-based instruction set. The model reads it and follows it. You attach it to models or invoke it in chat with `$`. `[OWUI]`
- Physically it is a small folder: `SKILL.md` at the root (UTF-8 text, required), plus optional `scripts/`, `references/`, `templates/` and `assets/`. `[OWUI]`
- It is not code. OWUI stores a skill's scripts for the model to read. Running one needs a tool that executes code, such as Open Terminal or the code interpreter. For computation, API calls or system access, use Tools (executable Python). `[OWUI]`
- A skill teaches the model how to use a tool well; the tool provides the capability. `[OWUI]`
- Skills are reusable across models: update one skill and every model it is attached to gets the new version. `[OWUI]`

**Skill or tool?** `[EXT]` (Nous Research) A skill fits when the capability can be expressed as instructions plus shell commands plus tools the model already has, including wrapping a CLI or API it can call. A tool fits when it needs integrated key or auth handling, custom logic that must run precisely every time, or binary, streaming or real-time handling.

**When a skill is the wrong container** `[DERIVED]`
- The model already does the task well without it. Test before building (section 10). `[SPEC]` supports this check.
- The need is a one-off prompt rather than a repeatable procedure.
- The need is deterministic computation. Use a Tool.

---

## 3. How skills reach the model

### 3.1 Delivery paths `[OWUI]`

| Path | What the model receives | Context cost | Requires |
| :--- | :--- | :--- | :--- |
| `$` mention (or `/` menu) | `SKILL.md` instructions injected into the system prompt with every message. A message that is only a `$` mention is sent with the skill names as its text. | Full body, every message | Read access |
| Per-chat toggle (**+** menu > **Skills**) | Same as `$`: instructions sent with every message. | Full body, every message | Read access |
| Model binding (Workspace > Models > Skills row) | A new chat with a single model starts with that model's skills selected, so they behave like the rows above. | Full body, every message | The user must have read access to each attached skill |
| Lazy loading (manifest) | Every other active skill the user can read is listed by name and description. The model calls `view_skill` when it needs one. | Description only, until loaded | Skill tools (3.2) |
| Admin setting **Load selected skills on demand** (`ENABLE_SKILL_LAZY_LOADING`, off by default) | Selected and model-bound skills also go into the manifest and load on demand. A skill `$`-mentioned in the chat is still sent every message. | Lower | Skill tools |
| No skill tools available | No manifest. Selected, mentioned and model skills are injected, up to the first 100,000 characters of `SKILL.md`. Model skills are injected even when switched off in the chat. | Highest | |

An inactive skill is left out of manifests and `$` mentions, even when bound to a model. A user with write access to an inactive skill can still have the model load it. `[OWUI]`

### 3.2 Skill tools `[OWUI]`

The model gets these only with native function calling, the **Builtin Tools** capability and the model's **Skills** category all on. `/skills:create` needs them too.

| Tool | Behavior |
| :--- | :--- |
| `view_skill` | Loads a skill's `SKILL.md` and the list of its files, up to the read limit. |
| `read_skill_file` | Reads one file from a given offset, up to the read limit per call. For a binary file it returns the size and a link to the skill editor, not the content. |
| `grep_skill_files` | Searches a skill's text files for a word, phrase or regular expression. Returns up to 10 excerpts. |
| `update_skill_files` | Saves file changes as a new version of a skill the user can edit. Refused with a conflict if someone saved in the meantime. |
| `create_skill` | Creates a new private workspace skill from `SKILL.md` text plus optional text files. Offered to admins and users with Skills Access. |

All five follow the skill's access grants.

**Read limits** `[OWUI]`: the model gets at most 20,000 characters per read (admin setting **Skill Read Limit**, env `SKILL_CONTENT_MAX_CHARS`). When `SKILL.md` is longer, the model first gets a 5,000-character preview (**Skill Preview Length**, env `SKILL_PREVIEW_MAX_CHARS`) and reads on with `read_skill_file`.

### 3.3 Design consequences `[DERIVED]`

1. **The description is the routing key.** Before a skill loads, the model sees only its name and description. Section 5 is therefore the highest-leverage part of authoring.
2. **A bound skill costs its whole body on every message** (unless the admin lazy-loading setting is on). Bind only skills the model should follow in every chat. OWUI says the same: the manifest already lists everything the user can read.
3. **Without skill tools, supporting files are unreachable.** Only `SKILL.md` text is injected. If the target model may lack native function calling or Builtin Tools, make `SKILL.md` self-contained.
4. **Manifest cost grows with the number of skills.** Each description is in context on every message. `[SPEC]` puts the metadata cost at roughly 100 tokens per skill. Keep descriptions tight.
5. **Small-context models suffer first.** OWUI warns that a long selected skill on a small-context model can crowd out the conversation. `[OWUI]`
6. **Front-load `SKILL.md`.** If a body ever exceeds the read limit, the model starts from a 5,000-character preview. Put routing and the core procedure first.
7. **Inaccessible skills vanish silently.** Attaching a skill to a model does not bypass access control; users without read access get no skill and no error. `[OWUI]` Design the sharing along with the skill.

### 3.4 Access, sharing and secrets

- Skills are private by default. Access can be granted as `read` or `write` to users or groups, and made public with the right permissions. `[OWUI]`
- Skills can be exported, shared to openwebui.com (up to 15 MiB in total) and imported from ZIP, `.skill`, folder, JSON, Markdown or URL (URL import is admin-only). `[OWUI]`
- **No secrets in a skill.** `[DERIVED]` Anyone with read access, and anyone who receives an export or share, sees the text. Name the required environment variables or credentials in `## Prerequisites` instead. `[OWUI]` says Prerequisites holds exact environment variables, credentials and install steps.
- **Treat imported or community skills as untrusted instructions.** `[DERIVED]` Skill text goes into the model's context and can steer its behavior. OWUI's import flow shows a preview of each skill before anything is created, and a skill from openwebui.com opens in the editor for review before you click Save & Create. Read the whole skill, including supporting files, before enabling it. `[EXT]` (Nous) runs a security scan on installed skills for exfiltration, prompt injection and destructive commands, which shows the concern is real elsewhere.

### 3.5 Terminal skills `[OWUI]`

A connected Open Terminal server can carry its own skills, kept next to the files they describe. OWUI offers them in the `$` picker under ids like `terminal:<url-encoded name>` (Open Terminal serves them from v0.13.0).

- They behave like other skills: `$` injects the content, otherwise they sit in the manifest and load through `view_skill`.
- What gets injected carries the skill's directory and the file paths it ships with, so the model knows where to read.
- `read_skill_file` pages through a terminal skill's `SKILL.md`, and `grep_skill_files` searches it. Its other files stay on the terminal, and the model opens them with the terminal's own file tools.
- The skill editor, version history and `update_skill_files` work only on workspace skills.

`[DERIVED]` Because a terminal skill's scripts live on the machine that runs them, terminal skills are the cleanest home for skills that bundle executable scripts (section 8).

---

## 4. The authoring standard

OWUI states that this is the standard it follows when it writes a skill through `/skills:create`, and the one to hold your own skills to. `[OWUI]` Treat it as the default. Whether the section order is enforced is not documented. `[OPEN]`

### 4.1 Frontmatter

| Key | Rule |
| :--- | :--- |
| `name` | `[OWUI]` Lowercase and hyphenated, no spaces, 64 characters at most. `[SPEC]` is stricter: letters, digits and hyphens only; no leading, trailing or consecutive hyphens; must match the folder name. **Follow the stricter form.** The Skill ID is generated from the name and is read-only after creation. `[OWUI]` |
| `description` | `[OWUI]` 1024 characters at most, ends in a period. Name the capability, skip the implementation, do not repeat the skill name, and leave out words like powerful, comprehensive, seamless, advanced or robust. Count the characters before saving. See section 5. |
| `platforms` | `[OWUI]` `[macos]`, `[linux]` or `[windows]`, only when the skill uses something OS-bound. Omit for portable skills. How OWUI enforces it, and which OS it refers to (server, terminal or client), is undocumented. `[OPEN]` |
| Other keys | `[SPEC]` defines optional `license`, `compatibility` (500 characters at most), `metadata` and an experimental `allowed-tools`. OWUI documents none of them. Do not rely on them. If you add them for portability, confirm creation and import still work. `[OPEN]` |

**YAML handling.** OWUI parses frontmatter as real YAML, so quoted and multi-line values come through as written. `[OWUI]` `[DERIVED]` Because it is real YAML, an unquoted value containing a colon followed by a space is invalid. Quote the whole `description`.

**Name and Description fields vs frontmatter.** Frontmatter fills an empty Name and Description while you create a skill, and imports read name and description from the frontmatter. `[OWUI]` What happens when you later edit the frontmatter in the editor is not documented. `[OPEN]` Keep the Description field and the frontmatter identical, and verify both after every edit.

**Choosing a name** `[EXT]` (Anthropic): make it specific and descriptive, use a consistent pattern across your skills, and avoid vague names (`helper`, `utils`, `tools`) and over-generic ones (`documents`, `data`, `files`). A verb-led or gerund form such as `drafting-postmortems` reads well but is optional. Anthropic's reserved-word rule for names applies to Anthropic's platform, not to OWUI.

### 4.2 Body sections, in this order `[OWUI]`

| # | Section | Contents |
| :--- | :--- | :--- |
| 1 | `# <Human Title>` plus a short intro | What it does, what it does not do, what it assumes is installed. |
| 2 | `## When to Use` | Concrete trigger phrases. |
| 3 | `## Prerequisites` | Exact environment variables, credentials and install steps, or `None`. |
| 4 | `## How to Run` | The canonical workflow, framed through the tools the model actually has. |
| 5 | `## Quick Reference` | A flat list of commands, routes, files or APIs. |
| 6 | `## Procedure` | Numbered steps, copy-paste exact. |
| 7 | `## Pitfalls` | Known limits and failure modes. |
| 8 | `## Verification` | One focused check that proves the skill works. |

`[DERIVED]` `## Pitfalls` is the place for the high-value corrections described in section 6.4. `## When to Use` is only read after the skill has loaded, so it reinforces the description rather than replacing it (section 5).

### 4.3 Framing tools `[OWUI]`

- Name tools in backticks. The standard names `run_command`, `write_file` and `view_skill` as examples.
- Describe shell work as run through `run_command`. Third-party CLIs are fine inside a procedure when it is clear the agent invokes them through `run_command`.
- Prefer the read and search tools over raw shell utilities where one exists.
- Open Terminal's built-in tools include `run_command`, `read_file`, `write_file`, `grep_search`, `glob_search` and process management. `[OWUI-WEB]` Tool names depend on what is connected. `[DERIVED]` Reference only tools the target model will actually have, and ask the user when unsure.

### 4.4 Quality bar `[OWUI]`

- Use commands, routes, paths, function names, config keys and error text exactly as in the source.
- Keep `SKILL.md` scannable: roughly 100 lines for a simple workflow, 200 for a complex one. Put the bulk in `scripts/`, `references/`, `templates/` and `assets/`.
- A skill that only points at other skills is worth nothing. Write the one that does the work.

`[SPEC]` and `[EXT]` suggest up to 500 lines or about 5,000 tokens. OWUI's bar is stricter and governs.

`[EXT]` (Nous) Do not restate what the model already receives every turn, such as system prompt text or tool schemas.

---

## 5. Writing the description

### 5.1 Why it carries so much weight

- The manifest shows only a skill's name and description. For lazy-loaded skills the model uses the description to decide whether to call `view_skill`. `[OWUI]`
- The description carries the entire burden of triggering. If it does not say when the skill is useful, the model will not reach for it. `[SPEC]`
- Models usually consult a skill only for tasks that need knowledge or capabilities beyond what they handle alone, so a simple one-step request may not trigger a skill even when the description matches. `[SPEC]` Whether OWUI models behave this way is untested. `[OPEN]` It is another reason to build skills for specialized knowledge, not for tasks the model already does well.
- `## When to Use` in the body is read only after the skill has loaded. `[DERIVED]` Trigger contexts therefore belong in the description. The body section reinforces them and can cover finer boundaries.

### 5.2 Formula `[DERIVED]`

```
<Capability, verb-first, 1-2 sentences>. Use when <trigger contexts, including indirect phrasings>. Do not use for <nearest look-alike task>.
```

The third sentence is optional. Add it when a neighboring task would plausibly trigger the skill by mistake.

### 5.3 Rules

1. **State what and when.** Both the capability and the contexts that call for it. `[SPEC]` `[EXT]`
2. **Describe the user's intent, not the internals.** The model matches against what the user asked for, not how the skill works. `[SPEC]` OWUI says the same: skip the implementation. `[OWUI]`
3. **List contexts exhaustively, including indirect ones.** Cover cases where the user never names the domain, for example someone who pastes raw data and asks for "a chart for my boss". `[SPEC]`
4. **Front-load the key use case and trigger words.** Some hosts shorten descriptions, and the front survives. `[EXT]` (OpenAI)
5. **State the boundary** when look-alike tasks exist: what the skill does not cover. `[EXT]` (OpenAI) `[SPEC]`
6. **Be thorough about contexts, not about praise.** `[SPEC]` encourages listing applicable contexts generously. OWUI bans filler adjectives such as powerful, comprehensive, seamless, advanced and robust. `[OWUI]` Both hold: more trigger contexts, no marketing words.
7. **Avoid first and second person.** Write "Extracts text from PDFs", not "I can extract text" or "You can use this to extract text". `[EXT]` (Anthropic) For the imperative-versus-third-person conflict, see section 14.2.
8. **Do not repeat the skill name,** and end with a period. `[OWUI]`
9. **Keep it to a few sentences.** `[SPEC]` Hard limit 1024 characters. `[OWUI]`
10. **Measure length; do not eyeball it.** `[DERIVED]` LLMs count characters unreliably. Aim for 600 characters or fewer, or count with code if a code tool is available. Descriptions tend to grow during optimization.
11. **Generalize when fixing misses.** Do not paste keywords from a failed test prompt into the description. Find the category the prompt belongs to and describe that. `[SPEC]`

### 5.4 Examples (illustrative)

Weak:
```yaml
description: Helps with incidents.
```
Why it fails: no capability, no triggers, no boundary, no period, and "helps" matches nothing in particular.

Better:
```yaml
description: "Turns incident timelines and chat logs into a blameless postmortem covering impact, root cause and follow-up actions. Use when the user shares an outage timeline or asks for an incident write-up, retrospective or RCA, even if they never say 'postmortem'. Do not use for status-page updates during a live incident."
```
Why it works: verb-first capability, direct triggers and indirect ones, an explicit boundary, under 1024 characters, ends in a period, and the whole value is quoted so YAML parses it.

Weak (marketing):
```yaml
description: A powerful, comprehensive skill for all your data needs.
```
Fix: name the data task, the file types and the situations. Remove the adjectives.

---

## 6. Writing the body

### 6.1 Add what the model lacks

Include project conventions, domain procedures, non-obvious edge cases, and the specific tools or APIs to use. Leave out what the model already knows, such as what a PDF is or how HTTP works. For each paragraph ask: would the model get this wrong without it? If not, cut it. If unsure, test. `[SPEC]` `[EXT]`

Do not generate a skill from the model's general knowledge alone. That yields vague guidance like "handle errors appropriately". Ground it in real material: a worked conversation including your corrections, runbooks, style guides, API specs, past failures and their fixes. `[SPEC]`

### 6.2 Scope `[SPEC]`

A skill should be a coherent unit of work, like a well-scoped function. Too narrow, and several skills must load for one task, adding overhead and conflicting instructions. Too broad, and the skill is hard to trigger precisely. A skill that queries a database and formats the results is probably one unit. One that also administers the database is probably doing too much. OWUI's own standard adds that a skill whose only content is pointers to other skills is worthless. `[OWUI]`

### 6.3 Calibrate firmness `[SPEC]` `[EXT]`

- **Fragile or order-dependent operations** (migrations, destructive actions): be prescriptive. Give the exact command and say not to modify it.
- **Flexible work** (reviews, analysis): describe what to look for and why. Reasoning-based instructions generalize better than rigid ALWAYS and NEVER rules.
- **Provide defaults, not menus.** Pick one tool or approach, and mention an alternative only with its condition: "For scanned PDFs, use X instead."
- **Favor procedures over declarations.** Teach an approach that works for a class of tasks, not the answer to one instance. Specific details such as output templates and constraints are still welcome.

### 6.4 Pitfalls are the highest-value content

- The most valuable lines in many skills are concrete, environment-specific facts that defy reasonable assumptions. `[SPEC]` Example (illustrative): "The export uses semicolons, not commas. Set the delimiter explicitly, otherwise every row collapses into one column." Generic advice such as "be careful with data" earns nothing.
- Keep them in `SKILL.md` (`## Pitfalls`), where the model reads them before it meets the situation. A separate file works only if the model recognizes the trigger. `[SPEC]`
- Write each as a rule plus one clause of why, attached to the step it affects, stated once. Leave out incident stories, ticket numbers, dates and quoted chat. The rule must stand without the story. `[EXT]` (Nous)
- Whenever you correct the model during testing, add the correction here. `[SPEC]`

### 6.5 Instruction patterns

- **Numbered steps** with the expected output stated. OWUI's workflow guidance says numbered instructions are followed more reliably than prose, and that the output you expect should be spelled out. `[OWUI-WEB]`
- **Templates for output format.** Concrete structures are matched more reliably than prose descriptions. Keep short ones inline and put long or rarely needed ones in `templates/` or `assets/`. State whether the template is strict or a flexible default. `[SPEC]` `[EXT]`
- **Examples.** Input and output pairs convey style and level of detail better than description. `[EXT]`
- **Checklists** for multi-step workflows with dependencies or validation gates. `[SPEC]` `[EXT]`
- **Validation loops.** Do the work, run a check (a script, a reference checklist or a self-check), fix, repeat until it passes. This is what OWUI's `## Verification` section is for. `[SPEC]` `[EXT]` `[DERIVED]`
- **Plan, validate, execute** for batch or destructive work. Have the model write a structured plan, check it against a source of truth, then act. Make the validator's errors specific enough to self-correct from. `[SPEC]` `[EXT]`

### 6.6 Style `[EXT]`

- Use one term per concept and keep it throughout (always "field", never a mix of "field", "box" and "element").
- Avoid time-sensitive statements such as "before August, use the old API". Keep the current method in the main text and move legacy notes to a clearly marked section.
- Write steps as imperatives with explicit inputs and outputs. (OpenAI)
- Use forward slashes in all paths.
- Do not duplicate content already delivered every turn (section 4.4).

### 6.7 Write for the weakest model you will use `[EXT]`

Skills add to a model, so results depend on the model. Weaker or smaller models need more explicit guidance, and stronger ones suffer from over-explaining. In OWUI, where the user may run several local models, test on each and tune to the weakest one you must support. Open Terminal itself needs a model with agentic-quality tool calling, not just nominal support. `[OWUI-WEB]`

---

## 7. Supporting files and progressive disclosure

### 7.1 Layout `[OWUI]` `[SPEC]`

```
skill-name/
  SKILL.md        required; UTF-8 text
  scripts/        optional; executable code
  references/     optional; documentation read on demand
  templates/      optional; output templates
  assets/         optional; static resources
```

Name files descriptively (`form_validation_rules.md`, not `doc2.md`) and organize by domain or feature (`reference/finance.md`, not `docs/file1.md`). `[EXT]`

### 7.2 The three levels, mapped to OWUI `[SPEC]` `[DERIVED]`

| Level | General idea `[SPEC]` | In OWUI |
| :--- | :--- | :--- |
| 1. Metadata | Name and description always loaded (roughly 100 tokens per skill). | The manifest, or nothing if skill tools are unavailable. |
| 2. Instructions | The full `SKILL.md` loads on activation. The spec recommends under 5,000 tokens and 500 lines. | `view_skill` returns `SKILL.md` plus the file list, up to 20,000 characters. OWUI's own bar is about 100 to 200 lines. |
| 3. Resources | Other files load only when needed. | `read_skill_file` and `grep_skill_files` for workspace skills. The terminal's file tools for terminal skills. |

The token figures are `[SPEC]` guidance. OWUI's limits are in characters, so do not mix the units.

### 7.3 Rules

1. **Link every supporting file directly from `SKILL.md`, one level deep.** Models may only partly read files reached through chains of references. `[SPEC]` `[EXT]`
2. **Say when to load each file.** "Read `references/api-errors.md` if the API returns a non-200 status" works. "See references/ for details" does not. `[SPEC]`
3. **Keep each reference file focused,** because smaller files cost less context when loaded. `[SPEC]`
4. **Keep each reference file under 20,000 characters, or add a contents list at the top** so the model can jump or search. `[EXT]` suggests a contents list for reference files over about 100 lines. In OWUI, a read returns at most 20,000 characters from an offset, and `grep_skill_files` returns up to 10 excerpts. `[OWUI]` `[DERIVED]`
5. **Put the most common workflow first** and edge cases last. `[EXT]` (Nous)
6. **Keep gotchas in `SKILL.md`.** `[SPEC]`
7. **Do not depend on the model reading binary files.** `read_skill_file` returns only a size and an editor link for them. Reference material the model must read has to be text. `[OWUI]` `[DERIVED]`
8. **Respect the file limits:** 10 MiB per file, 50 MiB per skill, 1,000 files. `[OWUI]`
9. **If skill tools may be missing, nothing outside `SKILL.md` is reachable.** Fold the essentials into `SKILL.md`. `[DERIVED]`

### 7.4 Common structures `[EXT]`

- **Guide plus references:** `SKILL.md` has quick start and links to `FORMS.md`, `REFERENCE.md` and similar files, each with a load condition.
- **Domain split:** one reference file per domain (finance, sales, product), so a question about sales loads only sales material.
- **Conditional details:** basic content inline, advanced content in separate files loaded only for those cases.

---

## 8. Scripts and execution

### 8.1 What is documented

- OWUI stores a skill's scripts for the model to read. Running one needs a tool that executes code: Open Terminal or the code interpreter. `[OWUI]`
- Open Terminal's built-in tools include `run_command`, `read_file` and `write_file`. `[OWUI-WEB]`
- The standard says to frame shell work through `run_command`. `[OWUI]`
- Terminal skills keep their files on the terminal and inject the skill directory and file paths. `[OWUI]`

### 8.2 Instructions first

Prefer an instruction-only skill unless you need deterministic behavior or external tooling. `[EXT]` (OpenAI) Reach for a script when:

- a command is complex enough that it is hard to get right on the first try, or
- you see the model reinventing the same logic every run (building charts, parsing a format, validating output). `[SPEC]`

Where an existing package already does the job, one-off commands with pinned versions need no `scripts/` folder. State prerequisites, such as the runtime version, in `## Prerequisites`. `[SPEC]` `[EXT]` (Nous) also prefers the standard library, `curl` and tools the agent already has, and documenting install steps when a dependency is unavoidable.

### 8.3 The unresolved question: getting a workspace skill's script to a place it can run `[OPEN]`

The sources do not say how a script stored in a workspace skill reaches the filesystem where `run_command` executes, nor whether relative paths in the skill text resolve there. Do not assume either. Options, none of them verified:

1. **Terminal skill.** Host the skill on the Open Terminal server so scripts already sit where they run. `[DERIVED]`
2. **No bundled file.** Use one-off commands that need no stored script. `[DERIVED]`
3. **Copy then run.** For a workspace skill, instruct the model to read the script with `read_skill_file`, write it out with `write_file`, then run it with `run_command`. `[DERIVED]` This costs the script's tokens and may fail on weaker models.

**To resolve it,** build a minimal skill with a three-line script, ask a model to run it, and watch which tools it calls. Report the result to the user, and tell them it is worth recording in this guide.

### 8.4 Token economics warning `[DERIVED]`

Anthropic's guidance says bundled scripts save tokens because only their output enters context. `[EXT]` That relies on a filesystem the agent runs the script from directly. For a workspace skill in OWUI, the model may have to read the script first (8.3, option 3), so the savings may not apply. Do not promise them to the user.

### 8.5 Designing scripts for agent use `[SPEC]`

When a model runs your script it reads stdout and stderr to decide what to do next.

- **Non-interactive.** No prompts, password dialogs or confirmation menus. Accept input through flags, environment variables or stdin. A script that waits for input hangs.
- **Document `--help`.** It is how the model learns the interface. Include a brief description, flags and examples, and keep it short.
- **Helpful errors.** Say what went wrong, what was expected and what to try.
- **Structured output.** Prefer JSON, CSV or TSV. Send data to stdout and progress or diagnostics to stderr.
- **Idempotent.** Models retry. "Create if not exists" beats "create and fail on duplicate".
- **Closed input sets.** Reject ambiguous input with a clear error instead of guessing.
- **Dry-run and safe defaults** for destructive or stateful operations, and require explicit confirmation flags where risk warrants.
- **Meaningful exit codes,** documented in `--help`.
- **Bounded output.** Many harnesses truncate long tool output (the guidance cites roughly 10,000 to 30,000 characters). Default to a summary or limit and support `--offset` or an `--output` flag. How OWUI or Open Terminal truncates output is undocumented. `[OPEN]`

For fragile scripts, also use the plan-validate-execute pattern from section 6.5. `[SPEC]` `[EXT]` Also say whether the model should **run** a script or **read** it for reference. `[EXT]` (Anthropic)

### 8.6 Verify before shipping `[DERIVED]`

OWUI's standard requires exact commands. Before presenting a skill, run every command, flag and path it contains at least once, or mark the unverified ones for the user.

---

## 9. Design intake (run this before drafting)

Ask only what you cannot infer from the conversation, and batch the questions. If the user has already given an answer, use it.

1. **Outcome.** What should the model do differently with this skill than without it? If the answer is "nothing", do not build a skill.
2. **Source material.** What real expertise grounds it: a conversation where the task went well (including the corrections you made), runbooks, a style guide, API docs, past failures and their fixes? If none exists, ask for some. Do not synthesize from general knowledge. `[SPEC]`
3. **Triggers.** What will the user say when the skill should apply, including indirect phrasings? Which similar tasks should not trigger it?
4. **Execution.** Instruction-only (the default) or scripts? Is Open Terminal connected? Which tools does the target model have, by exact name? `[EXT]` (OpenAI) asks the same three things: what the skill does, when it triggers, and whether it stays instruction-only.
5. **Models.** Which models will use it? Which has the smallest context window and the weakest tool calling? Do they have native function calling, the Builtin Tools capability and the Skills category on?
6. **Activation.** Always on for a model, discoverable on demand, or explicit with `$`?
7. **Audience and access.** Private, shared with users or groups, or public? Do those users have read access?
8. **Constraints.** Exact commands, paths and environment variables to use. Anything the model must never do. Required output format.

### 9.1 Choosing the activation `[DERIVED]`

| Situation | Choose |
| :--- | :--- |
| Rules the model should follow in every chat (house style, standing safety rules) | Bind to the model. Keep the body short, because it costs context on every message. |
| A procedure needed only for some tasks | Leave unbound and rely on the description. Needs skill tools. |
| The user wants control over when it applies | `$` mention or per-chat toggle. |
| The target model lacks native function calling or Builtin Tools | Bind or `$`-mention, and make `SKILL.md` self-contained with no required references. |
| Many skills on one model | Keep descriptions short. Consider the admin lazy-loading setting so bound skills load on demand. |
| The skill needs executable scripts | Terminal skill, or one-off commands with no bundled file (section 8.3). |

### 9.2 Using `/skills:create` `[OWUI]`

Typing `/skills:create` in a chat, optionally followed by guidance, has the model gather material from the chat and save a new private skill with `create_skill`, authored to the standard in section 4. It needs the Skills Access permission, a model with built-in tools on and its Skills category switched on, and a chat that already has content.

`[DERIVED]` Treat the result as a draft. Review it against section 12, then revise it in the editor or with `update_skill_files`.

`[EXT]` (Anthropic) A workflow that works well: complete a real task with one assistant first, then extract the reusable pattern; review the draft for concision; reorganize detail into reference files; test it in a fresh chat; bring the observations back and iterate.

---

## 10. Testing and iteration

Test with real requests, not assumed ones. A skill that looked right on one prompt may fail across varied ones, may not trigger, or may add nothing over the bare model. `[SPEC]`

### 10.1 Structural checks (after every save) `[DERIVED]`

- Name, Description and Skill ID are populated and match the frontmatter.
- The description is 1024 characters or fewer (measure it) and ends in a period.
- `SKILL.md` is about 100 lines (simple) or 200 (complex), and well under 20,000 characters.
- Every command, flag, path and tool name matches the source, and each was run at least once.
- Access works for a user who should have the skill and fails cleanly for one who should not.

### 10.2 Trigger tests (needs skill tools) `[SPEC]` adapted `[DERIVED]`

1. Write about 20 realistic prompts: 8 to 10 that should trigger the skill and 8 to 10 that should not. For a small personal skill, 10 is a sensible minimum.
2. Vary the positives in phrasing (formal, casual, typos), explicitness (names the domain or only describes the need), detail (terse or context-heavy) and complexity (a single step or buried in a longer task). The most useful positives are those where the skill helps but the link is not obvious.
3. Make the negatives **near-misses** that share keywords or concepts but need something else. Obvious non-matches test nothing.
4. Make the prompts realistic: file paths, personal context, specific details, occasional typos.
5. Run each in a **fresh chat** and look for a `view_skill` call for this skill. How tool calls are displayed depends on your build.
6. Run each prompt about 3 times, because behavior is nondeterministic. A positive passes if it triggers in more than half its runs. A negative passes if it triggers in fewer than half.
7. If you iterate a lot, hold out about 40% of the prompts as a validation set that you never use to guide edits, so you do not overfit to the prompts you tuned on.
8. When positives fail, the description is probably too narrow: broaden the scope or contexts. When negatives fire, it is too broad: add a boundary. Fix categories, not individual keywords. If several rounds do not help, try a structurally different description. Usually about five iterations is enough. Re-measure the length after each edit.

### 10.3 Output tests `[SPEC]`

1. Start with 2 or 3 realistic prompts, including one edge case: malformed input, an unusual request, or an ambiguous instruction.
2. Run each in a clean chat **with** the skill and **without** it, or against the previous version when improving a skill.
3. Add assertions after you see the first outputs. Good ones are specific and checkable: valid JSON, exactly three items, both axes labeled. Weak ones are vague ("output is good") or brittle (an exact phrase).
4. Require concrete evidence for a pass. A heading with one vague sentence under it is a fail.
5. Drop assertions that pass in both configurations. They measure the base model. Investigate ones that fail in both. Focus on those that pass only with the skill, and work out which instruction made the difference.
6. Review outputs by hand as well. Some quality is hard to express as a check. Record concrete feedback ("the chart has no axis labels"), not impressions ("looks off").
7. Compare cost too: extra tokens and time against the quality gain. A large cost for a small gain is a bad trade.
8. Read the tool calls and transcript. If the model skipped an instruction, it may be ambiguous. If it wasted steps, the instructions may be vague, inapplicable, or offer too many options without a default.

### 10.4 Model matrix `[EXT]` `[OWUI-WEB]`

Run the tests on every model the skill is meant for, starting with the weakest. If it works on the weakest, it usually works on stronger ones, though they may then be over-instructed. Remember that Open Terminal needs genuine agentic tool calling.

### 10.5 The iteration loop `[SPEC]`

1. Give the failed assertions, your feedback and the current `SKILL.md` to an LLM and ask for proposed changes.
2. Tell it to **generalize** (fix the underlying problem, not the single example), to **keep the skill lean** (remove instructions that cause wasted work; if pass rates stall as rules pile up, try removing some), to **explain why** rather than issue rigid commands, and to **bundle repeated work** (if every run writes the same helper, it belongs in `scripts/`, subject to section 8.3).
3. Review and apply the changes, rerun everything, and compare. Stop when results are good and feedback is consistently empty, or when you stop seeing improvement.

### 10.6 OWUI workflow notes `[OWUI]`

- Add a short note in **Describe this change** beside **Save**. It labels the version that save creates. `[DERIVED]` Record what failed and what you changed, so the version list reads as an iteration log.
- **Compare to current** shows name, description and translation differences plus a line-by-line diff of changed files. **Set as Production** makes an earlier version current again; it needs write access.
- In the file editor, **Compare revision** lets you paste a proposed revision of a file next to the original and apply changes selectively.
- Each edit that changes name, description, translations or files creates a version, so experiments are reversible.

---

## 11. Template and worked example

### 11.1 Template

Replace every `<...>`. Do not leave placeholders in a shipped skill.

````markdown
---
name: <lowercase-hyphenated-name>
description: "<Capability, verb-first, 1-2 sentences>. Use when <trigger contexts, including indirect phrasings>. Do not use for <nearest look-alike task>."
---

# <Human Title>

<Two or three sentences: what the skill does, what it does not do, and what it assumes is already installed.>

## When to Use

- <Concrete trigger phrase or situation>
- <Another, including an indirect phrasing>
- Not for: <look-alike task, and where to go instead>

## Prerequisites

<Exact environment variables, credentials and installs (with pinned versions), or `None`. Never put secret values here.>

## How to Run

<One short paragraph: the canonical workflow and the tools it uses, each named in backticks, for example `run_command`. Name only tools the target model has.>

## Quick Reference

- <exact command, route, file or API, copied from the source>
- <...>

## Procedure

1. <Imperative step with its input and expected output.>
2. <Step. If a supporting file matters here: "Read `references/<file>.md` if <condition>.">
3. <Validation step: run the check, fix, repeat until it passes.>

## Pitfalls

- <Concrete rule>, because <one-clause reason>.
- <...>

## Verification

<One focused check that proves it worked, and what a pass looks like.>
````

### 11.2 Worked example (illustrative, not a real skill; contains no commands)

````markdown
---
name: drafting-postmortems
description: "Turns incident timelines and chat logs into a blameless postmortem covering impact, root cause and follow-up actions. Use when the user shares an outage timeline or asks for an incident write-up, retrospective or RCA, even if they never say 'postmortem'. Do not use for status-page updates during a live incident."
---

# Drafting Postmortems

Produces a blameless written postmortem from raw incident material. It does not investigate the incident itself or invent facts missing from the material.

## When to Use

- The user pastes a timeline, chat export or alert history and wants a write-up.
- The user asks for an RCA, retrospective or incident summary.
- Not for live-incident status updates. Draft those directly.

## Prerequisites

None.

## How to Run

Work from the material the user supplied. Read the layout in `templates/postmortem.md` with `read_skill_file` before drafting.

## Quick Reference

- Sections, in order: Summary, Impact, Timeline, Root cause, What went well, What went poorly, Follow-up actions.

## Procedure

1. List every event in the material with its timestamp and source. Do not add events that are not in it.
2. Fill the template. Write impact in measurable terms (who, how long, how many) and say "unknown" where the material does not say.
3. Phrase causes in terms of systems and processes, never individuals.
4. Give each follow-up action an owner field and a due-date field, left blank if the user has not supplied them.
5. Re-read the draft against the event list. Fix any claim that has no source event.

## Pitfalls

- Timestamps from different tools may be in different time zones. Normalize before ordering events, because a wrong order produces a wrong root cause.
- Do not name an individual as the cause, because a blameless review is the point of the document.

## Verification

Every statement in Impact and Root cause traces to a line in the supplied material.
````

---

## 12. Final checklist (run before presenting a skill)

**Frontmatter**
- [ ] `name` is lowercase and hyphenated, 64 characters or fewer, with no leading, trailing or doubled hyphens, and is specific.
- [ ] `description` is 1024 characters or fewer (measured), ends in a period, and contains capability, trigger contexts and, if needed, a boundary.
- [ ] No filler adjectives, no first or second person, and the skill name is not repeated.
- [ ] YAML values containing a colon and a space are quoted.
- [ ] `platforms` is set only if the skill is OS-bound. No undocumented keys are relied on.

**Body**
- [ ] The eight sections appear in the standard order, in about 100 lines (simple) or 200 (complex).
- [ ] Every command, flag, path, function name and error text is exact and was verified, or is marked for the user.
- [ ] Only tools the target model has are named, in backticks.
- [ ] Each paragraph earns its place ("would the model get this wrong without it?").
- [ ] Defaults are given instead of menus. Firmness matches fragility.
- [ ] Steps are numbered and the expected output is stated.
- [ ] Pitfalls are concrete, each with a one-clause reason, and contain no incident stories or dates.
- [ ] One term per concept. No time-sensitive statements.
- [ ] No secrets. Required environment variables are named in Prerequisites.

**Files**
- [ ] Each supporting file is linked from `SKILL.md` with a when-to-load condition, one level deep.
- [ ] No reference file exceeds 20,000 characters without a contents list.
- [ ] Nothing the model must read is binary.
- [ ] If the target model may lack skill tools, `SKILL.md` stands alone.
- [ ] Scripts, if any, are non-interactive, have `--help`, give clear errors, print structured output and bound it. The delivery path (section 8.3) is settled or flagged to the user.

**Testing and reporting**
- [ ] Trigger and output tests were run, or the user was told they were not.
- [ ] Tested on the weakest intended model, or the user was told it was not.
- [ ] Every `[OPEN]` item that affects this skill is reported to the user, not guessed.

---

## 13. Reference numbers `[OWUI]` unless marked

| Item | Value |
| :--- | :--- |
| `name` length | 64 characters at most |
| `description` length | 1024 characters at most, ends in a period |
| `SKILL.md` size guidance | About 100 lines (simple), 200 (complex) |
| Skill Read Limit (default) | 20,000 characters per read (`SKILL_CONTENT_MAX_CHARS`) |
| Skill Preview Length (default) | 5,000 characters (`SKILL_PREVIEW_MAX_CHARS`) |
| Injection cap without skill tools | First 100,000 characters of `SKILL.md` |
| `grep_skill_files` results | Up to 10 excerpts |
| File limits | 10 MiB per file, 50 MiB per skill, 1,000 files per skill |
| Import size | Up to 200 MiB per import |
| Share to openwebui.com | Up to 15 MiB per skill |
| Version list page size | 20 per page |
| Open Terminal skill endpoints | Served from v0.13.0 |
| Metadata cost per skill `[SPEC]` | About 100 tokens |
| Body size `[SPEC]` | Under 500 lines and about 5,000 tokens (OWUI is stricter) |

---

## 14. Open questions, conflicts and non-features

### 14.1 Open questions (`[OPEN]`): verify in a live instance and report to the user

| # | Question | Why it matters | How to check |
| :--- | :--- | :--- | :--- |
| O1 | How does a script stored in a workspace skill reach the filesystem where `run_command` runs? Do relative paths resolve? | Decides whether bundled scripts work at all, and their token cost. | Build a skill with a three-line script. Ask a model to run it. Observe the tool calls. |
| O2 | After creation, does editing `description` in the frontmatter update the Description field the manifest shows? | A stale field means the model routes on the wrong text. | Edit the frontmatter, save, then check the field and the manifest behavior. |
| O3 | What happens with extra frontmatter keys (`license`, `compatibility`, `metadata`, `allowed-tools`) and with invalid YAML? Does `platforms` hide or block anything, and relative to which OS? | Affects portability and silent failures. | Create test skills and observe the editor and manifest. |
| O4 | Is the eight-section order enforced, or only recommended? | Determines how strictly to hold hand-written skills to it. | Save a skill with a different order and see whether anything objects. |
| O5 | Do model-bound skills appear in the manifest when admin lazy loading is off? | The main docs and the Advanced Workflows page word this differently (see 14.2). | Check a model with a bound skill and the setting off and on. |
| O6 | Which OWUI docs version matches your instance? | A differently worded Skills page was seen on docs.openwebui.com with older-looking import and export behavior. This guide is based on the supplied doc, whose version is not recorded. | Check your instance's version against the docs. |
| O7 | Do simple one-step tasks trigger OWUI skills less reliably, as other platforms report? | Sets expectations for descriptions on basic tasks. | Trigger tests (section 10.2) with simple positives. |
| O8 | How are long tool outputs from `run_command` truncated, if at all? | Determines how strictly scripts must bound their output. | Run a command that prints a lot and inspect what the model receives. |
| O9 | OWUI can translate a skill's Name and Description per language. Does the model see the translated text or the original in the manifest? | A translated description could change how reliably the model routes to the skill. | Set a translation, start a chat in that language and inspect the manifest behavior. |

### 14.2 Conflicts between sources and how this guide resolves them

1. **Description voice.** Anthropic says write in third person and avoid "I" and "you". `[EXT]` The Agent Skills guide says use imperative phrasing, such as "Use this skill when...". `[SPEC]` **Resolution `[DERIVED]`:** both sources' own examples lead with a verb-first capability statement and then a "Use when..." trigger clause. Do that: no "I" or "you", capability first, then the trigger clause.
2. **Where trigger phrases live.** OWUI's standard puts trigger phrases in the body's `## When to Use`. The manifest, which drives lazy loading, shows only name and description. **Resolution `[DERIVED]`:** put the triggers in the description and repeat or refine them in `## When to Use`.
3. **Size.** OWUI says about 100 to 200 lines. `[SPEC]` and `[EXT]` say up to 500 lines. **Resolution:** OWUI's bar governs.
4. **Manifest membership.** The main OWUI doc says model-bound and selected skills join the manifest only when the admin lazy-loading setting is on. The Advanced Workflows page says the manifest lists every active skill the user can read, attached ones included. `[OWUI-WEB]` **Resolution:** follow the main doc, and check O5.
5. **Script token savings.** Anthropic says scripts save tokens because code never enters context. `[EXT]` For OWUI workspace skills this is unverified (O1, section 8.4). **Resolution:** do not promise it.
6. **Description length.** OWUI's limit is 1024 characters. Hermes uses a much shorter house standard. `[EXT]` **Resolution:** OWUI's limit governs. Shorter is still cheaper in the manifest.

### 14.3 Documented elsewhere but not documented in OWUI: do not assume these exist

Absence from the OWUI docs is not proof of absence in the software. These are simply not something to rely on.

- Path template tokens (such as an automatic skill-directory variable) and inline shell snippets in `SKILL.md` (Hermes).
- Conditional activation by available tools or toolsets, required environment-variable declarations, config settings declared in frontmatter, and blueprint or schedule blocks (Hermes).
- An invocation-policy file that disables implicit triggering, or an `agents/` metadata folder (OpenAI).
- Skill discovery by scanning `.agents/skills` directories (OpenAI).
- A manifest token budget that shortens or omits descriptions (OpenAI Codex).
- The `allowed-tools`, `compatibility`, `license` and `metadata` frontmatter keys (Agent Skills spec). See O3.
- Reserved-word restrictions on names (Anthropic's platform).

---

## Appendix A. Provenance and verification notes

*For human reviewers. An LLM using this guide can skip this appendix.*

**Baseline.** The OWUI Skills documentation supplied by the user (identical to the copy in the knowledge-base repository). Everything tagged `[OWUI]` was checked against it. A few specific claims (version-note field, community skills, the 15 MiB share limit, binding advice, inactive-skill behavior, the Open Terminal v0.13.0 note) were re-verified directly in the text.

**Knowledge base** (`Kyra-Lee/Agent-Skills-Knowledge-Base`, read 2026-11-08): the Agent Skills specification, best-practices, optimizing-descriptions, evaluating-skills and using-scripts guides; Anthropic's skill authoring best practices; OpenAI's build-skills page; and the Nous Research Hermes skills docs. Used for `[SPEC]` and `[EXT]` guidance, paraphrased, not copied.

**Web pages fetched** (docs.openwebui.com, 2026-11-08): the Tools page (Open Terminal built-in tool names), the Open Terminal page (agentic-quality tool-calling note), and Advanced Workflows with Skills (authoring tips and the manifest wording that conflicts with the main doc). These were not available to the local models.

**Local-model council.** Four primers were evaluated claim by claim: Gemma 4 26B-A4B, Qwen 3.5 120B-A10B, Qwen 3.8 27B and Nemotron 3.5 120B-A12B. The Qwen 3.8 27B primer supplied most of the skeleton and the augmentation that survived. Rejected from the primers:

- statements that OWUI "does not implement" features, which were inferences from silence presented as sourced fact;
- guidance aimed at people building skill-capable clients (catalog filtering, activation strategies, compaction, per-file allowlisting, trust gating, discovery paths);
- cross-platform items irrelevant to authoring (Codex budget numbers, invocation-policy files, document-task skills, vendor APIs);
- three errors: a wrong mapping of the preview to progressive-disclosure tiers, a claim that `## When to Use` helps the catalog decision, and a claim that OWUI has no skill catalog (it has Community skills);
- process and self-summary sections.

**Observation, not confirmed.** OWUI's authoring standard closely resembles the Hermes (Nous Research) skill format: the same body section names, a `platforms` field, tool-framing, and a no-invented-commands rule. This suggests the standard was adapted from it, which makes the Hermes docs a plausible but unverified analog where OWUI is silent.

**Limits of this guide.** `[SPEC]` and `[EXT]` content comes from other platforms' documentation snapshots and applies to OWUI by analogy. Section 14.1 lists what has not been tested. When a live test settles one of these questions, update the tag from `[OPEN]` or `[DERIVED]` to the verified source.
