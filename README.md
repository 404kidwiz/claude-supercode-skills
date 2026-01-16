# Claude Supercode Skills

A comprehensive collection of **161 AI agent skills** for Claude Code, Gemini CLI, and Google Antigravity. All skills are fully compliant with Anthropic's official Agent Skills specification.

## Quick Start

### Claude Code
```bash
# Copy all skills to your Claude Code skills folder
cp -r *-skill ~/.claude/skills/
```

### Gemini CLI
```bash
# Switch to gemini-cli branch and copy
git checkout gemini-cli
cp -r *-skill ~/.gemini/skills/
```

### Google Antigravity
```bash
# Switch to antigravity branch and copy
git checkout antigravity
cp -r *-skill ~/.gemini/antigravity/skills/
```

## Branches

| Branch | Platform | Skill File | Location |
|--------|----------|------------|----------|
| `main` | Claude Code | `SKILL.md` | `~/.claude/skills/` |
| `gemini-cli` | Gemini CLI | `skill.yaml` | `~/.gemini/skills/` |
| `antigravity` | Google Antigravity | `skill.yaml` | `~/.gemini/antigravity/skills/` |

## Skills Overview

### Categories

| Category | Count | Examples |
|----------|-------|----------|
| **Frontend Development** | 15 | react-specialist, vue-expert, nextjs-developer, frontend-ui-ux-engineer |
| **Backend Development** | 18 | python-pro, golang-pro, java-architect, django-developer, rails-expert |
| **DevOps & Infrastructure** | 14 | kubernetes-specialist, terraform-engineer, devops-engineer, sre-engineer |
| **Data & ML** | 12 | data-scientist, ml-engineer, llm-architect, data-engineer, mlops-engineer |
| **Security** | 8 | security-auditor, penetration-tester, ad-security-reviewer, incident-responder |
| **Architecture & Design** | 10 | solution-architect, microservices-architect, api-designer, graphql-architect |
| **Project Management** | 6 | project-manager, scrum-master, product-manager, business-analyst |
| **Documentation** | 5 | technical-writer, document-writer, api-documenter, documentation-engineer |
| **Testing & QA** | 6 | test-automator, qa-expert, accessibility-tester, performance-testing |
| **Specialized** | 67 | threejs-pro, quant-analyst, game-developer, blockchain-developer, video-engineer |

### Complete Skills List (161 total)

<details>
<summary>Click to expand full list</summary>

#### A-C
- accessibility-tester, ad-security-reviewer, agent-organizer, ai-engineer
- algorithmic-art, angular-architect, api-designer, api-documenter
- architect-reviewer, azure-infra-engineer, backend-developer, blockchain-developer
- bmad-bmm-analyst, bmad-master, build-engineer, build, business-analyst
- canvas-design, chaos-engineer, cli-developer, cloud-architect, code-reviewer
- codebase-exploration, competitive-analyst, compliance-auditor, content-marketer
- context-manager, cpp-pro, csharp-developer, csv-data-wrangler, customer-success-manager

#### D-G
- data-analyst, data-engineer, data-researcher, data-scientist
- database-administrator, database-optimizer, debugger, dependency-manager
- deployment-engineer, devops-engineer, devops-incident-responder, django-developer
- document-writer, documentation-engineer, docx, dotnet-core-expert
- dotnet-framework-4.8-expert, dx-optimizer, electron-pro, elixir-expert
- embedded-systems, error-coordinator, error-detective, error-detector
- event-driven-architect, explore, fintech-engineer, flutter-expert
- frontend-developer, frontend-ui-ux-engineer, fullstack-developer, game-developer
- general, git-workflow-manager, golang-pro, graphql-architect

#### H-M
- incident-responder, internal-comms, iot-engineer, it-ops-orchestrator
- java-architect, javascript-pro, kafka-engineer, knowledge-synthesizer
- kotlin-specialist, kubernetes-specialist, laravel-specialist, legacy-modernizer
- legal-advisor, librarian, llm-architect, m365-admin, machine-learning-engineer
- macos-developer, market-researcher, mcp-developer, microservices-architect
- ml-engineer, mlops-engineer, mobile-app-developer, mobile-developer
- multi-agent-coordinator, multimodal-looker

#### N-R
- network-engineer, nextjs-developer, nlp-engineer, oracle
- payment-integration, pdf, penetration-tester, performance-engineer
- performance-monitor, performance-testing, php-pro, plan, platform-engineer
- postgres-pro, powershell-5.1-expert, powershell-7-expert, powershell-module-architect
- powershell-security-hardening, powershell-ui-architect, pptx, product-manager
- project-manager, prompt-engineer, python-pro, qa-expert, quant-analyst
- rails-expert, react-native-specialist, react-specialist, refactoring-specialist
- research-analyst, risk-manager, rust-engineer

#### S-Z
- sales-engineer, scrum-master, search-specialist, security-auditor, security-engineer
- seo-specialist, skill-creator, slack-expert, solution-architect, spring-boot-engineer
- sql-pro, sre-engineer, swift-expert, task-distributor, technical-advisory
- technical-writer, terraform-engineer, test-automator, theme-factory, threejs-pro
- tooling-engineer, trend-analyst, typescript-pro, ui-designer, ux-researcher
- video-engineer, vue-expert, websocket-engineer, windows-app-developer
- windows-infra-admin, wordpress-master, workflow-orchestrator, xlsx

</details>

## Skill Structure

### Claude Code (main branch)
```
skill-name-skill/
├── SKILL.md           # Main skill file (required)
├── REFERENCE.md       # Technical details (optional)
└── EXAMPLES.md        # Code examples (optional)
```

