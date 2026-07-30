# Design Patterns Skill

A symptom-driven design pattern reference for AI coding agents. Maps concrete code pressures to the right pattern without over-engineering.

This repository provides a structured reference system that helps AI agents (Claude Code, Cursor, Cline, Roo Code, GitHub Copilot, OpenCode, Zed AI) recognize when a design pattern is justified and implement it correctly.

The skill operates on symptoms rather than pattern names. An agent reads the symptom table, finds the match in the user's code, and recommends the minimal implementation that resolves the pressure. The guiding principle is that a pattern must be justified by a force already present in the codebase, not by one imagined for the future.

## What it contains

| File                                    | Purpose                                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------------------- |
| `SKILL.md`                              | Core decision framework, symptom-to-pattern mapping, language idioms, cost/benefit guidance |
| `references/factory.md`                 | Factory pattern implementation guide                                                        |
| `references/strategy.md`                | Strategy pattern implementation guide                                                       |
| `references/repository.md`              | Repository pattern implementation guide                                                     |
| `references/dependency-injection.md`    | Dependency Injection pattern implementation guide                                           |
| `references/observer.md`                | Observer (Pub/Sub) pattern implementation guide                                             |
| `references/command.md`                 | Command pattern implementation guide                                                        |
| `references/chain-of-responsibility.md` | Chain of Responsibility pattern implementation guide                                        |
| `references/state.md`                   | State pattern implementation guide                                                          |
| `references/builder.md`                 | Builder pattern implementation guide                                                        |
| `references/adapter.md`                 | Adapter pattern implementation guide                                                        |
| `references/facade.md`                  | Facade pattern implementation guide                                                         |
| `references/cqrs.md`                    | CQRS pattern implementation guide                                                           |
| `references/event-sourcing.md`          | Event Sourcing pattern implementation guide                                                 |
| `references/saga.md`                    | Saga pattern implementation guide                                                           |
| `references/outbox.md`                  | Outbox pattern implementation guide                                                         |
| `references/circuit-breaker.md`         | Circuit Breaker pattern implementation guide                                                |
| `references/bulkhead.md`                | Bulkhead pattern implementation guide                                                       |
| `references/producer-consumer.md`       | Producer-Consumer pattern implementation guide                                              |
| `references/clean-architecture.md`      | Clean Architecture implementation guide                                                     |
| `references/hexagonal-architecture.md`  | Hexagonal Architecture (Ports and Adapters) implementation guide                            |
| `README.md`                             | This file                                                                                   |
| `AGENTS.md`                             | Config for agents that read AGENTS.md from the project root                                 |
| `CLAUDE.md`                             | Config for Claude Code and Zed AI agents                                                    |
| `.clinerules`                           | Config for Cline and Roo Code agents                                                        |
| `.cursorrules`                          | Config for Cursor editor agents                                                             |
| `.github/copilot-instructions.md`       | Config for GitHub Copilot                                                                   |
| `.opencode/rules.md`                    | Config for OpenCode project rules                                                           |
| `_opencode-agent.md`                    | Config for OpenCode agent system                                                            |

## Installation

Copy the files from this repository to the agent skill directory and install the relevant config file into your project root.

### Clone to a skill directory

```bash
git clone https://github.com/attaxr/design-patterns.git ~/.agents/skills/design-patterns
```

Then copy the config file for your agent system into your project root.

### Claude Code and Zed AI

Copy `CLAUDE.md` to your project root. The agent loads pattern guidance when it detects relevant triggers.

```bash
cp CLAUDE.md /path/to/your/project/CLAUDE.md
```

### Cline and Roo Code

Cline and Roo Code read `.clinerules` from the project root:

```bash
cp .clinerules /path/to/your/project/.clinerules
```

### Cursor

Cursor reads `.cursorrules` from the project root:

```bash
cp .cursorrules /path/to/your/project/.cursorrules
```

### GitHub Copilot

Copilot reads `.github/copilot-instructions.md`:

```bash
cp .github/copilot-instructions.md /path/to/your/project/.github/copilot-instructions.md
```

### OpenCode

OpenCode supports both project-level rules and agent-level configuration.

**Project rules:**

```bash
cp .opencode/rules.md /path/to/your/project/.opencode/rules.md
```

**Agent-level (applies to all projects):**

```bash
cp _opencode-agent.md ~/.config/opencode/agent/_design-patterns.md
```

Then select the "Design Patterns" agent from the OpenCode agent picker.

### AGENTS.md

For agents that read `AGENTS.md` from the project root:

```bash
cp AGENTS.md /path/to/your/project/AGENTS.md
```

## Prompt to integrate this skill into your project

