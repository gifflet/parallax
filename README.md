# Parallel Development Claude Code Command

A sophisticated multi-agent development orchestration system for concurrent task implementation with Task Master integration.

## Overview

The `parallel-dev` command enables concurrent development of multiple tasks using specialized agents, isolated git worktrees, and automated quality assurance.

## Architecture

```
parallel-dev/
├── agents/                 # Agent specifications
│   ├── _template.md       # Template for new agents
│   ├── developer.md       # Development agent
│   ├── reviewer.md        # Code review agent
│   ├── corrector.md       # Correction agent
│   └── finalizer.md       # Finalization agent
├── protocols/             # Reusable protocols
│   ├── safe-cleanup.md    # Safe resource cleanup
│   ├── merge-conflict.md  # Conflict resolution
│   ├── error-recovery.md  # Error handling
│   └── pr-integration.md  # Pull request workflow
├── config/                # Configuration files
│   ├── defaults.yaml      # Default settings
│   └── profiles/          # Execution profiles
└── lib/                   # Core libraries
    ├── argument-parser.md # CLI argument parsing
    ├── state-manager.md   # Session state management
    └── progress-tracker.md # Visual progress tracking
```

## Quick Start

```bash
# Auto-select available tasks
/parallel-dev

# Develop specific tasks
/parallel-dev 8 9 14

# Use conservative mode
/parallel-dev --max-agents=2 --review=strict

# Check status
/parallel-dev --status

# Resume previous session
/parallel-dev --resume
```

## Command Options

| Option | Short | Description | Default |
|--------|-------|-------------|---------|
| `--max-agents` | `-m` | Maximum parallel agents | 5 |
| `--review` | `-r` | Review mode (strict/balanced/lenient) | balanced |
| `--profile` | `-p` | Load configuration profile | - |
| `--branch` | `-b` | Branch naming pattern | feature/task-{id} |
| `--status` | - | Show current execution status | - |
| `--resume` | - | Resume previous session | - |
| `--dry-run` | - | Preview execution plan | - |
| `--help` | `-h` | Show help message | - |

## Execution Flow

### 1. Task Analysis
- Connects to Task Master
- Identifies available tasks
- Validates dependencies
- Creates execution plan

### 2. Environment Setup
- Creates `.worktrees` directory
- Validates git repository
- Loads configuration
- Initializes progress tracking

### 3. Agent Orchestration
- **Developer Agent**: Implements task in isolated worktree
- **Review Agent**: Validates implementation against specs
- **Corrector Agent**: Fixes issues identified in review
- **Finalizer Agent**: Creates PR and manages integration

### 4. Continuous Integration
- Monitors progress in real-time
- Handles errors gracefully
- Manages resource cleanup
- Persists state for recovery

## Extending the System

### Adding a New Agent
1. Copy `agents/_template.md` to `agents/your-agent.md`
2. Define agent role, inputs, workflow, and outputs
3. Import in main `parallel-dev.md` file
4. Add to agent orchestration logic

### Creating Custom Protocols
1. Create new file in `protocols/` directory
2. Define trigger conditions and workflow
3. Export functions for use by agents
4. Import where needed

### Custom Configuration Profiles
1. Create YAML file in `config/profiles/`
2. Override any default settings
3. Use with `--profile=your-profile`

## Troubleshooting

### Common Issues

**"No session to resume"**
- No previous session found
- Check `.worktrees/.parallel-dev-state.json`

**"Worktree already exists"**
- Previous run didn't clean up properly
- Run: `git worktree prune`
- Check `.worktrees/` directory

**Merge conflicts during finalization**
- Interactive resolution will be triggered
- Or manually: `gh pr checkout <pr-number>`

**Agent timeout**
- Increase timeout in configuration
- Check agent logs for root cause
- Use `--profile=conservative` for longer timeouts

## Configuration