### Gemini CLI / Antigravity (gemini-cli / antigravity branches)
```
skill-name-skill/
├── skill.yaml         # Main skill file (required)
├── REFERENCE.md       # Technical details (optional)
└── EXAMPLES.md        # Code examples (optional)
```

## Anthropic Compliance

All skills conform to [Anthropic's official Agent Skills specification](https://docs.claude.com/en/docs/claude-code/skills):

| Requirement | Status |
|-------------|--------|
| File named `SKILL.md` (case-sensitive) | ✅ |
| YAML frontmatter with `name` and `description` | ✅ |
| Name: lowercase, hyphens, max 64 chars | ✅ |
| Description: third-person voice, max 1024 chars | ✅ |
| SKILL.md under 500 lines | ✅ |
| Progressive disclosure for large skills | ✅ |

See [ANTHROPIC-COMPLIANCE-REPORT.md](ANTHROPIC-COMPLIANCE-REPORT.md) for full audit.

## Progressive Disclosure

25 skills use progressive disclosure to keep main files under 500 lines:

| Skill | SKILL.md | REFERENCE.md | EXAMPLES.md |
|-------|----------|--------------|-------------|
| rails-expert | 151 lines | ✅ | ✅ |
| backend-developer | 203 lines | ✅ | ✅ |
| ui-designer | 205 lines | ✅ | ✅ |
| frontend-developer | 218 lines | ✅ | ✅ |
| sql-pro | 159 lines | ✅ | ✅ |
| ... and 20 more | | | |

## Installation

### Option 1: Clone and Copy
```bash
# Clone repository
git clone https://github.com/404kidwiz/claude-supercode-skills.git
cd claude-supercode-skills

# For Claude Code
cp -r *-skill ~/.claude/skills/

# For Gemini CLI
git checkout gemini-cli
cp -r *-skill ~/.gemini/skills/

# For Antigravity
git checkout antigravity
cp -r *-skill ~/.gemini/antigravity/skills/
```

### Option 2: Symlink (recommended for updates)
```bash
# Clone once
git clone https://github.com/404kidwiz/claude-supercode-skills.git ~/claude-supercode-skills

# Symlink for Claude Code
ln -s ~/claude-supercode-skills/*-skill ~/.claude/skills/

# For Gemini/Antigravity, checkout appropriate branch first
```

## Usage Examples

### Claude Code
```
> What skills are available?
> Use the sql-pro skill to optimize this query
> Review this code using the code-reviewer skill
```

### Gemini CLI
```bash
gemini skills list
gemini "Optimize this SQL query"  # Auto-invokes sql-pro skill
```

### Antigravity
Skills are auto-discovered based on task context.

## Skill File Format

### YAML Frontmatter (required)
```yaml
---
name: skill-name
description: Third-person description of what the skill does and when to use it. Include trigger terms users would naturally say.
---
```

### Markdown Body
```markdown
# Skill Title

## Purpose
Brief description of the skill's purpose.

## When to Use
- Scenario 1
- Scenario 2

## Quick Start
**Invoke this skill when:**
- Condition 1
- Condition 2

**Do NOT invoke when:**
- Use other-skill instead for X
- Use another-skill for Y

## Decision Framework
[Decision tree or matrix]

## Core Workflows
[Step-by-step workflows]

## Best Practices
[Domain-specific recommendations]

## Anti-Patterns
[Common mistakes to avoid]

## Additional Resources
- See [REFERENCE.md](REFERENCE.md) for technical details
- See [EXAMPLES.md](EXAMPLES.md) for code examples
```

## Contributing

1. Fork the repository
2. Create a new skill directory: `your-skill-skill/`
3. Create `SKILL.md` following the format above
4. Ensure compliance with Anthropic spec:
   - Name: lowercase, hyphens only, max 64 chars
   - Description: third-person, max 1024 chars
   - Content: under 500 lines (use progressive disclosure if needed)
5. Submit a pull request

## Sync Script

Keep all platforms in sync:

```bash
#!/bin/bash
# ~/sync-skills.sh

# Sync Claude skills to Gemini/Antigravity
for dir in ~/.claude/skills/*-skill; do
  skill=$(basename "$dir")
  
  # Gemini CLI
  mkdir -p ~/.gemini/skills/$skill
  cp "$dir/SKILL.md" ~/.gemini/skills/$skill/skill.yaml
  cp "$dir"/*.md ~/.gemini/skills/$skill/ 2>/dev/null
  
  # Antigravity
  mkdir -p ~/.gemini/antigravity/skills/$skill
  cp "$dir/SKILL.md" ~/.gemini/antigravity/skills/$skill/skill.yaml
  cp "$dir"/*.md ~/.gemini/antigravity/skills/$skill/ 2>/dev/null
done

echo "Synced $(ls ~/.claude/skills | grep -c skill) skills"
```

## Platform Documentation

| Platform | Documentation |
|----------|---------------|
| Claude Code | https://docs.claude.com/en/docs/claude-code/skills |
| Gemini CLI | https://geminicli.com/docs/cli/skills/ |
| Antigravity | https://antigravity.google/docs/skills |

## License

MIT License - Feel free to use and modify for your projects.

## Acknowledgments

- [Anthropic](https://anthropic.com) for Claude Code and Agent Skills specification
- [Google](https://google.com) for Gemini CLI and Antigravity
- Community contributors

---

**161 skills | 3 platforms | Full Anthropic compliance**
