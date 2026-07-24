# Bob Concepts and Best Practices for IBM i Modernization

## Essential Concepts

| Concept | Description | Location/Usage |
|---------|-------------|----------------|
| **Modes** | Specialized AI personas with distinct tool access and behavior for different tasks (Agent, Plan, Ask, etc.) | Switch via mode selector dropdown or `/mode <slug>` command |
| **Skills** | Reusable instruction sets (SKILL.md files) that teach Bob specialized workflows for consistent, repeatable execution | `.bob/skills/<skill-name>/SKILL.md` or `~/.bob/skills/` globally |
| **AGENTS.md** | Persistent project context files that Bob reads at the start of every conversation to understand structure and conventions | Root directory + `.bob/rules-*/` subdirectories; created by `/init` |
| **Rules files** | Plain-text instruction files automatically injected into every conversation to enforce standards | `.bob/rules/*.md` (project) or `~/.bob/rules/*.md` (global) |
| **`.bobignore`** | Exclude files/directories from Bob's tool access (read, edit, execute) using glob patterns | Root directory (like `.gitignore`) — changes are applied automatically |
| **`.bob/rules-<mode-slug>/`** | Mode-specific instruction files loaded alphabetically and combined with mode `customInstructions` | Created by `/init`; add your own `.md` or `.txt` files |
| **MCP Servers** | External tool integrations via Model Context Protocol — extend Bob with databases, APIs, custom scripts | `.bob/mcp.json` (project) or `~/.bob/mcp.json` (global) |
| **`/init` Command** | Scans the codebase and generates persistent `AGENTS.md` context files for all built-in modes | Run in Agentic Chat (Code mode recommended) at project start |
| **Slash Commands** | Built-in and custom automation commands triggered by `/` — map to markdown files in `.bob/commands/` | Built-in: `/init`, `/review`, `/create-pr`; custom: add `.md` to `.bob/commands/` |
| **Context Mentions** | `@`-prefixed references that inject files, folders, terminal output, or git state into the conversation | Type `@` in chat input; supports `@/path/file`, `@terminal`, `@git-changes`, `@url` |

---

## `/init` Command

### Purpose

The `/init` command solves a fundamental LLM limitation: **large language models are stateless** — each new conversation starts with zero memory of previous interactions. Without explicit project context, Bob must re-discover the codebase on every interaction. `/init` fixes this by generating structured `AGENTS.md` files that Bob reads automatically at the start of every conversation.

### When to Run

- **At project start** — before first interactions with Bob
- **After major changes** — structural refactoring, new conventions, new services
- **After adopting new technologies or frameworks**
- **Team onboarding** — help new developers and their Bob instance understand the project
- **Manually supplement** — edit `AGENTS.md` directly to add business rules, deployment conventions, or team-specific context that automated scanning cannot capture

### What It Does

1. Scans project files and configuration
2. Extracts build/test/lint commands
3. Documents code style, naming conventions, and architectural patterns
4. Creates a root `AGENTS.md` with project overview, directory structure, tech stack, and workflows
5. Creates mode-specific context files under `.bob/` for each built-in mode

### Output Files

```
project-root/
├── AGENTS.md                              # General project guidance (all modes)
└── .bob/
    ├── rules-code/AGENTS-code.md          # File structure, naming patterns (Code mode)
    ├── rules-plan/AGENTS-plan.md          # Architectural conventions (Plan mode)
    ├── rules-ask/AGENTS-ask.md            # Documentation context (Ask mode)
    └── rules-advanced/AGENTS-advanced.md  # Deep-dive context (Advanced mode)
```

Each mode-specific file **extends** the root `AGENTS.md` with information relevant to that mode's purpose. For example, Plan mode context emphasizes architecture, while Code mode context highlights file naming patterns.

---

## Rules System

### Rule Scopes and Priority

Bob loads rules from multiple sources, applied in this priority order (later wins over earlier):

1. **Global rules** — `~/.bob/rules/*.md` (personal standards across all projects)
2. **Workspace rules** — `.bob/rules/*.md` (project-specific; version-controlled with the repo)
3. **Mode-specific rules** — `.bob/rules-<mode-slug>/*.md` (overrides for a specific mode)

**Workspace rules override global rules.** Mode-specific rules layer on top of workspace rules for that mode only.

### Rules vs. AGENTS.md

