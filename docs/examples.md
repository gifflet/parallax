# Parallax Examples & Use Cases

This guide provides detailed examples of using Parallax for various development scenarios.

## Basic Usage Scenarios

### 1. Automatic Task Selection

When you run Parallax without specifying tasks, it automatically identifies available tasks without dependencies:

```bash
/parallax
```

**What happens:**
- Queries Task Master for all pending tasks
- Filters out tasks with unmet dependencies
- Selects optimal tasks for parallel execution
- Launches agents based on `max_agents` limit

### 2. Specific Task Implementation

Target specific tasks by their IDs:

```bash
/parallax task_ids=12,15,23
```

**Use case:** When you know exactly which features to implement or have prioritized certain tasks.

### 3. Limited Parallel Execution

Control resource usage by limiting concurrent agents:

```bash
/parallax max_agents=2 task_ids=5,8,10,12
```

**What happens:**
- Processes tasks in waves of 2
- First wave: Tasks 5 and 8
- Second wave: Tasks 10 and 12
- Ensures system stability on resource-constrained environments

## Advanced Scenarios

### 4. Custom Branch Naming

Align with your team's branch naming conventions:

```bash
/parallax branch_pattern="feat/{id}-{date}" task_ids=7
```

**Result:** Creates branch like `feat/7-2024-01-15`

### 5. Strict Review Mode

For critical features requiring thorough validation:

```bash
/parallax review_mode=strict task_ids=45
```

**Review Agent behavior:**
- Checks every requirement meticulously
- Validates edge cases
- Ensures comprehensive test coverage
- May require multiple correction cycles

### 6. Bulk Feature Development

Implement an entire feature set:

```bash
/parallax task_ids=100,101,102,103,104 max_agents=5
```

**Example scenario:** Implementing a complete authentication system where:
- Task 100: User registration
- Task 101: Login functionality
- Task 102: Password reset
- Task 103: Session management
- Task 104: OAuth integration

## Real-World Examples

### Example 1: API Endpoint Development

```bash
# Implement multiple API endpoints in parallel
/parallax task_ids=201,202,203,204
```

Tasks might include:
- Task 201: GET /api/users endpoint
- Task 202: POST /api/users endpoint
- Task 203: PUT /api/users/{id} endpoint
- Task 204: DELETE /api/users/{id} endpoint

Each agent:
1. Creates endpoint implementation
2. Adds request validation
3. Implements tests
4. Updates API documentation

### Example 2: Bug Fix Sprint

```bash
# Fix multiple bugs in parallel with lenient review
/parallax task_ids=301,302,303,304,305 review_mode=lenient max_agents=3
```

Benefits:
- Rapid bug resolution
- Isolated fixes prevent conflicts
- Lenient review for straightforward fixes
- Automatic merge and cleanup

### Example 3: Feature Flag Implementation

```bash
# Add feature flags to multiple components
/parallax branch_pattern="feature-flag/{id}"
```

Parallax will:
1. Identify components needing feature flags
2. Implement flags in isolation
3. Test flag behavior
4. Merge without conflicts

## Workflow Integration

### Continuous Development Flow

```bash
# Morning: Start with high-priority tasks
/parallax task_ids=1,2,3 review_mode=strict

# Afternoon: Handle bug fixes
/parallax task_ids=10,11,12,13,14 review_mode=balanced max_agents=3

# End of day: Quick improvements
/parallax review_mode=lenient
```

### Team Collaboration

```bash
# Developer A works on backend tasks
/parallax task_ids=50,51,52 branch_pattern="backend/{id}"

# Developer B works on frontend tasks
/parallax task_ids=60,61,62 branch_pattern="frontend/{id}"
```

## Handling Merge Conflicts

When conflicts occur, Parallax provides interactive resolution:

```
=== MERGE CONFLICTS DETECTED FOR TASK #15 ===

Branch: feature/task-15
Files with conflicts: 2

📄 File: src/api/users.js
🔍 Conflict at lines 45-52

RESOLUTION OPTIONS:
1. Accept current branch version
2. Accept task implementation
3. Manual merge
4. Skip this file

Your choice: 3
[Provide your custom resolution]
```

## Performance Optimization

### Large Codebase Strategy

```bash
# Process in smaller batches for large codebases
/parallax max_agents=2 task_ids=1,2,3,4,5,6

# Instead of
/parallax max_agents=6 task_ids=1,2,3,4,5,6
```

### Resource-Aware Execution

Monitor system resources and adjust:

```bash
# Light tasks - more parallel execution
/parallax max_agents=8 task_ids=100,101,102,103,104,105,106,107

# Heavy tasks - limit parallelism
/parallax max_agents=2 task_ids=200,201
```

## Best Practices

1. **Start Small**: Begin with 2-3 tasks to understand the workflow
2. **Review Mode Selection**:
   - `strict`: Critical features, security updates
   - `balanced`: Standard features, refactoring
   - `lenient`: Bug fixes, documentation
3. **Task Selection**: Group related tasks for better context
4. **Branch Patterns**: Use descriptive patterns for easy identification
5. **Monitor Progress**: Check `.worktrees/` directory for active development

## Troubleshooting Common Scenarios

### Scenario: Agent Fails During Development

Parallax automatically:
- Logs the failure
- Cleans up the worktree
- Reports which task failed
- Continues with other tasks

### Scenario: Review Keeps Failing

Options:
1. Check task specification clarity
2. Adjust review_mode to be less strict
3. Manually complete the task

### Scenario: System Running Slow

Solutions:
```bash
# Reduce parallel agents
/parallax max_agents=1

# Process fewer tasks
/parallax task_ids=1,2
```

## Integration Examples

### CI/CD Pipeline

```yaml
# .github/workflows/parallax.yml
name: Parallel Development
on:
  schedule:
    - cron: '0 9 * * *'
jobs:
  develop:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run Parallax
        run: |
          /parallax max_agents=3 review_mode=strict
```

### Task Master Integration

Parallax seamlessly integrates with Task Master:
- Reads task specifications
- Updates task status
- Respects dependencies
- Reports completion

## Summary

Parallax transforms how you approach multi-task development by enabling true parallel execution with intelligent coordination. Start with simple scenarios and gradually explore advanced features as you become comfortable with the workflow.