# Claude Supercode Skills

**161 AI Agent Skills for Claude Code, Gemini CLI, and Google Antigravity**

A comprehensive collection of specialized AI coding skills that extend the capabilities of AI coding assistants. Each skill provides domain-specific knowledge, workflows, and best practices for professional software development.

## Platforms Supported

| Platform | Branch | Skill File | Install Location |
|----------|--------|------------|------------------|
| **Claude Code** | `main` | `SKILL.md` | `~/.claude/skills/` |
| **Gemini CLI** | `gemini-cli` | `skill.yaml` | `~/.gemini/skills/` |
| **Antigravity** | `antigravity` | `skill.yaml` | `~/.gemini/antigravity/skills/` |

## Quick Install

### Claude Code (Anthropic)

```bash
# Clone main branch
git clone https://github.com/404kidwiz/claude-supercode-skills.git ~/.claude/skills

# Or add specific skills
git clone https://github.com/404kidwiz/claude-supercode-skills.git /tmp/skills
cp -r /tmp/skills/python-pro-skill ~/.claude/skills/
```

### Gemini CLI

```bash
# Clone gemini-cli branch
git clone -b gemini-cli https://github.com/404kidwiz/claude-supercode-skills.git ~/.gemini/skills
```

### Google Antigravity

```bash
# Clone antigravity branch
mkdir -p ~/.gemini/antigravity
git clone -b antigravity https://github.com/404kidwiz/claude-supercode-skills.git ~/.gemini/antigravity/skills
```

## Skills by Category

### Development - Languages (21)

| Skill | Description |
|-------|-------------|
| `cpp-pro-skill` | C++20 specialist with performance optimization and system programming |
| `csharp-developer-skill` | .NET 8 and C# 12 with ASP.NET Core and EF Core |
| `dotnet-core-expert-skill` | .NET 8 cross-platform with MAUI and modern C# |
| `dotnet-framework-4.8-expert-skill` | Legacy .NET Framework with WCF and ASP.NET MVC |
| `elixir-expert-skill` | Elixir, Phoenix, and OTP for concurrent applications |
| `golang-pro-skill` | Go 1.21+ with goroutines, channels, and stdlib |
| `java-architect-skill` | Java 21, Spring Boot 3, and Jakarta EE |
| `javascript-pro-skill` | ES2023+, Node.js, and async patterns |
| `kotlin-specialist-skill` | Kotlin 2.0, KMP, Coroutines, and Ktor |
| `laravel-specialist-skill` | Laravel 11+, Octane, and Livewire 3 |
| `php-pro-skill` | PHP 8.2+ with modern patterns and Composer |
| `python-pro-skill` | Python 3.11+, type annotations, and FastAPI |
| `rails-expert-skill` | Rails 7+ with Hotwire, Turbo, and Stimulus |
| `rust-engineer-skill` | Rust async, ownership patterns, FFI, and WASM |
| `spring-boot-engineer-skill` | Spring Boot 3+, Virtual Threads, and Cloud Native |
| `swift-expert-skill` | iOS/macOS with SwiftUI and Combine |
| `typescript-pro-skill` | TypeScript 5+ advanced types and generics |

### Development - Frontend (12)

| Skill | Description |
|-------|-------------|
| `angular-architect-skill` | Angular 15+ enterprise patterns and NgRx |
| `frontend-developer-skill` | Modern frontend development tools and patterns |
| `frontend-ui-ux-engineer-skill` | Designer-developer for stunning UI/UX |
| `nextjs-developer-skill` | Next.js 14+, App Router, and Server Components |
| `react-native-specialist-skill` | React Native cross-platform mobile development |
| `react-specialist-skill` | React 18+, hooks, concurrent features, Zustand |
| `threejs-pro-skill` | Three.js, WebGL, React Three Fiber, and WebGPU |
| `vue-expert-skill` | Vue 3 Composition API, Pinia, and Nuxt.js |
| `canvas-design-skill` | Visual art in PNG/PDF using design philosophy |
| `algorithmic-art-skill` | Generative art with p5.js and seeded randomness |
| `ui-designer-skill` | Visual UI design and component systems |
| `theme-factory-skill` | Styling artifacts with preset themes |