| | Rules files (`.bob/rules/`) | AGENTS.md |
|-|-|-|
| **Purpose** | Enforce coding standards, communication style, workflow constraints | Describe project structure, conventions, commands |
| **Scope** | Injected into every conversation (all modes unless mode-specific) | Applied as persistent context, re-read at conversation start |
| **Format** | Any plain text — Markdown, `.txt` | Markdown, structured with headers |
| **Generated by** | You (manually) | `/init` (then manually maintained) |

### Rule Priority and Configuration Structure

```
.bob/
├── rules/                     # General rules — all modes
│   ├── coding-style.md
│   └── communication.md
├── rules-agent/               # Agent mode only
│   └── typescript.md
└── rules-code/
    └── AGENTS-code.md         # Generated by /init
```

Within a directory, files are loaded **alphabetically** — use numeric prefixes (`01-`, `02-`) to control order.

### Useful Rule Patterns for IBM i Projects

```markdown
# IBM i Coding Standards
- Always use free-format RPG (no fixed-form columns)
- Declare all variables with explicit types and sizes
- Use named indicators via DS (indds) — never raw *IN##
- Prefix all SQL error checks with SQLCODE only (0=success, 100=not found)
- Always include AUT(*EXCLUDE) on new objects
```

> **Tip:** Commit `.bob/rules/` to version control so all team members automatically get the same Bob behavior when they clone the repository.

---

## Bob Interfaces: When to Use What

### Agentic Chat UI vs. Inline Assistant

| Interface | Best For | Use When | Example Tasks |
|-----------|----------|----------|---------------|
| **Agentic Chat UI** | Complex, multi-step tasks | Need planning, analysis, or coordination across files | • Modernizing entire programs<br>• Impact analysis<br>• Architecture design<br>• Multi-file refactoring |
| **Inline Assistant** | Quick, focused edits | Working in a single file with clear intent | • Fix syntax error<br>• Add a procedure<br>• Rename variables<br>• Format code block |

### Agentic Chat UI — Autonomous Task Execution

The **Chat UI** provides Bob with full autonomy to:
- Read and analyze multiple files
- Execute commands and run tests
- Create, modify, and delete files
- Switch between modes as needed
- Plan and execute multi-step workflows
- Spawn subtasks and subagents for parallel work

### Inline Assistant — Direct Code Editing

The **Inline Assistant** (`Ctrl`/`Cmd`+`I` in the editor) provides:
- Quick edits within the current file
- Immediate code generation at cursor
- Focused refactoring without switching context
- Limited to current file context (no cross-file analysis)

**When to use:**
```
✅ "Add error handling to this procedure"
✅ "Convert this fixed-format code block to free format"
✅ "Add comments explaining this calculation"
✅ "Rename this variable to follow naming conventions"
```

### Choosing the Right Interface

**Use Agentic Chat when:**
- Task requires reading/analyzing multiple files
- Need to understand project structure
- Requires running commands or tests
- Involves creating/modifying multiple files
- Need impact analysis or planning

**Use Inline Assistant when:**
- Working within a single file
- Making localized changes
- Quick fixes or additions
- Clear, specific edit at cursor position

---

## Context Mentions (`@`)

Context mentions let you inject precise references into your conversation without copy-pasting. Type `@` in the chat input to see a suggestions dropdown.

### Available Mention Types

| Syntax | What It Provides | Notes |
|--------|-----------------|-------|
| `@/path/to/file.rpgle` | Full file contents with line numbers | Supports text, PDF, DOCX. Large files may be truncated. Use `@/file.rpgle:10-50` for a range |
| `@/path/to/folder` | All text files directly in the folder (non-recursive) | Be mindful of context window limits for large directories |
| `@problems` | Bob Findings panel diagnostics | Useful for "fix all errors" prompts |
| `@terminal` | Recent terminal command and output | Ideal for "fix the error in @terminal" workflows |
| `@git-changes` | `git status` + diff of uncommitted changes | Respects `.gitignore`; useful for commit message generation |
| `@<commit-hash>` | Commit message, author, date, and full diff | e.g., `@a1b2c3d` |
| `@https://example.com` | Fetched content of the URL | Useful for referencing live documentation |

### Important Behaviors

- `@` file and folder mentions **bypass `.bobignore`** — use this when you deliberately need to reference a generated or excluded file
- Git mentions (`@git-changes`, `@<commit-hash>`) respect `.gitignore` since they rely on Git commands
- Combine mentions: `Compare @/src/v1/api.rpgle with @/src/v2/api.rpgle and explain the differences`
- Highlight code and press `Ctrl`/`Cmd`+`L` to add the selected text directly to chat

