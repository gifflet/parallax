# Parallax

Multi-agent development orchestration for Claude Code - execute multiple development tasks in parallel with automatic coordination and review.

## What is Parallax?

Parallax is a custom command for [Claude Code](https://github.com/anthropics/claude-code) that enables parallel development across multiple tasks. It orchestrates specialized AI agents to work on different features simultaneously in isolated Git worktrees, with built-in review and merge conflict resolution.

## Prerequisites

- [Claude Code](https://github.com/anthropics/claude-code) installed and configured
- [ccmd](https://github.com/gifflet/ccmd) - Claude Code Command Manager
- Git repository with write access
- Task Master MCP integration (for task management)

## Installation

```bash
ccmd install https://github.com/gifflet/parallax
```

## Usage

Execute Parallax in your project directory:

```bash
/parallax [options]
```

### Options

- **max_agents**: Maximum parallel agents (default: 5)
- **branch_pattern**: Git branch naming (default: "feature/task-{id}")
- **review_mode**: Review strictness - "strict", "balanced", "lenient" (default: "balanced")
- **task_ids**: Specific task IDs to implement (e.g., "2,3,7,15")

### Examples

```bash
# Auto-select available tasks
/parallax

# Work on specific tasks
/parallax task_ids=2,3,7

# Limit parallel agents
/parallax max_agents=3

# Custom branch pattern
/parallax branch_pattern="dev/task-{id}"

# Strict review mode
/parallax review_mode=strict
```

## How It Works

1. **Task Discovery**: Connects to Task Master to identify available tasks
2. **Worktree Setup**: Creates isolated Git worktrees in `.worktrees/` directory
3. **Agent Deployment**: Launches specialized agents for each task:
   - Development Agent: Implements the feature
   - Review Agent: Validates against specifications
   - Correction Agent: Applies review feedback
   - Finalization Agent: Merges and completes task
4. **Conflict Resolution**: Interactive merge conflict handling when needed
5. **Cleanup**: Removes worktrees and updates task status

## Documentation

- [Examples & Use Cases](docs/examples.md) - Detailed usage scenarios
- [Architecture Guide](docs/architecture.md) - Technical design and agent coordination

## Configuration

Parallax respects your project's existing configuration and coding standards. Each agent analyzes the codebase to maintain consistency.

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Submit a pull request

## License

MIT License - See LICENSE file for details