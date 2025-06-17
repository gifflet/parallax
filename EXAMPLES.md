# Parallel Development - Usage Examples

## Basic Usage

### Auto-select Available Tasks
```bash
# Automatically select and develop all available tasks
/parallel-dev
```

### Develop Specific Tasks
```bash
# Develop tasks 8, 9, and 14
/parallel-dev 8 9 14

# Using comma-separated format
/parallel-dev 8,9,14
```

### With Options
```bash
# Limit to 3 parallel agents with strict review
/parallel-dev --max-agents=3 --review=strict

# Short form
/parallel-dev -m 3 -r strict 8 9 14
```

## Configuration Profiles

### Conservative Development
Best for critical features or production code:
```bash
/parallel-dev --profile=conservative
```
- Only 2 parallel agents
- Strict code review
- 90% test coverage requirement
- Comprehensive documentation

### Balanced Development (Default)
Standard development workflow:
```bash
/parallel-dev --profile=balanced
# or just
/parallel-dev
```
- 5 parallel agents
- Balanced review (80% coverage)
- Standard quality checks

### Aggressive Development
For rapid prototyping or experimental features:
```bash
/parallel-dev --profile=aggressive
```
- 10 parallel agents
- Lenient review (60% coverage)
- Faster execution, less strict

### CI/CD Integration
For automated environments:
```bash
/parallel-dev --profile=ci
```
- Optimized for CI runners
- Deterministic behavior
- Full logging and reports

## Advanced Usage

### Custom Branch Pattern
```bash
# Use custom branch naming
/parallel-dev --branch="experiment/task-{id}-v2"

# Short form
/parallel-dev -b "hotfix/task-{id}"
```

### Session Management

#### Check Current Status
```bash
/parallel-dev --status
```
Shows:
- Active tasks and their progress
- Agent activity
- Resource usage
- Time estimates

#### Resume Interrupted Session
```bash
# If execution was interrupted
/parallel-dev --resume
```
Automatically:
- Recovers session state
- Continues from last checkpoint
- Preserves completed work

### Dry Run Mode
Preview what would be executed:
```bash
/parallel-dev --dry-run 8 9 14
```

## Common Workflows

### 1. Feature Development Sprint
Develop multiple related features:
```bash
# Get all UI-related tasks (assuming they're tagged)
/parallel-dev --tag=ui --profile=balanced
```

### 2. Bug Fix Batch
Quick fixes with minimal overhead:
```bash
# Fix specific bugs with fast turnaround
/parallel-dev --profile=aggressive --branch="bugfix/task-{id}" 23 24 25
```

### 3. Pre-Release Polish
High-quality implementation for release:
```bash
# Maximum quality checks
/parallel-dev --profile=conservative --review=strict
```

### 4. Experimentation
Try out ideas quickly:
```bash
# Rapid prototyping mode
/parallel-dev --profile=aggressive --max-agents=10 --review=lenient
```

## Handling Common Scenarios

### Merge Conflicts
When conflicts occur during finalization:
1. Interactive prompt appears
2. Choose resolution strategy
3. Automatic retry after resolution

### Failed Tasks
```bash
# Check what failed
/parallel-dev --status

# Resume with fixes
/parallel-dev --resume
```

### Resource Constraints
```bash
# Reduce parallel execution
/parallel-dev --max-agents=2

# Use conservative profile
/parallel-dev --profile=conservative
```

## Integration Examples

### With Task Master
```bash
# Auto-select high-priority tasks
/parallel-dev --priority=high

# Complete specific milestone
/parallel-dev --milestone=v1.0
```

### With GitHub
```bash
# All PRs created automatically
# Enable auto-merge (if using appropriate profile)
/parallel-dev --auto-merge
```

## Monitoring and Debugging

### Verbose Output
```bash
# Detailed progress tracking
/parallel-dev --verbose

# Debug mode for troubleshooting
/parallel-dev --debug
```

### Log Analysis
```bash
# View logs for specific task
cat .worktrees/task-8/agent.log

# Check state file
cat .worktrees/.parallel-dev-state.json | jq
```

## Best Practices

1. **Start Conservative**: Use conservative profile for first run
2. **Monitor Progress**: Use `--status` to track execution
3. **Handle Failures**: Use `--resume` instead of starting over
4. **Clean Resources**: Let automatic cleanup handle worktrees
5. **Review PRs**: Even with auto-merge, review generated PRs

## Troubleshooting

### "No tasks available"
- Check Task Master connection
- Verify task dependencies
- Use specific task IDs

### "Agent timeout"
- Use conservative profile
- Reduce parallel agents
- Check system resources

### "Merge conflicts"
- Interactive resolution provided
- Can manually resolve with `gh pr checkout`

### "State corrupted"
- Delete `.worktrees/.parallel-dev-state.json`
- Start fresh (completed work is preserved)