---

## Context Window Management

### How the Context Window Works

Bob's underlying LLM has a **context window cap of 270,000 tokens** (approximately 200,000 words). Every conversation starts with this full budget, and everything loaded counts against it.

**What fills the context window:**

| Category | What It Contains | Management Tip |
|----------|-----------------|----------------|
| **System prompt** | Bob's core instructions for the session | Fixed — not controllable |
| **Tool definitions** | Built-in tool schemas + connected MCP tool definitions | Disconnect unused MCP servers |
| **Rules** | Content of `AGENTS.md` and `.bob/rules-*` files | Keep rules files concise |
| **Skills** | Instructions from loaded skill SKILL.md files | Load only needed skills |
| **Messages** | Your prompts, Bob's replies, `@` mentions, tool results, command output | Start new conversations for unrelated tasks |

> **Reserved for model response:** ~20,000 tokens are always held back for Bob's next reply.

### Managing the Context Window

```
💡 Hover over the token usage indicator (top-right of chat panel)
   to see a live breakdown of what is consuming your context window.
```

**Best practices to keep context lean:**

- Keep `AGENTS.md` and rules files **short** — include only build commands, test commands, and style rules (e.g., `pnpm test`, `makei build`)
- Connect only the **MCP servers you actually need** for the current task; prefer project-scoped `.bob/mcp.json` over global
- Start a **new conversation** (`+` icon) when switching to an unrelated task — previous conversations are accessible in history but don't consume the new window's budget
- Use **specific `@` file:line-range mentions** instead of mentioning entire large files
- Avoid including large binary files or `node_modules/`-style directories

> The context window is **working memory**, not storage. Reset or start a new conversation when the thread fills with stale output from a previous task.

---

## Auto-Approval Settings

Bob asks for confirmation before each action by default. You can configure auto-approval to speed up workflows — but each level carries different risk.

### Available Actions and Risk Levels

| Action | What It Allows | Risk Level | Recommendation |
|--------|---------------|------------|----------------|
| **Read** | View files and directory contents | Medium | Safe for most projects |
| **Edit** | Create, edit, and save files | **High** | Enable only in controlled environments |
| **Execute** | Run commands in terminal | **High** | Use an allowlist; avoid wildcards |
| **MCP** | Use configured MCP server tools | Medium-High | Only with trusted servers |
| **Skill** | Auto-activate skills without confirmation | Medium | Safe if skills are well-defined |
| **Todo** | Update the task todo list | Low | Safe to auto-approve |
| **Subtask** | Create and complete subtasks | Low | Safe to auto-approve |
| **Subagent** | Spawn subagents for focused tasks | Low | Safe to auto-approve |
| **Mode** | Switch to another mode | Low | Safe to auto-approve |

> **Warning:** Auto-approve for Edit and Execute bypasses all confirmation prompts. This can result in data loss, file corruption, or worse. Never enable these broadly — use task-level approvals instead.

### Hybrid Approach (Recommended)

- **Auto-approve:** `Read`, `Todo`, `Subtask`, `Mode`, `Skill`
- **Manual approval:** `Edit`, `Execute`, `MCP`
- **Editable commands:** Bob lets you edit any proposed command before it runs — use this to review without canceling the task

---

## Slash Commands

Slash commands provide quick access to built-in workflows and custom automations. Type `/` in the chat input to see a searchable, autocomplete menu.

### Built-in Commands

| Command | Description |
|---------|-------------|
| `/init` | Scan project and generate `AGENTS.md` context files |
| `/review` | Review uncommitted changes for bugs, security, performance, and style |
| `/review <branch>` | Compare a branch against current HEAD |
| `/review #<issue>` `--issue-coverage` | Validate changes against a GitHub issue number |
| `/review <issue-url>` `--issue-coverage` | Validate changes against a GitHub issue URL |
| `/create-pr` | Create a pull request directly from Bob |
| `/pr-description` | Generate a PR description from the git diff |

### Custom Slash Commands

Create your own commands by adding Markdown files to `.bob/commands/` (project-scoped) or `~/.bob/commands/` (global). The filename becomes the command name:

```
.bob/commands/
├── ibmi-review.md      → /ibmi-review
├── rpg-convert.md      → /rpg-convert
└── deploy-check.md     → /deploy-check
```

