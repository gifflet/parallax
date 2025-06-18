**PARALLEL DEVELOPMENT COMMAND**

Think deeply about this parallel development orchestration task. You are about to embark on a sophisticated multi-agent development process integrated with Task Master.

**Variables:**

max_agents: $ARGUMENTS
branch_pattern: $ARGUMENTS
review_mode: $ARGUMENTS
task_ids: $ARGUMENTS

**ARGUMENTS PARSING:**
Parse the following arguments from "$ARGUMENTS":
1. `max_agents` - Maximum number of parallel development agents (default: 5)
2. `branch_pattern` - Git branch naming pattern for implementations (default: "feature/task-{id}")
3. `review_mode` - Review strictness level: "strict", "balanced", "lenient" (default: "balanced")
4. `task_ids` - Comma-separated list of specific task IDs to implement (e.g., "2,3,7,15"). If not provided, automatically select available tasks without dependencies

**PHASE 1: TASK MASTER INTEGRATION ANALYSIS**
Establish deep integration with Task Master to identify available work:
- Connect to Task Master and retrieve task information
- If `task_ids` provided: Validate specified task IDs exist and are in pending status
- If `task_ids` NOT provided: Retrieve all pending tasks without dependencies
- Analyze task specifications and complexity requirements
- For specified tasks: Verify dependencies are satisfied before proceeding
- For automatic selection: Identify tasks suitable for parallel development (no blocking dependencies)
- Evaluate task specifications for completeness and clarity
- Prioritize tasks based on complexity, priority, and estimated development time

Think carefully about task selection criteria and how to maximize parallel development efficiency. When specific task IDs are provided, ensure all their dependencies are met before proceeding.

**PHASE 2: DEVELOPMENT ENVIRONMENT RECONNAISSANCE**
Thoroughly analyze the current development environment state:
- Assess current git repository status and available worktree slots
- Identify existing worktrees and their current usage
- Create `.worktrees` directory in project root if it doesn't exist
- Verify git configuration and branch management capabilities
- Analyze codebase structure and shared dependencies
- Evaluate system resources for parallel agent execution

**PHASE 3: AGENT DEPLOYMENT STRATEGY**
Based on Task Master analysis and environment assessment:
- Determine optimal number of parallel agents (respecting max_agents limit)
- Plan git worktree allocation strategy for task isolation within `.worktrees` directory
- Create worktree directory structure:
  ```
  .worktrees/
  ├── task-1/     # Worktree for Task ID 1
  ├── task-3/     # Worktree for Task ID 3
  ├── task-7/     # Worktree for Task ID 7
  └── ...
  ```
- Design agent specialization roles (Developer, Reviewer, Corrector, Finalizer)
- Establish inter-agent communication and coordination protocols
- Create rollback and error recovery mechanisms

**PHASE 4: MULTI-AGENT ORCHESTRATION**
Deploy specialized agents in coordinated parallel execution:

**Agent Distribution Strategy:**
- For 1-3 tasks: Launch all development agents simultaneously
- For 4-8 tasks: Launch in waves of 3-4 agents to manage coordination
- For 9+ tasks: Launch waves of max_agents size with queue management

**Development Agent Coordination Protocol:**
Each Development Agent receives:
1. **Task Context**: Complete task specification from Task Master
2. **Worktree Assignment**: Dedicated git worktree path for isolated development
3. **Implementation Guidelines**: Coding standards and architectural patterns
4. **Quality Requirements**: Definition of done and acceptance criteria
5. **Integration Points**: Dependencies and interface specifications

**Development Agent Task Specification:**
```
TASK: Implement Task #{TASK_ID} in isolated worktree

You are Development Agent #{AGENT_ID} implementing task #{TASK_ID}.

CONTEXT:
- Task specification: {COMPLETE_TASK_SPEC}
- Worktree path: .worktrees/task-{TASK_ID}
- Implementation requirements: {REQUIREMENTS}
- Quality standards: {QUALITY_CRITERIA}

DEVELOPMENT WORKFLOW:
1. Create and switch to dedicated git worktree in .worktrees/task-{TASK_ID}
2. Analyze task specification thoroughly
3. Implement solution following specifications exactly
4. Create comprehensive tests for implementation
5. Document implementation decisions and approach (production docs only)
6. Signal completion for review phase

IMPORTANT: Only implement production code. Do NOT create:
- AI specification documents
- Task planning files
- Agent coordination artifacts
- Any files in .taskmaster directory

DELIVERABLE: Complete implementation ready for review validation (production code only)
```