### Development - Backend (8)

| Skill | Description |
|-------|-------------|
| `backend-developer-skill` | Production-ready server-side applications |
| `django-developer-skill` | Django 4+, DRF, PostgreSQL, and Celery |
| `graphql-architect-skill` | GraphQL schema, federation, and resolvers |
| `api-designer-skill` | REST/GraphQL API with OpenAPI 3.1 |
| `websocket-engineer-skill` | Real-time WebSocket and Socket.IO systems |
| `kafka-engineer-skill` | Apache Kafka streaming and event processing |
| `event-driven-architect-skill` | Async systems with AsyncAPI and CloudEvents |
| `payment-integration-skill` | Stripe, PayPal, Adyen integration |

### Development - Mobile (4)

| Skill | Description |
|-------|-------------|
| `flutter-expert-skill` | Flutter 3+, Dart, and Firebase integration |
| `mobile-app-developer-skill` | Native/cross-platform iOS and Android |
| `mobile-developer-skill` | React Native and Flutter development |
| `macos-developer-skill` | macOS development with SwiftUI |

### Development - Desktop (3)

| Skill | Description |
|-------|-------------|
| `electron-pro-skill` | Electron desktop apps with auto-update |
| `windows-app-developer-skill` | WinUI 3, WPF, and Windows App SDK |
| `game-developer-skill` | Unity, Unreal, and custom game engines |

### Development - Fullstack (2)

| Skill | Description |
|-------|-------------|
| `fullstack-developer-skill` | Frontend-backend integration and architecture |
| `mcp-developer-skill` | Model Context Protocol servers and clients |

### Architecture & Design (9)

| Skill | Description |
|-------|-------------|
| `architect-reviewer-skill` | System design validation and pattern assessment |
| `cloud-architect-skill` | AWS, Azure, GCP multi-cloud strategies |
| `microservices-architect-skill` | Service decomposition and orchestration |
| `solution-architect-skill` | Enterprise solutions with TOGAF adaptation |
| `legacy-modernizer-skill` | Strangler Fig, CDC, and anti-corruption layers |
| `refactoring-specialist-skill` | Design patterns and SOLID principles |
| `llm-architect-skill` | LLM deployment and optimization strategies |
| `workflow-orchestrator-skill` | Temporal, Camunda, and event-driven workflows |
| `context-manager-skill` | Vector databases, RAG, and context optimization |

### DevOps & Infrastructure (16)

| Skill | Description |
|-------|-------------|
| `devops-engineer-skill` | CI/CD automation and SRE practices |
| `deployment-engineer-skill` | CI/CD with Jenkins, GitHub Actions, GitLab |
| `kubernetes-specialist-skill` | K8s orchestration across EKS, AKS, GKE |
| `terraform-engineer-skill` | Infrastructure as Code and cloud architecture |
| `azure-infra-engineer-skill` | Azure infrastructure and Bicep IaC |
| `platform-engineer-skill` | Internal developer platforms and self-service |
| `sre-engineer-skill` | SLOs, error budgets, and incident management |
| `build-engineer-skill` | Build systems and optimization |
| `build-skill` | Compilation and packaging specialist |
| `chaos-engineer-skill` | Failure injection and resilience testing |
| `devops-incident-responder-skill` | Incident detection and root cause analysis |
| `incident-responder-skill` | Security and operational incident response |
| `network-engineer-skill` | Network architecture and zero-trust |
| `video-engineer-skill` | Video processing and streaming protocols |
| `iot-engineer-skill` | Edge computing and sensor networks |
| `embedded-systems-skill` | RTOS, microcontrollers, and firmware |

### Data & AI/ML (14)