**Example** (`.bob/commands/ibmi-review.md`):
```markdown
Review the current file for IBM i RPG best practices:
- Check for fixed-format RPG that should be converted to free-format
- Identify RLA file I/O that could be replaced with embedded SQL
- Flag raw *IN## indicators that should use named indicator DS fields
- Check for hardcoded library names
```

> **Command priority:** Project-level commands override global commands with the same name.
> Mode-switching commands (`/code`, `/ask`, `/plan`) always take priority and cannot be overridden by custom commands.

---

## Modes — Deep Dive

### Built-in Modes Compared

| Mode | Primary Use | Tool Access | When to Use |
|------|-------------|-------------|-------------|
| **Agent** | Writing and modifying code, implementing features, debugging | `Read`, `Edit`, `Execute`, `MCP`, `Skill`, `Todo`, `Subtask`, `Subagent`, `Mode` | Day-to-day development, refactoring, bug fixes |
| **Plan** | Architecture design, technical planning before implementation | `Read`, `Edit`, `MCP`, `Skill`, `Subagent`, `Mode` | New features, impact analysis, breaking down complex problems |
| **Ask** | Information, explanations, analysis without making changes | `Read`, `MCP`, `Skill`, `Subagent`, `Mode` | Understanding code, exploring concepts, reviewing without modifying |

### Custom Modes

Create specialized modes for team-specific workflows in `.bob/custom_modes.yaml` (project) or `~/.bob/settings/custom_modes.yaml` (global).

**Full mode schema:**
```yaml
customModes:
  - slug: ibmi-reviewer           # Used in /mode ibmi-reviewer command
    name: 🔍 IBM i Reviewer
    description: Reviews IBM i RPG/CL code for modernization opportunities.
    roleDefinition: >-
      You are an IBM i expert specializing in RPG and CL code modernization.
      You analyze legacy OPM/ILE code and identify conversion opportunities.
    whenToUse: Use for reviewing RPG, CL, and DDS files for modernization.
    customInstructions: |-
      Always check for:
      - Fixed-format RPG → free-format conversion opportunities
      - RLA file I/O → embedded SQL refactoring
      - Hardcoded library names and magic numbers
      - Missing error handling on SQL statements
    groups:
      - read
      - mcp
      - skill
      # No 'edit' or 'execute' — review only, no modifications
```

**Available tool groups for `groups`:**

| Group | Permission Granted |
|-------|--------------------|
| `read` | Read files and directories |
| `edit` | Modify files (can be scoped with `fileRegex`) |
| `execute` | Run terminal commands |
| `mcp` | Access MCP servers |
| `skill` | Load skills |
| `workflow` | Launch pre-defined workflows |
| `todo` | Update task todo lists |
| `subtask` | Create subtasks |
| `subagent` | Spawn subagents |
| `mode` | Switch to another mode |

**Restrict edits to specific file types:**
```yaml
groups:
  - read
  - - edit
    - fileRegex: ".*\\.(rpgle|sqlrpgle|clle)$"
      description: RPG and CL source files only
  - execute
```

**Override a default mode** by using its slug (`agent`, `plan`, `ask`):
```yaml
customModes:
  - slug: ask   # Overrides the built-in Ask mode
    name: ❓ Ask
    roleDefinition: You are a knowledgeable IBM i assistant for this project.
    whenToUse: Use this mode to ask questions about the codebase.
    customInstructions: Always reference relevant source files when answering.
    groups:
      - read
      - mcp
      - skill
```

> **Precedence:** Project modes override global modes, which override built-in defaults.

---

## Skills — Deep Dive

### What Skills Are

Skills are **reusable instruction sets** stored as `SKILL.md` files. When Bob activates a skill, it receives the full instructions and gains access to any supporting files in the skill directory. Unlike rules (which always apply), skills are activated on-demand — either automatically by Bob when your request matches the skill's description, or manually.

### Creating a Skill

```
.bob/
└── skills/
    └── rpg-to-free-format/
        ├── SKILL.md             # Instructions Bob follows
        └── checklist.md         # Supporting reference file
```

**`SKILL.md` structure:**
```markdown
---
name: rpg-to-free-format
description: >
  Converts fixed-format ILE RPG source to modern free-format RPG.
  Use when asked to modernize, convert, or refactor RPG source files.
---

# RPG to Free-Format Conversion

## Steps
1. Read the source file to identify the RPG version (II, III, IV fixed)
2. Check the checklist.md for conversion rules
3. Convert H-specs → **CTL-OPT** declarations
4. Convert F-specs → **DCL-F** declarations  
5. Convert D-specs → **DCL-S** / **DCL-DS** declarations
6. Convert C-specs → free-format operation codes
7. Validate: no column-sensitive code remains, compile with CVTOPT(*SHOWCPY)
```

