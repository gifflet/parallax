**[AGENT_NAME] AGENT SPECIFICATION**

Brief description of this agent's role and responsibilities.

**ROLE:** [Primary responsibility in one sentence]

**EXAMPLE:** To create a "Security Scanner" agent:
- Replace [AGENT_NAME] with "SECURITY SCANNER"
- Set ROLE to "Scan implementations for security vulnerabilities"
- Define workflow steps for security analysis
- Output security findings in results

**IMPORTS:**
```
- ../lib/state-manager.md       # For state persistence
- ../lib/progress-tracker.md    # For progress updates
- ../protocols/[protocol].md    # Any required protocols
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
  worktree_path: string,
  branch_name: string,
  config: object,
  session_id: string
}
```

**WORKFLOW:**

**STEP 1: [Initial Setup Phase]**
```
1. Validate input context
2. Set up working environment
3. Initialize agent state
4. Report progress: "Starting [agent task]"
```

**STEP 2: [Main Execution Phase]**
```
1. [Primary action 1]
2. [Primary action 2]
3. [Primary action 3]
4. Update progress at each milestone
```

**STEP 3: [Validation Phase]**
```
1. Verify outputs meet requirements
2. Run quality checks
3. Generate metrics/reports
```

**STEP 4: [Completion Phase]**
```
1. Update task status
2. Clean up temporary resources
3. Return results
```

**QUALITY GATES:**
- [ ] Gate 1: [Description of check]
- [ ] Gate 2: [Description of check]
- [ ] Gate 3: [Description of check]

**ERROR HANDLING:**
```
ON error:
    LOG: Error details to session log
    UPDATE: Task status to "failed"
    EXECUTE: Error recovery protocol
    RETURN: Error result with details
END
```

**OUTPUT FORMAT:**
```
{
  status: "success|failed|needs_correction",
  task_id: number,
  agent_id: string,
  duration_minutes: number,
  results: {
    [agent_specific_results]
  },
  metrics: {
    [agent_specific_metrics]
  },
  next_action: "review|correct|finalize|none"
}
```

**PERFORMANCE GUIDELINES:**
- Target completion time: [X] minutes
- Resource usage limits: [CPU/Memory constraints]
- Parallelization opportunities: [What can be done in parallel]

**AGENT-SPECIFIC CONSIDERATIONS:**
- [Special consideration 1]
- [Special consideration 2]
- [Special consideration 3]

**INTEGRATION POINTS:**
- Previous agent: [Which agent hands off to this one]
- Next agent: [Which agent receives output]
- Shared resources: [What resources are shared]

**MONITORING HOOKS:**
- Progress updates every [X] operations
- Status heartbeat every [Y] seconds
- Detailed logging for [specific operations]

**USAGE EXAMPLE:**
```
# After creating your agent file (e.g., security-scanner.md):

1. Import in parallel-dev.md:
   - ./parallel-dev/agents/security-scanner.md

2. Add to orchestration logic:
   IF security_scan_enabled:
       LAUNCH_AGENT:
           type: "security-scanner"
           task_id: task.id
           context: { ... }
       END
   END

3. Use with: parallel-dev --enable-security-scan
```