**Review Agent Coordination:**
```
TASK: Review implementation of Task #{TASK_ID}

You are Review Agent for task #{TASK_ID} implementation.

CONTEXT:
- Original task specification: {TASK_SPEC}
- Implementation location: .worktrees/task-{TASK_ID}
- Review mode: {REVIEW_MODE}
- Quality checklist: {REVIEW_CRITERIA}

REVIEW PROCESS:
1. Compare implementation against original specification
2. Verify all requirements are fully met
3. Check code quality and architectural compliance
4. Validate test coverage and documentation
5. Return either "OK" or detailed correction list

DELIVERABLE: Review result (OK | CORRECTIONS_NEEDED with detailed list)
```

**Correction Agent Specification:**
```
TASK: Apply corrections to Task #{TASK_ID} implementation

You are Correction Agent for task #{TASK_ID}.

CONTEXT:
- Implementation location: .worktrees/task-{TASK_ID}
- Correction requirements: {CORRECTION_LIST}
- Original specification: {TASK_SPEC}
- Quality standards: {QUALITY_CRITERIA}

CORRECTION WORKFLOW:
1. Analyze correction requirements from reviewer
2. Apply necessary changes to meet specifications
3. Verify corrections address all reviewer concerns
4. Re-test implementation after corrections
5. Signal completion for re-review

DELIVERABLE: Corrected implementation addressing all review concerns
```

**Finalization Agent Protocol:**
```
TASK: Finalize Task #{TASK_ID} after successful review

You are Finalization Agent for task #{TASK_ID}.

CONTEXT:
- Approved implementation: .worktrees/task-{TASK_ID}
- Branch pattern: {BRANCH_PATTERN}
- Task ID: {TASK_ID}
- Integration requirements: {INTEGRATION_SPECS}

FINALIZATION WORKFLOW:
1. Create feature branch following naming pattern
2. Attempt merge implementation from .worktrees/task-{TASK_ID} to branch
3. IF merge conflicts occur -> Execute MERGE CONFLICT RESOLUTION PROTOCOL
4. Update Task Master status to completed
5. Clean up worktree resources by removing .worktrees/task-{TASK_ID} directory
6. Generate completion report
7. IMPORTANT: Do NOT commit AI-related specifications or artifacts:
   - Exclude .taskmaster directory
   - Exclude any AI specification documents (*.spec.md, *.ai.md)
   - Exclude task planning artifacts
   - Only commit actual code implementation files

DELIVERABLE: Task marked complete with clean integration (excluding AI artifacts)
```

**MERGE CONFLICT RESOLUTION PROTOCOL:**

When merge conflicts occur during finalization, execute the following interactive resolution process:

**Conflict Detection and Analysis:**
```
TASK: Handle merge conflicts for Task #{TASK_ID}

CONFLICT DETECTION WORKFLOW:
1. Identify all conflicted files and conflict markers
2. Analyze conflict types (content, rename, deletion conflicts)
3. Extract conflict context and competing changes
4. Prepare conflict summary for user presentation

CONFLICT ANALYSIS OUTPUT:
- List of conflicted files with line numbers
- Summary of competing changes for each conflict
- Context of why conflicts occurred
- Suggested resolution strategies
```

**User Interaction Protocol:**
```
CONFLICT PRESENTATION FORMAT:

=== MERGE CONFLICTS DETECTED FOR TASK #{TASK_ID} ===

Branch: {BRANCH_NAME}
Files with conflicts: {CONFLICTED_FILES_COUNT}

CONFLICT DETAILS:
{For each conflicted file:}
📄 File: {FILE_PATH}
🔍 Conflict at lines {LINE_RANGE}

COMPETING CHANGES:
{CURRENT_BRANCH_CONTENT}

CONTEXT: {EXPLANATION_OF_CONFLICT}

RESOLUTION OPTIONS:
1. Accept current branch version (keep HEAD)
2. Accept task implementation version (keep incoming)
3. Manual merge (provide custom resolution)
4. Skip this file for now (mark as unresolved)

What would you like to do for this conflict?
[Please provide your choice and any custom resolution if option 3]
```

