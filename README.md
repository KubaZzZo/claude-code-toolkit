# Claude Code Toolkit

Personal Claude Code engineering system — rules, custom agents, and workflow skills.

## Structure

```
├── rules/           # Coding & workflow rules (per-language + common)
│   ├── common/      # Shared across all projects
│   ├── golang/
│   ├── python/
│   ├── typescript/
│   └── perl/
├── agents/          # Custom agent definitions (11 agents)
├── skills/          # Custom workflow skills
│   └── dev-workflow/ # Multi-agent orchestration pipeline
└── settings.template.json  # Reference config (no secrets)
```

## Agents

| Agent | Model | Role |
|-------|-------|------|
| pm-orchestrator | haiku | Task analysis & pipeline scheduling |
| researcher | haiku | Codebase research |
| github-researcher | haiku | GitHub reference search |
| architect | opus | System design (complex tasks) |
| developer | sonnet | Code implementation |
| tdd-engineer | sonnet | TDD red/green phases |
| reviewer | haiku | Code quality review |
| qa-engineer | haiku | Test execution & regression |
| security-reviewer | opus | OWASP security review |
| performance-reviewer | haiku | DB/loop/cache review |
| doc-writer | haiku | Documentation sync |

## Pipeline

```
simple:  researcher → developer → tdd-engineer(green) → qa-engineer
normal:  researcher → architect → tdd-engineer(red) → developer → tdd-engineer(green) → reviewer → qa-engineer
complex: normal + security-reviewer + performance-reviewer
```

## Setup

```bash
# Clone to your .claude directory
git clone https://github.com/KubaZzZo/claude-code-toolkit.git

# Symlink what you need
ln -s $(pwd)/claude-code-toolkit/rules ~/.claude/rules
ln -s $(pwd)/claude-code-toolkit/agents ~/.claude/agents
ln -s $(pwd)/claude-code-toolkit/skills ~/.claude/skills
```