| Skill | Description |
|-------|-------------|
| `ai-engineer-skill` | LLMs, RAG, and autonomous agents |
| `data-analyst-skill` | BI, visualization, and statistical analysis |
| `data-engineer-skill` | Scalable data pipelines and ETL/ELT |
| `data-researcher-skill` | Data discovery and insight extraction |
| `data-scientist-skill` | ML, EDA, and predictive modeling |
| `machine-learning-engineer-skill` | ML model deployment and inference |
| `ml-engineer-skill` | ML system development and pipelines |
| `mlops-engineer-skill` | ML lifecycles and production monitoring |
| `nlp-engineer-skill` | NLP with Hugging Face, spaCy, LangChain |
| `prompt-engineer-skill` | Prompt design and Chain-of-Thought |
| `quant-analyst-skill` | Algorithmic trading and financial analysis |
| `csv-data-wrangler-skill` | High-performance CSV processing |
| `knowledge-synthesizer-skill` | Ontology building and GraphRAG |
| `trend-analyst-skill` | Forecasting and market intelligence |

### Databases (5)

| Skill | Description |
|-------|-------------|
| `database-administrator-skill` | PostgreSQL, MySQL, MongoDB HA and tuning |
| `database-optimizer-skill` | Query optimization and index strategies |
| `postgres-pro-skill` | PostgreSQL administration and optimization |
| `sql-pro-skill` | SQL development across major platforms |
| `fintech-engineer-skill` | Double-entry ledgers and high-precision math |

### Security (9)

| Skill | Description |
|-------|-------------|
| `security-auditor-skill` | Infrastructure security and zero-trust |
| `security-engineer-skill` | DevSecOps and vulnerability management |
| `penetration-tester-skill` | Ethical hacking and security testing |
| `ad-security-reviewer-skill` | Active Directory security analysis |
| `compliance-auditor-skill` | SOC2, HIPAA, GDPR compliance |
| `powershell-security-hardening-skill` | Windows security configuration |
| `risk-manager-skill` | Financial and operational risk assessment |
| `legal-advisor-skill` | Contracts, compliance, and IP guidance |
| `dependency-manager-skill` | Package management and supply chain security |

### Testing & Quality (6)

| Skill | Description |
|-------|-------------|
| `test-automator-skill` | Test framework selection and automation |
| `qa-expert-skill` | Test strategy and quality processes |
| `accessibility-tester-skill` | WCAG 2.1 AA compliance and audits |
| `performance-testing-skill` | Load testing and performance optimization |
| `performance-engineer-skill` | Application optimization and profiling |
| `code-reviewer-skill` | Quality-focused code review with security |

### Debugging & Analysis (5)

| Skill | Description |
|-------|-------------|
| `debugger-skill` | Advanced debugging and root cause analysis |
| `error-detective-skill` | Error pattern analysis across microservices |
| `error-detector-skill` | Error analysis and pattern detection |
| `error-coordinator-skill` | Multi-agent resilience and self-healing |
| `codebase-exploration-skill` | Deep contextual grep for codebases |

### Documentation (6)

| Skill | Description |
|-------|-------------|
| `document-writer-skill` | Technical docs, ADRs, and RFCs |
| `documentation-engineer-skill` | Documentation systems and knowledge sharing |
| `technical-writer-skill` | API docs, user guides, and tutorials |
| `api-documenter-skill` | OpenAPI/Swagger specifications |
| `docx-skill` | Word document automation |
| `pptx-skill` | PowerPoint automation |
| `pdf-skill` | PDF generation and manipulation |
| `xlsx-skill` | Excel workflow automation |

### Windows & PowerShell (7)

| Skill | Description |
|-------|-------------|
| `windows-infra-admin-skill` | AD, DNS/DHCP, and Group Policy |
| `m365-admin-skill` | Microsoft 365 administration and Graph API |
| `powershell-5.1-expert-skill` | Legacy Windows PowerShell with WMI/ADSI |
| `powershell-7-expert-skill` | Cross-platform PowerShell Core |
| `powershell-module-architect-skill` | PowerShell module design and structure |
| `powershell-ui-architect-skill` | PowerShell GUI/TUI with WinForms/WPF |
| `it-ops-orchestrator-skill` | Cross-domain IT task coordination |

### Project Management (7)

