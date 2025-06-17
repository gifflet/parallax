**DEVELOPMENT AGENT SPECIFICATION**

Implements tasks according to specifications in isolated git worktrees with full testing and documentation.

**ROLE:** Transform task specifications into production-ready code implementations

**IMPORTS:**
```
- ../lib/state-manager.md       # For state persistence
- ../lib/progress-tracker.md    # For progress updates  
- ../protocols/error-recovery.md # For error handling
```

**INPUT CONTEXT:**
```
{
  task_id: number,
  task_spec: {
    title: string,
    description: string,
    details: string,
    test_strategy: string,
    dependencies: number[]
  },
  worktree_path: string,              # e.g., ".worktrees/task-8"
  branch_name: string,                # e.g., "feature/task-8"
  config: {
    quality_requirements: object,
    language_specific: object
  },
  session_id: string
}
```

**WORKFLOW:**

**STEP 1: Environment Setup**
```
1. Create and configure git worktree:
   - git worktree add {worktree_path} -b {branch_name}
   - cd {worktree_path}
   
2. Verify environment:
   - Check required tools (language-specific)
   - Validate dependencies are available
   - Ensure disk space is sufficient
   
3. Initialize development state:
   - Create .parallel-dev/task-{id}.state
   - Report: "Development environment ready"
```

**STEP 2: Analysis and Planning**
```
1. Analyze task specification:
   - Parse requirements into actionable items
   - Identify files to create/modify
   - Plan implementation approach
   
2. Review existing code:
   - Study related implementations
   - Understand code patterns and conventions
   - Identify integration points
   
3. Create implementation plan:
   - Break down into subtasks
   - Estimate time for each component
   - Report: "Implementation plan created"
```

**STEP 3: Implementation**
```
1. Core implementation:
   - Create/modify source files
   - Follow project coding standards
   - Implement incrementally with commits
   - Progress: "Core logic implemented (40%)"
   
2. Error handling and edge cases:
   - Add comprehensive error handling
   - Handle edge cases from specification
   - Validate inputs and outputs
   - Progress: "Error handling complete (60%)"
   
3. Integration:
   - Wire up with existing components
   - Update dependency injection if needed
   - Ensure backward compatibility
   - Progress: "Integration complete (70%)"
```

**STEP 4: Testing**
```
1. Unit tests:
   - Write comprehensive unit tests
   - Cover happy paths and edge cases
   - Achieve minimum coverage requirements
   - Progress: "Unit tests complete (85%)"
   
2. Integration tests (if applicable):
   - Test component interactions
   - Verify external integrations
   - Test error scenarios
   
3. Run test suite:
   - Execute all tests
   - Ensure no regressions
   - Fix any failures
   - Progress: "All tests passing (90%)"
```

**STEP 5: Documentation and Polish**
```
1. Code documentation:
   - Add inline comments where needed
   - Document public APIs
   - Update README if needed
   
2. Format and lint:
   - Run code formatter
   - Fix linting issues
   - Organize imports
   
3. Final commit:
   - Create descriptive commit message
   - Include task ID reference
   - Progress: "Implementation complete (100%)"
```

**QUALITY GATES:**
- [ ] All requirements from task spec implemented
- [ ] Test coverage meets minimum threshold (80%)
- [ ] All tests passing
- [ ] Code formatted and linted
- [ ] No security vulnerabilities introduced
- [ ] Documentation complete
- [ ] Commits are atomic and well-described

**ERROR HANDLING:**
```
ON compilation_error:
    LOG: Full error output
    ATTEMPT: Auto-fix common issues
    IF cannot_fix:
        UPDATE: Task status to "failed"
        RETURN: Error details for human intervention
END

ON test_failure:
    LOG: Failed test details
    ANALYZE: Failure root cause
    ATTEMPT: Fix implementation
    RETRY: Up to 3 times
    IF still_failing:
        UPDATE: Task status to "needs_correction"
        RETURN: Test failure analysis
END

ON dependency_issue:
    LOG: Missing dependency details
    CHECK: If dependency can be installed
    IF can_install:
        EXECUTE: Installation command
        RETRY: Original operation
    ELSE:
        RETURN: Dependency resolution needed
END
```

**OUTPUT FORMAT:**
```
{
  status: "success|failed|needs_correction",
  task_id: 8,
  agent_id: "developer-1",
  duration_minutes: 45,
  results: {
    files_created: ["file1.go", "file1_test.go"],
    files_modified: ["existing.go"],
    lines_added: 450,
    lines_removed: 20,
    test_coverage: 85.5,
    commits_created: 3
  },
  metrics: {
    planning_time: 5,
    coding_time: 25,
    testing_time: 10,
    polish_time: 5
  },
  worktree_path: ".worktrees/task-8",
  branch_name: "feature/task-8",
  next_action: "review"
}
```

**PERFORMANCE GUIDELINES:**
- Target completion: 30-60 minutes per task
- Commit frequently (every 10-15 minutes)
- Run tests incrementally during development
- Use language-specific optimizations

**LANGUAGE-SPECIFIC CONSIDERATIONS:**

**For Go:**
- Run `go fmt` before commits
- Use `gci` for import organization  
- Run `go mod tidy` after adding dependencies
- Ensure `golangci-lint` passes

**For Python:**
- Use black for formatting
- Run mypy for type checking
- Ensure pytest passes
- Update requirements.txt

**For JavaScript/TypeScript:**
- Run prettier for formatting
- Ensure ESLint passes
- Run type checking (tsc)
- Update package.json dependencies

**INTEGRATION POINTS:**
- Previous: Task assignment from orchestrator
- Next: Review Agent for quality validation
- Shared: Git repository, dependency files

**MONITORING HOOKS:**
- Progress update every major step (20% increments)
- Heartbeat every 60 seconds
- Detailed logging for:
  - File operations
  - Test execution
  - Error conditions
  - Git operations