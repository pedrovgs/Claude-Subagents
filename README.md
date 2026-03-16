# MyClaude

This repository contains my personal collection of Claude Code subagents, commands, skills and dools - specialized AI assitants designed for specific software engineering tasks.

# Claude Code Custom Agents

A collection of custom agents for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that enhance AI-assisted software development workflows.

## Overview

This repository contains 6 specialized agents designed to work with Claude Code CLI:

| Agent | Purpose | Model |
|-------|---------|-------|
| **Pedro** | Principal software engineering tasks | Opus |
| **Milo** | Jira project management | Opus |
| **Natalia** | Automated testing specialist | Opus |
| **Raúl** | Software architecture and design | Opus |
| **Yago** | Pull request descriptions | Opus |
| **Yuta** | Swift WebAssembly expert | Opus |

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/pedrovgs/MyClaude.git
   ```

2. Copy the agent files to your Claude configuration directory:
   ```bash
   mkdir -p ~/.claude/agents
   cp agents/*.md ~/.claude/agents/
   ```

3. Restart Claude Code to load the new agents.

## Agents

### Pedro - Principal Software Engineer

A senior-level engineering agent that prioritizes code quality, architecture, and testing.

**When to use:**
- Architectural decisions
- Feature implementation with proper testing
- Code refactoring to reduce duplication
- Code reviews
- Performance optimization

**Key principles:**
- Readability first with expressive naming
- Small, focused functions with single responsibilities
- Public APIs as orchestrators of private implementations
- Behavioral testing over implementation testing
- Low coupling, high cohesion architecture

**Example:**
```
User: Implement a caching layer for the API responses
Claude: [Invokes Pedro agent for architectural design and implementation]
```

### Milo - Jira Expert

A meticulous Jira management agent that enforces organizational standards for issue creation.

**When to use:**
- Creating epics with all mandatory tasks
- Creating user stories, bugs, and tasks
- Querying and managing Jira issues
- Sprint planning and issue transitions

**Epic creation includes automatic tasks for:**
- Technical implementation planning
- Feature flag configuration
- Design tasks
- QE test cases
- Analytics tracking (Amplitude, DataDog)
- Translation requests
- QA checks
- Release coordination

**Example:**
```
User: Create an epic for the new dark mode feature
Claude: [Invokes Milo agent to create epic with all required tasks]
```

### Natalia - Test Engineer

A testing specialist agent with deep expertise across testing frameworks including vitest, Swift Testing, XCTest, Cypress, and Playwright.

**When to use:**
- Writing unit, integration, end-to-end, or visual regression tests
- Fixing flaky or broken tests
- Designing test strategies and test architecture
- Adding test coverage to new or existing features

**Key principles:**
- Sociable tests over solitary tests — real collaborators over mocks
- Maximize the subject under test — test through public entry points
- Never test implementation details — verify observable behavior only
- Tests are documentation — clear names following Arrange-Act-Assert

**Example:**
```
User: Write tests for the DocumentRenderer class
Claude: [Invokes Natalia agent to write comprehensive behavioral tests]
```

### Raúl - Principal Software Architect

A software architect agent with deep expertise in Clean Architecture, Hexagonal Architecture, Domain-Driven Design, and SOLID principles.

**When to use:**
- Architectural planning and system design decisions
- Implementation design and technical plans
- Refactoring strategies and migration planning
- Structuring code with proper boundaries and contracts

**Key principles:**
- SOLID principles in every design decision
- High cohesion, low coupling as the golden rule
- All concrete implementations behind abstractions
- Dependency injection for testability and flexibility
- No duplication — search for existing components before creating new ones

**Example:**
```
User: How should we structure the new notification system?
Claude: [Invokes Raúl agent to design the architecture with proper boundaries]
```

### Yago - PR Description Generator

A communication-focused agent that analyzes git history and generates comprehensive pull request descriptions.

**When to use:**
- Writing PR descriptions for completed work
- Generating merge request documentation
- Creating structured change documentation

**Features:**
- Analyzes branch history from creation point
- Extracts issue references from commits
- Identifies affected platforms from file changes
- Generates acceptance criteria
- Detects CommonSwift folder changes for cross-platform impact

**Example:**
```
User: Write a PR description for my changes
Claude: [Invokes Yago agent to analyze commits and generate description]
```

### Yuta - Swift WebAssembly Compatibility Agent

A specialized agent for analyzing and refactoring Swift code for WebAssembly compatibility.

### When to use:

- Checking if Swift packages are compatible with WASM
- Identifying incompatible Apple frameworks in code
- Refactoring code with conditional compilation for cross-platform support
- Building and testing Swift projects with the WASM toolchain
- Finding WASM-safe alternatives for platform-specific APIs

### Features:

- Detects incompatible frameworks: UIKit, SwiftUI, CoreGraphics, CoreML, URLSession, Accelerate, AVFoundation, Network
- Analyzes Package.swift for WASM-incompatible dependencies
- Identifies binary targets (xcframeworks) that lack WASM slices
- Suggests `#if canImport()` conditional compilation patterns
- Attempts WASM builds using local Swift WASM toolchain
- Provides compatibility assessment: compatible, partially compatible, or incompatible
- Recommends WASM-safe alternatives (e.g., Matft for Accelerate, ONNX for CoreML)

### Example:

```
User: Can you check if my StringUtils package is compatible with Swift WASM?
Claude: [Invokes Yuta agent to analyze package and attempt WASM build]

User: This code uses Accelerate, can you make it work on WASM?
Claude: [Invokes Yuta agent to refactor with conditional compilation and fallback]

User: I'm getting URLSession errors when building for WASM
Claude: [Invokes Yuta agent to identify networking incompatibilities and suggest alternatives]
```

# Claude Code Custom Commands

* **/fix**: Fixes a bug using a strict Test-Driven Debugging workflow with specialized agents. Accepts a Jira ticket ID/URL or a bug description. Includes branch creation, failing test first, automated verification, and optional PR creation. Supports [agent teams](https://code.claude.com/docs/en/agent-teams) when enabled — spawns Natalia, Pedro, and Yago as teammates with the lead enforcing TDD gates.
* **/implement**: Implements a feature from a Jira ticket or description using a structured multi-agent engineering workflow. Covers architecture planning, implementation, automated testing, code review, quality validation, and coverage verification. Supports [agent teams](https://code.claude.com/docs/en/agent-teams) when enabled — spawns Raúl, Pedro, Natalia, and Yago as teammates with parallel execution of implementation and testing.
* **/tdd**: Implements a feature or Jira ticket using strict Test-Driven Development. Enforces red-green-refactor cycles coordinated by specialized agents (Raúl for architecture, Natalia for tests, Pedro for implementation, Yago for PR). Includes architecture design, strict one-test-at-a-time TDD, refactoring, code review, and optional PR creation. Supports [agent teams](https://code.claude.com/docs/en/agent-teams) when enabled — the lead strictly enforces red→green cycle ordering between Natalia and Pedro teammates.
* **/rebase**: Rebases a list of branches one by one using the list of branches as a cascade.
* **/analyse-code-dependencies**: Generates an HTML report using a package or module as input.
* **/analyse-package-dependencies**: Generates an HTML report using a source file as input.
* **/analyse-tests-quality**: Generates an HTML report using several packages as input after analysing the quality of the automated tests implemented.

## Agent Teams Support

The `/fix`, `/implement`, and `/tdd` commands support [agent teams](https://code.claude.com/docs/en/agent-teams) — an experimental feature that coordinates multiple Claude Code instances working in parallel. When enabled, each named agent (Pedro, Natalia, Raúl, Yago) runs as an independent teammate with its own context window, coordinated by a lead session.

When agent teams are not available, commands fall back to subagents automatically.

| Command | Team Structure | Parallelism |
|---------|---------------|-------------|
| `/fix` | Lead + Natalia → Pedro → Yago | Sequential (TDD gates) |
| `/implement` | Lead + Raúl → Pedro + Natalia → Yago | Pedro and Natalia run in parallel after architecture |
| `/tdd` | Lead + Raúl → Natalia ↔ Pedro → Yago | Strict red→green cycles, lead enforces ordering |

## CLAUDE.md

The `CLAUDE.md` file at the root of this repository contains project-level instructions automatically loaded by Claude Code at the start of every conversation. It defines:

- **Developer info** and basic behavior preferences (concise communication)
- **Subagent defaults** — Pedro is the default coding agent
- **Testing policy** — always propose tests, prefer fast/deterministic, minimize mocking
- **Constraints** — no inventing APIs, no large refactors unless requested, no logging secrets
- **Git policy** — never commit/push without confirmation

You can also place a personal `CLAUDE.md` at `~/.claude/CLAUDE.md` for global instructions across all projects.

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- Claude API access with Opus model support
- For Milo: Jira MCP server configured
- For Yago: Git repository with commit history

## Developed By

* Pedro Vicente Gómez Sánchez - <pedrovicente.gomez@gmail.com>

<a href="https://x.com/pedro_g_s">
  <img alt="Follow me on X" src="https://img.icons8.com/?size=100&id=6Fsj3rv2DCmG&format=png&color=000000" height="60" width="60"/>
</a>
<a href="https://es.linkedin.com/in/pedrovgs">
  <img alt="Add me to Linkedin" src="https://img.icons8.com/?size=100&id=447&format=png&color=000000" height="60" width="60"/>
</a>

## License

Apache V2