| Skill | Description |
|-------|-------------|
| `product-manager-skill` | Product strategy and roadmap development |
| `project-manager-skill` | Traditional and agile project management |
| `scrum-master-skill` | Scrum facilitation and team coaching |
| `business-analyst-skill` | Requirements engineering and BPMN |
| `bmad-bmm-analyst-skill` | BMAD methodology business analysis |
| `bmad-master-skill` | BMAD workflow orchestration |
| `ux-researcher-skill` | User research and behavior analysis |

### Research & Analysis (5)

| Skill | Description |
|-------|-------------|
| `research-analyst-skill` | Multi-source research and synthesis |
| `market-researcher-skill` | Market analysis and consumer insights |
| `competitive-analyst-skill` | Competitor analysis and positioning |
| `search-specialist-skill` | Advanced information retrieval |
| `technical-advisory-skill` | Expert guidance for architecture decisions |

### Communication & Marketing (5)

| Skill | Description |
|-------|-------------|
| `content-marketer-skill` | Content strategy and SEO storytelling |
| `sales-engineer-skill` | Technical demos and PoC validation |
| `customer-success-manager-skill` | Onboarding, adoption, and retention |
| `internal-comms-skill` | Corporate communication and change management |
| `seo-specialist-skill` | Search rankings and organic traffic |

### Agent Orchestration (7)

| Skill | Description |
|-------|-------------|
| `agent-organizer-skill` | Multi-agent system design and orchestration |
| `multi-agent-coordinator-skill` | 100+ agent coordination with scaling |
| `task-distributor-skill` | Load balancing and token economics |
| `performance-monitor-skill` | Agent benchmarking and latency analysis |
| `explore-skill` | Contextual grep for codebase exploration |
| `librarian-skill` | External reference and documentation search |
| `oracle-skill` | Expert reasoning for complex decisions |
| `general-skill` | General-purpose research and task execution |
| `plan-skill` | Strategic planning and task breakdown |

### Developer Tools (5)

| Skill | Description |
|-------|-------------|
| `tooling-engineer-skill` | CLI utilities and IDE extensions |
| `cli-developer-skill` | Command-line and terminal interfaces |
| `git-workflow-manager-skill` | Git branching and collaboration strategies |
| `dx-optimizer-skill` | Developer portals and DORA metrics |
| `skill-creator-skill` | Guide for creating new skills |

### Specialized (6)

| Skill | Description |
|-------|-------------|
| `blockchain-developer-skill` | Smart contracts and DeFi protocols |
| `wordpress-master-skill` | WordPress themes, plugins, and scaling |
| `slack-expert-skill` | Slack app development with Bolt |
| `multimodal-looker-skill` | PDF, image, and diagram analysis |

## Skill File Format

All skills use YAML frontmatter with markdown content:

```yaml
---
name: skill-name
description: Third-person description with trigger phrases. Use when...
---

# Skill Content

Instructions, patterns, and best practices...
```

### Requirements

- **name**: lowercase, hyphens only, max 64 characters
- **description**: third-person voice, max 1024 characters, include trigger phrases
- **content**: under 500 lines (use REFERENCE.md for large skills)

## Progressive Disclosure

Large skills (500+ lines) are split into:

- `SKILL.md` / `skill.yaml` - Core instructions (under 500 lines)
- `REFERENCE.md` - Detailed patterns and templates
- `EXAMPLES.md` - Code examples and use cases

## Anthropic Compliance

This collection follows the [official Anthropic skill specification](https://docs.claude.com/en/docs/claude-code/skills):

- Case-sensitive `SKILL.md` filenames (main branch)
- Clean frontmatter (only `name` and `description`)
- Third-person descriptions with trigger terms
- Progressive disclosure for large skills

## Contributing

1. Fork the repository
2. Create a skill directory: `my-skill-skill/`
3. Add `SKILL.md` with proper frontmatter
4. Follow existing patterns in similar skills
5. Submit a pull request

## License

MIT License - Free to use and modify.

## Resources

- [Anthropic Skills Documentation](https://docs.claude.com/en/docs/claude-code/skills)
- [Anthropic Best Practices](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices)
- [Gemini CLI Skills](https://geminicli.com/docs/cli/skills/)
- [Antigravity Documentation](https://antigravity.google/docs/skills)