### Default Configuration
Located in `config/defaults.yaml`:
- Execution limits and timeouts
- Quality requirements
- Resource constraints
- Feature toggles

### Profiles
Pre-configured execution profiles for different scenarios:

| Profile | Max Agents | Review Mode | Test Coverage | Use Case |
|---------|-----------|-------------|---------------|----------|
| `conservative` | 2 | strict | 90% | Critical projects |
| `balanced` | 5 | balanced | 80% | General development |
| `aggressive` | 10 | lenient | 60% | Rapid prototyping |
| `ci` | 3 | strict | 85% | CI/CD pipelines |
Pre-configured execution profiles:
- **conservative**: Safe, thorough execution
- **balanced**: Default balanced approach
- **aggressive**: Fast execution, minimal checks
- **ci**: Optimized for CI/CD environments

### Project Configuration
Create `.taskmaster/parallel-dev.config.yaml` for project-specific settings.

## State Management

Session state is persisted in `.worktrees/.parallel-dev-state.json`:
- Active sessions
- Task progress
- Agent assignments
- Execution metrics

## Safety Features

### Safe Cleanup Protocol
- Verifies branch merge status
- Checks for uncommitted changes
- Preserves work-in-progress
- Logs all cleanup decisions

### Error Recovery
- Automatic retry with backoff
- Session state preservation
- Graceful degradation
- Manual intervention support

### Conflict Resolution
- Interactive merge conflict handling
- User-guided resolution
- Automatic commit creation
- PR update integration

## Progress Tracking

Real-time visual progress display:
```
╭─ PARALLEL DEV STATUS ─────────────────────────╮
│ Session: 2024-06-17-001    Uptime: 00:15:32  │
│ Tasks:   [████████░░] 80% (8/10)             │
│ Agents:  [██████████] 3/3 active             │
│                                               │
│ Task 8:  ✅ Completed (PR #12 merged)        │
│ Task 9:  🔄 In Review (agent-2) [12:45]      │
│ Task 14: 🚀 Development (75%) [08:30]        │
╰───────────────────────────────────────────────╯
```

## Extending the System

### Adding New Agents
1. Copy `agents/_template.md`
2. Implement agent specification
3. Update orchestration workflow
4. Test in isolation

### Creating New Protocols
1. Define protocol trigger conditions
2. Implement workflow steps
3. Export reusable functions
4. Update agent imports

### Custom Profiles
1. Create YAML in `config/profiles/`
2. Override default settings
3. Use with `--profile=name`

## Troubleshooting

### Common Issues

**Worktree conflicts**
```bash
git worktree prune
parallel-dev --cleanup
```

**State corruption**
```bash
rm .worktrees/.parallel-dev-state.json
parallel-dev --reset
```

**Resource cleanup**
```bash
parallel-dev --status
parallel-dev --cleanup-preserved
```

### Logs
- Session logs: `.worktrees/.parallel-dev.log`
- Agent logs: `.worktrees/.parallel-dev-agents/`
- Git operations: Standard git log

## Best Practices

1. **Start Small**: Test with 1-2 tasks before scaling
2. **Monitor Progress**: Keep progress display visible
3. **Review PRs**: Don't auto-merge without review
4. **Clean Regularly**: Remove preserved resources
5. **Profile Selection**: Use appropriate profile for context

## Integration with Task Master

The command deeply integrates with Task Master:
- Reads task specifications
- Updates task status
- Respects dependencies
- Reports completion

## Performance Optimization

- Parallel git operations
- Cached dependency analysis
- Lazy task loading
- Compressed state files
- Resource pooling

## Security Considerations

- No force push to protected branches
- Safe cleanup verification
- Isolated worktree execution
- Audit trail logging
- Credential management

## Version History

- **v2.0**: Modular architecture, state management
- **v1.0**: Initial parallel development support

## Contributing

To contribute improvements:
1. Follow modular structure
2. Update relevant documentation
3. Test with various scenarios
4. Ensure backward compatibility