### Skill Scope

| Location | Availability |
|----------|-------------|
| `.bob/skills/<name>/SKILL.md` | Project only (version-controlled) |
| `~/.bob/skills/<name>/SKILL.md` | Global (all projects) |

### Skill Activation

- Bob automatically determines when to activate a skill based on your request and the skill's `description` frontmatter
- Skills load **once per conversation** to avoid duplicate prompts
- Enable **Auto-approve → Skills** in settings to skip the confirmation prompt
- Best practice: load only skills relevant to the current task to avoid consuming context window tokens unnecessarily

---

## MCP (Model Context Protocol) Servers

### What MCP Does

MCP servers act as bridges between Bob and external services (databases, APIs, internal systems). They expose **tools** (callable functions), **resources** (readable data), and **prompts** to Bob.

Bob uses MCP tools when your request involves capabilities provided by configured servers — for example, querying a live IBM i database, interacting with a ticketing system, or calling an internal REST API.

### Configuration Levels

| Level | File Location | Precedence | Use For |
|-------|--------------|------------|---------|
| **Global** | `~/.bob/mcp.json` | Lower | Personal tools used across all projects |
| **Project** | `.bob/mcp.json` | **Higher** (overrides global) | Team-shared tools; version-controlled |

When a server name exists in both files, the **project-level configuration takes precedence**.

### Configuration Format

```json
{
  "mcpServers": {
    "ibmi-db": {
      "command": "node",
      "args": ["/path/to/ibmi-mcp-server/index.js"],
      "env": {
        "IBMI_HOST": "my-system.example.com",
        "IBMI_USER": "DEVELOPER"
      }
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

> **Security:** Never hardcode API keys, tokens, or credentials directly in MCP config files. Use environment variables and add config files containing secrets to `.gitignore`.

### IBM i-Specific MCP Use Cases

- **Live IBM i system access** — read job logs, spool files, object descriptions from a connected system
- **Database queries** — run Db2 for i SQL statements and return results directly into the conversation
- **Build system integration** — trigger `makei` builds and stream output back
- **Custom REST APIs** — call IWS (IBM i Web Services) endpoints exposed by PGMINFO-decorated service programs

---

## `.bobignore` — File Access Control

`.bobignore` controls which files and directories Bob can access through its tools. It uses glob patterns identical to `.gitignore` syntax.

### How It Works

- **Automatically reloaded** — changes take effect immediately (no restart needed in IDE; restart required in Bob Shell)
- **Always self-excluded** — `.bobignore` itself is implicitly ignored; Bob cannot modify its own access rules
- **Tool-level enforcement** — applies to `read_file`, `write_file`, `list_files`, `execute_command` (file-reading commands), and similar operations

### Example `.bobignore` for IBM i Projects

```gitignore
# Generated build objects
*.o
*.module
*.pgm

# Sensitive configuration
.env
secrets/
config/credentials.json
*.key

# Large binary files and archives
*.savf
*.zip
*.tar.gz

# Generated output — do not review
*.lst
*.splf