**Resolution Execution Protocol:**
```
TASK: Apply user's conflict resolution decisions

RESOLUTION APPLICATION WORKFLOW:
1. Parse user's resolution choice for each conflict
2. Apply resolution strategy:
   - Option 1: Accept current branch (`git checkout --ours {file}`)
   - Option 2: Accept task implementation (`git checkout --theirs {file}`)
   - Option 3: Apply user's custom resolution text
   - Option 4: Mark file as requiring manual attention later

3. For custom resolutions:
   - Replace conflict markers with user-provided content
   - Validate syntax if applicable (compile check for code files)
   - Ensure file integrity after resolution

4. Stage resolved files (`git add {resolved_files}`)
5. Create resolution commit with descriptive message
6. Continue with finalization process

RESOLUTION COMMIT MESSAGE FORMAT:
"resolve: merge conflicts for task #{TASK_ID}

- {FILE1}: {RESOLUTION_STRATEGY}
- {FILE2}: {RESOLUTION_STRATEGY}
- ...

Task: {TASK_TITLE}
Conflicts resolved via user interaction"
```

**Advanced Conflict Handling:**
```
COMPLEX CONFLICT SCENARIOS:

1. MULTIPLE OVERLAPPING CONFLICTS:
   - Present conflicts in logical groupings
   - Allow batch resolution for similar conflicts
   - Provide "apply to all similar" option

2. BINARY FILE CONFLICTS:
   - Present file modification summaries
   - Offer clear choice between versions
   - Handle file renames and moves appropriately

3. DEPENDENCY CONFLICTS:
   - Analyze impact on related files
   - Suggest dependency resolution order
   - Warn about potential breaking changes

4. STRUCTURAL CONFLICTS:
   - Directory structure changes
   - File moves and renames
   - Package/module reorganization

USER DECISION VALIDATION:
- Verify custom resolutions don't break syntax
- Check for logical consistency in resolution choices
- Warn about potential integration issues
- Offer resolution preview before applying
```

**Parallel Execution Management:**
- Create `.worktrees` directory structure for task isolation
- Launch Development Agents for each selected task simultaneously in separate worktrees
- Monitor agent progress and resource utilization across worktree instances
- Coordinate Review Agent deployment upon development completion
- Manage Correction Agent cycles for failed reviews
- Deploy Finalization Agents for successful implementations
- **Execute Merge Conflict Resolution Protocol when conflicts are detected**
- Handle user interaction for conflict resolution decisions
- Apply user-directed resolutions and create appropriate commits
- Clean up worktree instances after successful finalization

**PHASE 5: CONTINUOUS INTEGRATION ORCHESTRATION**
For sustained parallel development, orchestrate continuous task processing:

**Wave-Based Processing:**
1. **Task Queue Management**: Maintain queue of available tasks from Task Master
2. **Agent Pool Management**: Optimize agent allocation based on task complexity
3. **Worktree Lifecycle**: Manage creation, usage, and cleanup of git worktrees
4. **Progress Monitoring**: Track completion rates and quality metrics
5. **Resource Optimization**: Balance parallel load with system capabilities

**Continuous Processing Cycle:**
```
IF task_ids provided:
    1. Validate all specified task IDs and their readiness
    2. Process specified tasks in optimal batches
ELSE:
    WHILE available_tasks > 0 AND system_capacity > threshold:
        1. Query Task Master for pending tasks without dependencies
        2. Select optimal task set for parallel processing
        
THEN:
    3. Allocate worktrees and launch Development Agents
    4. Monitor development progress and completion
    5. Deploy Review Agents for completed implementations
    6. Handle review outcomes (OK -> Finalization, Corrections -> Correction Agent)
    7. Process Correction Agent outputs through re-review cycle
    8. Execute Finalization Agents for approved implementations
    9. **IF merge conflicts detected -> Execute Merge Conflict Resolution Protocol**
    10. **Handle user interaction and apply conflict resolution decisions**
    11. Update Task Master status and clean up resources
    12. Assess system capacity for next wave (if automatic mode)
```

**Quality Assurance Pipeline:**
- **Development Quality**: Code standards, test coverage, documentation
- **Review Quality**: Specification compliance, architectural alignment
- **Integration Quality**: Branch management, merge conflict resolution
- **Process Quality**: Agent coordination, resource management

