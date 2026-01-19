# Claude-Subagents
This repository contains my personal collection of Claude Code subagents - specialized AI assitants designed for specific software engineering tasks.

# Claude Code Custom Agents

A collection of custom agents for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that enhance AI-assisted software development workflows.

## Overview

This repository contains 3 specialized agents designed to work with Claude Code CLI:

| Agent | Purpose | Model |
|-------|---------|-------|
| **Pedro** | Principal software engineering tasks | Opus |
| **Milo** | Jira project management | Opus |
| **Yago** | Pull request descriptions | Opus |

## Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/pedrovgs/claude-agents.git
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

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- Claude API access with Opus model support
- For Milo: Jira MCP server configured
- For Yago: Git repository with commit history

## License

Apache V2