# Temporary files
tmp/
*.tmp
```

### Limitations

| Limitation | Details |
|------------|---------|
| **Workspace-scoped only** | Applies only to files inside the current workspace root |
| **`@` mentions bypass it** | Direct `@/path/to/file` mentions override `.bobignore` and include the file anyway |
| **Not a full sandbox** | Does not create a system-level sandbox — determined users can always access files outside Bob's tools |
| **Some write tools** | `insert_content` and `search_and_replace` may be able to write to ignored files due to tool implementation details |

---

## Code Review and GitHub Integration

### Code Review with `/review`

Bob provides automated code review that runs entirely within the IDE. Reviews run with **auto-approval** — no manual confirmation needed during analysis.

**Review command variants:**
```
/review                               # Review uncommitted local changes
/review <branch-name>                 # Compare branch against current HEAD
/review #123 --issue-coverage         # Validate changes against GitHub issue #123
/review https://github.com/.../123 --issue-coverage  # Validate by issue URL
```

**Review categories:**
- **Maintainability** — Code quality, naming, DRY violations, modularity
- **Security** — Vulnerabilities, hardcoded credentials, input validation
- **Performance** — Inefficient algorithms, memory/resource leaks
- **Functionality** — Logic errors, edge cases, error handling
- **Style** — Formatting inconsistencies, documentation gaps

**Bob Findings panel:**
- Categorized issues with severity levels (Critical / High / Medium / Low)
- Click any finding to navigate to the issue location
- Track resolution status across the review session

> **Note:** Branch comparisons work with both GitHub and GitLab. Issue validation (`--issue-coverage`) requires GitHub.

### Pull Request Workflow

```
/create-pr          # Full PR creation flow: select base branch → review description → create
/pr-description     # Generate PR description only (review before creating manually)
```

**PR template auto-detection** — Bob searches for a `pull_request_template.md` in these locations (in order):
1. `pull_request_template.md` (project root)
2. `docs/pull_request_template.md`
3. `.github/pull_request_template.md`
4. `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md`

If multiple templates are found, Bob asks which one to use.

---

## Best Practices for Large IBM i Projects

**Bob automatically handles:**
- Large file chunking (no manual intervention needed)
- Incremental diff application for large programs

### 1. Modernization Workflow

1. **Document** (IBM i Developer mode)
   - Understand existing business logic
   - Identify dependencies

2. **Analyze Impact** (Plan mode + `@` file mentions)
   - Check program/file dependencies
   - Assess change scope before touching code

3. **Convert Incrementally** (Agent mode)
   - Fixed→free format conversion
   - RLA→SQL refactoring
   - One program/module at a time

4. **Test** (Agent mode + RPGUnit)
   - Create RPGUnit test suites
   - Validate conversions compile and produce correct results

5. **Automate** (Agent mode + CI/CD)
   - Set up `makei`-based build pipelines
   - Configure automated testing

### 2. Effective Prompt Strategies

```
✅ "Convert the D-spec declarations in @/QRPGLESRC/ART200.RPGLE:1-50 to free-format"
✅ "Analyze @/QRPGLESRC/ART200.RPGLE and list all F-specs that use RLA — suggest SQL equivalents"
✅ "@problems Fix all compile errors in the current file"
✅ "Review @git-changes and check for missing SQLCODE error handling"
✅ "Compare the SAMREF reference fields in @/common/SAMREF.PF with the fields in @/QDDSSRC/ART100.PF"
```

### 3. Common Pitfalls to Avoid

❌ **Don't:** Ask Bob to read entire application or 10,000-line programs by default  
✅ **Do:** Use `@/file.rpgle:start-end` line ranges for the specific sections you need

❌ **Don't:** Mix multiple unrelated tasks in one conversation  
✅ **Do:** Start new conversations (`+`) for unrelated tasks to avoid context pollution

❌ **Don't:** Skip `/init`  
✅ **Do:** Run `/init` and manually add IBM i-specific business rules to `AGENTS.md`

❌ **Don't:** Put large amounts of generated output, file listings, or logs in `AGENTS.md`  
✅ **Do:** Keep `AGENTS.md` concise — build commands, naming conventions, key architectural decisions only

❌ **Don't:** Enable auto-approve for Edit and Execute globally  
✅ **Do:** Use task-level approvals and review commands before they run

❌ **Don't:** Hardcode IBM i system credentials in `.bob/mcp.json`  
✅ **Do:** Use environment variables for all credentials; add credential files to `.bobignore` and `.gitignore`

---

## Quick Start Checklist

- [ ] Run `/init` to create `AGENTS.md` files for all modes
- [ ] Review and supplement generated `.bob/rules-*/AGENTS.md` files with IBM i conventions
- [ ] Create `.bob/rules/ibmi-standards.md` with project coding standards
- [ ] Create `.bobignore` for generated objects, temp files, and sensitive config
- [ ] Create `.bob/mcp.json` for any IBM i system connections (use env vars for credentials)
- [ ] Choose appropriate mode for your task (Agent for code, Plan for architecture, Ask for questions)
- [ ] Use `@` context mentions instead of pasting code — prefer file references with line ranges
- [ ] Run impact analysis (`/review` + Plan mode) before major refactoring
- [ ] Create RPGUnit tests for converted code and run them to validate
- [ ] Configure auto-approve selectively: enable Read/Todo/Mode, keep Edit/Execute on manual
- [ ] Commit `.bob/rules/`, `.bob/custom_modes.yaml`, and `.bob/skills/` to version control for team sharing

---

## Additional Resources

- **IBM Bob Documentation**: https://bob.ibm.com/docs/