**Resource Management Strategy:**
- Dynamic worktree allocation and cleanup within `.worktrees` directory
- Structured worktree naming convention: `.worktrees/task-{TASK_ID}`
- Agent lifecycle management with proper resource release
- Memory and processing optimization across parallel streams
- Git repository integrity and branch management
- Automatic cleanup of unused worktree instances

**EXECUTION PRINCIPLES:**

**Task Master Integration:**
- Seamless integration with Task Master for task discovery and status updates
- Respect task dependencies and priority ordering
- Maintain task specification integrity throughout development
- Provide detailed progress reporting and completion metrics

**Version Control Guidelines:**
- NEVER commit AI-related specifications or planning artifacts
- Exclude .taskmaster directory from all commits
- Do not version control task specifications, AI prompts, or planning documents
- Only commit actual implementation code, tests, and production documentation
- Use .gitignore to prevent accidental inclusion of AI artifacts

**Agent Coordination:**
- Clear specialization roles to avoid agent overlap and conflicts
- Structured communication protocols between agent types
- Efficient resource sharing and conflict resolution within isolated worktrees
- Graceful handling of agent failures and task reassignment
- Isolated development environments in `.worktrees/task-{TASK_ID}` directories

**Quality Assurance:**
- Mandatory review process ensuring specification compliance
- Iterative correction cycles until full compliance achieved
- Comprehensive testing and documentation requirements
- Clean integration with main development branch

**Resource Optimization:**
- Intelligent worktree management within `.worktrees` directory preventing resource exhaustion
- Parallel processing balanced with system capabilities
- Efficient agent deployment and lifecycle management
- Clean resource cleanup and state management
- Organized worktree structure for better resource tracking

**ULTRA-THINKING DIRECTIVE:**
Before beginning orchestration, engage in extended thinking about:

**Task Master Strategy:**
- How to efficiently identify and prioritize tasks suitable for parallel development
- When task IDs are specified: How to validate dependencies and ensure readiness
- When automatic: How to select optimal task combinations for parallel execution
- Methods for maintaining task specification integrity across agent handoffs
- Strategies for handling task updates and specification changes during development
- Integration patterns with existing Task Master workflows and tooling

**Multi-Agent Coordination:**
- Optimal agent specialization and role distribution for maximum efficiency
- Communication protocols preventing agent conflicts and resource contention
- Error handling and recovery mechanisms for failed agent executions
- Load balancing strategies for varying task complexity and agent capabilities

**Git Worktree Management:**
- Efficient worktree creation, isolation, and cleanup strategies within `.worktrees` directory
- Structured naming convention: `.worktrees/task-{TASK_ID}` for task isolation
- Branch management and merge conflict prevention across parallel streams
- Repository integrity maintenance with multiple concurrent modifications
- Integration patterns for merging parallel work streams
- Centralized worktree location for better organization and cleanup

**Quality Control Implementation:**
- Review agent design ensuring thorough specification compliance checking
- Correction agent strategies for efficiently addressing review feedback
- Iterative review-correction cycles with convergence guarantees
- Quality metrics and reporting for continuous process improvement

**Resource and Performance Optimization:**
- System resource monitoring and adaptive agent deployment
- Memory and processing optimization across parallel agent execution
- Network and I/O optimization for Task Master integration
- Scalability patterns for handling varying workload sizes

**PULL REQUEST INTEGRATION AFTER CONFLICT RESOLUTION:**

After successful conflict resolution, ensure proper PR updates:

```
PR UPDATE PROTOCOL:
1. Verify all conflicts have been resolved and committed
2. Push resolved changes to the feature branch
3. Update PR description with conflict resolution summary:
   
   CONFLICT RESOLUTION SUMMARY:
   - Files affected: {LIST_OF_FILES}
   - Resolution strategy: {USER_CHOICES_SUMMARY}
   - Additional changes: {ANY_EXTRA_MODIFICATIONS}
   
4. Add appropriate labels (e.g., "conflicts-resolved", "ready-for-review")
5. Request re-review from relevant stakeholders
6. Notify team of successful conflict resolution
```

**CONFLICT RESOLUTION DOCUMENTATION:**
Maintain audit trail of all conflict resolution decisions:
- User choices for each conflict
- Rationale for resolution strategies
- Impact assessment of resolved conflicts
- Integration validation results

Begin execution with deep analysis of these multi-agent coordination challenges and proceed systematically through each phase, leveraging specialized agents for maximum development throughput and quality assurance. 