# Claude Code Project Structure

> Source: https://thisweb.dev/post/claude-code-structure
> Summarized: 2026-04-14

The `.claude/` folder is the settings center for Claude Code. Seven components to organize:

---

## 1. CLAUDE.md

The primary project knowledge file — read at session start automatically.

**What to include:**
- Tech stack and toolchain
- Directory structure and key entry points
- Development standards and conventions
- Deploy workflow
- Credential/security rules
- Team conventions (commit style, etc.)

**Placement:**
- `~/.claude/CLAUDE.md` — global defaults for all projects
- `.claude/CLAUDE.md` or `CLAUDE.md` — project-specific (committed, shared with team)

Generate a starting point with `/init`.

---

## 2. Rules

Behavior guidelines with optional **path filtering** — rules only load when Claude touches matching files. Keeps context lean for large projects.

**Location:** `.claude/rules/`

**Format (frontmatter):**
```markdown
---
description: Short description
globs: ["src/frontend/**", "*.tsx"]
---

Rule content here...
```

**Best practice:** Separate frontend, backend, database rules into different files. Each appears only when needed.

---

## 3. Commands

Quick `/command-name` shortcuts — define a workflow once, invoke it anytime.

**Location:** `.claude/commands/`

**Format:** Markdown file named after the command. E.g., `deploy.md` → `/deploy`

**Use cases:**
- Deploy checklists
- Release workflows
- Code review steps
- Any multi-step process you repeat

---

## 4. Hooks

Automated triggers that run scripts at defined moments.

**Configured in:** `settings.json` under `"hooks"`

**Event types:**
- `PreToolUse` — before Claude uses a tool
- `PostToolUse` — after Claude uses a tool

**Format:**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash /path/to/check.sh"
          }
        ]
      }
    ]
  }
}
```

**Common use cases:**
- Pre-commit validation (credential leak check, lint)
- Post-edit formatting
- Session start initialization

---

## 5. Agents

Custom AI specialists for specific tasks — their own role, tools, and behavior rules.

**Location:** `.claude/agents/`

**When to use:** Distinct workflows requiring different system prompts or tool restrictions. E.g., a code-review agent that only reads (no writes).

**Triggered by:** Task descriptions matching the agent's description, direct commands, or `@mentions`.

---

## 6. Skills

Reusable workflow procedures that **auto-load when relevant** — consuming tokens only when needed.

**Structure:** Folder-based with a `SKILL.md` configuration file.

**vs. Commands:** Skills are procedural templates for complex workflows; commands are simple shortcuts.

---

## 7. Settings.json

Permission controls and global configuration.

**Placement:**
- `~/.claude/settings.json` — global
- `.claude/settings.json` — project (committed)
- `.claude/settings.local.json` — project-local (gitignored)

**Key features:**
- `permissions.allow` / `permissions.deny` — allowlists and blocklists for tool use
- `hooks` — automated triggers (see above)

---

## Project Structure Summary

```
.claude/
├── settings.json          # Committed permissions + hooks
├── settings.local.json    # Local-only permissions (gitignored)
├── rules/
│   ├── frontend.md        # Path-filtered: src/frontend/**
│   ├── backend.md         # Path-filtered: src/api/**
│   └── patient-data.md    # Path-filtered: **/*.py
├── commands/
│   └── deploy.md          # /deploy command
├── hooks/
│   └── pre-commit-check.sh
└── agents/
    └── reviewer.md

CLAUDE.md                  # Root project knowledge (committed)
```

---

## Key Insights

- **Rules with path globs** are the highest-leverage feature for monorepos — context stays lean
- **CLAUDE.md** replaces repeated explanations at session start
- **Hooks** are Claude's tool events, not git hooks (though you can trigger git checks from them)
- Global (`~/.claude/`) vs project (`.claude/`) placement enables personal defaults + team configs
- Don't over-engineer for small solo projects — CLAUDE.md + a few rules covers most needs
