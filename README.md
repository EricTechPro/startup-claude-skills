# Startup Claude Skills

A curated collection of reusable [Claude Code](https://claude.com/claude-code) skills for launching and running a tech startup. Drop these into any project's `.claude/skills/` directory and start using them immediately.

## What Are Claude Code Skills?

Skills are reusable prompt templates that teach Claude Code how to perform complex, multi-step workflows autonomously. Think of them as "recipes" that turn Claude into a specialized agent for specific tasks.

## Skills Included

| Skill | Description | Use Case |
|-------|-------------|----------|
| **fix-ticket** | End-to-end bug fix pipeline: read Jira ticket, research, implement fix, review, commit, push, deploy, update ticket | Bug fixing with full CI/CD integration |
| **fix-issue** | End-to-end GitHub issue fix pipeline: read GitHub issue, research, implement fix, review, commit, push, create PR, comment on issue | Bug fixing for GitHub-native workflows |
| **develop-team** | Full feature development with parallel research agents, planning, phased implementation, review, and PR creation | Feature development from Jira ticket to PR |
| **review-team** | 5-agent PR review with Devil's Advocate adversarial filtering. Only findings that survive cross-examination make the final report | High-confidence PR reviews |
| **review-fix** | Automated review-fix loop: 8 parallel reviewers, auto-fix quick items, accumulate strategic items | Pre-PR code quality automation |
| **playwright-qa-cli** | Headless browser QA testing via `playwright-cli`. Provisions test users, logs in, navigates, screenshots, generates reports | Automated QA verification |

PRs welcome for additional skills!

## Quick Start

### 1. Copy a skill into your project

```bash
# Copy the fix-ticket skill
cp -r skills/fix-ticket /path/to/your-project/.claude/skills/

# Copy the config template
cp CONFIG.template.md /path/to/your-project/.claude/skills/fix-ticket/CONFIG.md
```

### 2. Configure it

Edit `CONFIG.md` with your project-specific values (Jira project key, team members, deployment platform, etc.).

### 3. Use it

```
/fix-ticket PROJ-123
```

## Configuration

Each skill uses a `CONFIG.md` file for project-specific settings. A `CONFIG.template.md` is provided with all available options documented. Copy it to `CONFIG.md` and fill in your values.

**Your `CONFIG.md` should NOT be committed to the skills repo** — it contains project-specific (and potentially sensitive) information. It's already in `.gitignore`.

## Project Structure

```
startup-claude-skills/
  README.md
  CONFIG.template.md              # Template for project-specific config
  .gitignore
  skills/
    fix-ticket/                   # End-to-end Jira bug fix pipeline
      SKILL.md
      references/
        qa-integration.md
    fix-issue/                    # End-to-end GitHub issue fix pipeline
      SKILL.md
      references/
        qa-integration.md
    develop-team/                 # Full feature development pipeline
      SKILL.md
      references/
        agent-prompts.md
        plan-template.md
        state-schema.md
    review-team/                  # PR review with Devil's Advocate
      SKILL.md
    review-fix/                   # Automated review-fix loop
      SKILL.md
      README.md
    playwright-qa-cli/            # Headless browser QA testing
      SKILL.md
      references/
        cli-patterns.md
        cleanup.md
        provision.md
        report-templates.md
```

## Requirements

These skills work with [Claude Code](https://claude.com/claude-code) and may use:

- **GitHub CLI (`gh`)** — for issue/PR management in fix-issue
- **Jira MCP** — for ticket management (`mcp__jira__*`)
- **Vercel MCP** — for deployment monitoring (`mcp__claude_ai_Vercel__*`)
- **Supabase MCP** — for database operations (`mcp__supabase__*` or `mcp__plugin_supabase_supabase__*`)
- **Playwright CLI** — for headless browser QA testing (`playwright-cli`)

Skills gracefully degrade when optional integrations aren't available — they'll warn you and skip those phases.

## Contributing

1. Fork this repo
2. Add your skill in `skills/<skill-name>/SKILL.md`
3. Add a config section to `CONFIG.template.md` if your skill needs project-specific values
4. Update the skills table in this README
5. Open a PR

### Skill Guidelines

- Skills should be **generic** — no hardcoded project names, credentials, or IDs
- Use `CONFIG.md` for all project-specific values
- Include clear `## When to Use` and `## Parameters` sections
- Document error handling and graceful degradation
- Add evaluation criteria so users can verify the skill works correctly

## License

MIT