Copy and paste the following into your AI coding agent to install the design-patterns skill into your codebase. The agent will clone the repository, copy the correct config file (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`, etc.) to your project root, and verify the installation.

```
Install the design-patterns skill from https://github.com/attaxr/design-patterns into this project.

1. Clone the repository to ~/.agents/skills/design-patterns/
2. Read ~/.agents/skills/design-patterns/README.md to understand which config
   file matches my agent system (CLAUDE.md, AGENTS.md, .cursorrules, etc.)
3. Copy the correct config file into this project root
4. Verify the file was placed correctly
```

### Integrating into your own AGENTS.md or CLAUDE.md

To permanently include this skill in your project configuration, add the following section to your existing `AGENTS.md`, `CLAUDE.md`, or equivalent project-level agent instructions file. The agent will then load the skill automatically whenever it detects relevant design pattern pressures.

```markdown
## Design Patterns Skill

This project uses the design-patterns skill from https://github.com/attaxr/design-patterns.
The skill is installed at ~/.agents/skills/design-patterns/.

### When to invoke

Load the design-patterns skill when:

- A switch or if/else on a type, provider, or status keeps growing
- A constructor or function call has many optional parameters
- A vendor SDK shape leaks into domain code
- Logging, retry, or metrics logic is copy-pasted at multiple call sites
- Work must be queued, retried, scheduled, or survive a restart
- A multi-service operation needs distributed rollback
- Reads and writes need different models
- Designing module or service boundaries
- A flaky external dependency causes cascading failures
- The user names a specific design pattern

### Core rule

Pattern follows pressure. A pattern is justified only by a force already present
in the codebase. Do not recommend a pattern based on hypothetical future needs.

### Reference files

Pattern reference files are located at:
~/.agents/skills/design-patterns/references/{pattern-name}.md

Full skill at: ~/.agents/skills/design-patterns/SKILL.md
```

## Usage

### How the skill works

When a design pattern question arises, the agent follows a five-step workflow:

1. **Name the pressure** -- identifies what concretely hurts in the code. Example: "adding a provider means touching four files" instead of "you need a Factory."
2. **Route to the reference file** -- picks the correct reference file and considers at least two pattern candidates.
3. **State the cost** -- names what gets worse: indirection, file count, consistency model changes.
4. **Implement the smallest version** -- one interface, one concrete implementation, no abstract base class.
5. **Prefer language idiom** -- adapts the textbook form to TypeScript, Go, or the target language.

### Trigger conditions

The agent invokes this skill when any of the following appear:

- A `switch` or `if/else` on a type, provider, or status that keeps growing
- A constructor or function with many optional parameters
- A vendor SDK shape leaking into domain code
- Logging, retry, or metrics logic copy-pasted at multiple call sites
- Work that must be queued, retried, scheduled, or survive a restart
- A multi-service operation needing distributed rollback
- Reads and writes with different shapes or scaling needs
- Module or service boundary design
- A flaky external dependency where failures cascade
- Any named pattern mention (Strategy, Factory, Builder, etc.)

### Response format

When recommending a pattern, the agent structures its response as:

```
Pressure:    what specifically hurts today, in the user's code
Pattern:     the name, and the lighter alternative rejected
Cost:        what gets worse (indirection, files, consistency model)
Minimal cut: the smallest version worth writing now
Deferred:    the pattern deliberately not added yet, and its trigger
```

### Core principle

**Pattern follows pressure.** A pattern is justified only by a force already present in the codebase, not by one imagined for the future. Applying Strategy to a single implementation, or Repository over one query, buys indirection and pays nothing back.

### Reference files

Each of the 20 core patterns has a standalone reference file containing:

- The specific pressure that justifies the pattern
- TypeScript and Go implementation examples
- The minimal cut (smallest useful version)
- Common mistakes and how to avoid them
- Cost and tradeoffs
- Related patterns

## Development

### Extending the skill

1. Create or update a reference file in `references/`. Use the existing files as a template: include the pressure, implementation in TS and Go, minimal cut, and common mistakes.
2. Update the symptom-to-pattern table in `SKILL.md` if adding a new pattern.
3. Update any agent config files that reference the pattern list (`CLAUDE.md`, `AGENTS.md`, `.clinerules`, `.cursorrules`, `.opencode/rules.md`, `.github/copilot-instructions.md`).
4. Test by opening a conversation that triggers the relevant symptom.

### File structure conventions

- Each pattern gets its own file in `references/` named after the pattern in kebab-case
- Each reference file follows the same structure: Pressure, Solution, Implementation, Minimal cut, Cost, Related patterns
- Code examples should include TypeScript and Go as the primary languages
- All agent config files live in the repository root, `.github/`, or `.opencode/`

## Contributing

Contributions are welcome. This skill is designed as a community-maintained reference for AI coding agents, and contributions improve the quality of pattern recommendations across all agents.

### What to contribute

- **New patterns** -- If a design pattern is missing and has a clear code pressure, add a reference file.
- **Better examples** -- Improve existing implementation examples with clearer code or additional languages.
- **Additional languages** -- Add implementation guidance for Python, Rust, Java, or other languages to existing reference files.
- **Cost and tradeoff analysis** -- Depth on when a pattern should not be used is as valuable as when it should.
- **Bug fixes** -- Fix errors or clarify misleading guidance in any file.
- **Agent configs** -- Add support for additional AI coding agent systems.

### How to contribute

1. Fork the repository.
2. Create a feature branch.
3. Make your changes following the file structure conventions above.
4. Ensure all reference files follow the same structure (pressure, solution, implementation, minimal cut, cost, related patterns).
5. Submit a pull request with a clear description of the change and its rationale.

### Guidelines

- Each pattern reference must state the concrete pressure that justifies the pattern. Do not add patterns without a clear code symptom.
- Code examples must compile or be syntactically valid. Prefer TypeScript and Go for new examples.
- The "minimal cut" section is required -- it tells the reader the smallest version worth writing.
- The "cost" section is required -- every pattern has tradeoffs. Do not omit them.
- Avoid promotional language. Patterns are tools with costs and benefits, not solutions to every problem.

### Creating a new issue

If you find a problem with the skill content, a missing pattern, or an area where the guidance is misleading or incomplete, open an issue.

Use one of the provided templates:

- **Pattern request** -- suggest a new pattern or improvement to an existing reference file
- **Bug report** -- report incorrect guidance, broken code examples, or factual errors

Open issues at: https://github.com/attaxr/design-patterns/issues/new/choose

Include the file or pattern affected, what the current guidance says, what is incorrect or missing, and a concrete example if applicable.

## License

MIT -- see LICENSE file.
