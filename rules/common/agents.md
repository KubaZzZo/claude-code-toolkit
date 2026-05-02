# Agent Orchestration

## Available Agents

Located in `~/.claude/agents/`:

| Agent | Model | Purpose | When to Use |
|-------|-------|---------|-------------|
| pm-orchestrator | opus | Task analysis, pipeline scheduling | Every task (entry point) |
| researcher | haiku | Codebase research | Before any implementation |
| github-researcher | haiku | GitHub reference search | No local pattern exists |
| architect | opus | System design | Complex features, cross-module changes |
| developer | opus | Code implementation | All code changes |
| tdd-engineer | sonnet | TDD red/green phases | New features, bug fixes |
| reviewer | sonnet | Code quality review | After code is written |
| qa-engineer | haiku | Test execution, regression | After developer + tdd-engineer |
| security-reviewer | sonnet | Security analysis | Auth, user input, DB, payments |
| performance-reviewer | haiku | DB/loop/cache review | Code touching DB, loops, caches |
| doc-writer | haiku | Documentation sync | Public API or functional changes |
| build-error-resolver | sonnet | Fix build errors | When build fails |
| e2e-runner | haiku | E2E testing | Critical user flows |
| refactor-cleaner | haiku | Dead code cleanup | Code maintenance |
| rust-reviewer | opus | Rust code review | Rust projects |
| contest-reviewer | sonnet | Contest solution review | Contest/leetcode mode |

## Immediate Agent Usage

No user prompt needed:
1. Complex feature requests - Use **pm-orchestrator** (always first), then follow execution plan
2. Code just written/modified - Use **reviewer** agent
3. Bug fix or new feature - Use **tdd-engineer** agent for red/green cycle
4. Architectural decision - Use **architect** agent
5. Build failure - Use **build-error-resolver** agent

## Parallel Task Execution

ALWAYS use parallel Agent execution for independent operations:

- **researcher + github-researcher**: Both start simultaneously when github-researcher trigger is met
- **reviewer + security-reviewer + performance-reviewer**: All three run in parallel after developer completes
- pm-orchestrator merges results from parallel agents before continuing pipeline

## Multi-Perspective Analysis

For complex problems, use split role review in parallel:
- reviewer (code quality)
- security-reviewer (OWASP Top 10)
- performance-reviewer (N+1, pagination, caching)
