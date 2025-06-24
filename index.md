**PARALLEL DEVELOPMENT COMMAND**

🧠 **ADVANCED AI INFERENCE MODE ACTIVATED**

Think deeply about this parallel development orchestration task. You are about to embark on a sophisticated multi-agent development process integrated with Task Master.

**ADVANCED INFERENCE TECHNIQUES IN USE:**
- **Chain-of-Thought (CoT)**: Step-by-step reasoning with explicit thought processes
- **Self-Consistency**: Multiple reasoning paths validated for consensus
- **Confidence Scoring**: Probabilistic assessment of decisions (0-100%)
- **Tree-of-Thoughts**: Explore multiple solution branches before selection
- **Semantic Analysis**: Deep understanding of task intent and context
- **Meta-Reasoning**: Reflect on reasoning quality and adjust strategies
- **Causal Inference**: Understand cause-effect relationships in code
- **Counterfactual Reasoning**: Consider "what-if" scenarios
- **Bayesian Updates**: Refine beliefs based on new evidence
- **Ensemble Thinking**: Combine multiple inference strategies

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

**🔍 ADVANCED INFERENCE PROTOCOL: Deep Task Analysis**

Establish deep integration with Task Master using multi-modal reasoning:

**Meta-Cognitive Chain-of-Thought Analysis:**
1. Connect to Task Master and retrieve task information
2. **Reasoning Step**: Analyze task relationships and dependencies
3. If `task_ids` provided: Validate specified task IDs exist and are in pending status
   - **Confidence Check**: Verify all dependencies with 95%+ confidence
4. If `task_ids` NOT provided: Retrieve all pending tasks without dependencies
   - **Semantic Analysis**: Understand task intent and requirements
5. **PARALLELIZATION STRATEGY DECISION**: Intelligent mode selection
   - **IF multiple independent tasks available**: Execute **TASK-LEVEL PARALLELIZATION**
     * Each agent handles one complete task from start to finish
     * Maximum efficiency through task-level isolation
   - **IF only one task available**: Execute **SUBTASK-LEVEL PARALLELIZATION**  
     * Break single task into parallel subtasks
     * Multiple agents work on subtasks of the same parent task
   - **Confidence Threshold**: 85%+ confidence in strategy selection
6. Analyze task specifications and complexity requirements
   - **Complexity Scoring**: Rate each task 1-10 for implementation difficulty
7. For specified tasks: Verify dependencies are satisfied before proceeding
   - **Dependency Graph**: Build mental model of task relationships
8. For automatic selection: Identify tasks suitable for parallel development (no blocking dependencies)
   - **Optimization Algorithm**: Select tasks that maximize throughput
9. Evaluate task specifications for completeness and clarity
   - **Clarity Score**: 0-100% specification completeness rating
10. Prioritize tasks based on complexity, priority, and estimated development time
    - **Multi-Factor Ranking**: Weight priority (40%), complexity (30%), time (30%)

**Self-Consistency Validation**: 
- Generate 3+ independent task selection strategies
- Compare selections for consensus (>80% agreement required)
- If divergence detected, apply meta-reasoning to resolve

**Alternative Path Exploration**:
- Use Tree-of-Thoughts to explore task grouping permutations
- Apply counterfactual analysis: "What if we select different tasks?"
- Score each path using multi-criteria decision analysis

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

**Adaptive Agent Distribution Strategy:**

**TASK-LEVEL PARALLELIZATION** (Multiple independent tasks):
- For 1-3 tasks: Launch all development agents simultaneously  
- For 4-8 tasks: Launch in waves of 3-4 agents to manage coordination
- For 9+ tasks: Launch waves of max_agents size with queue management
- Each agent handles complete task lifecycle independently

**SUBTASK-LEVEL PARALLELIZATION** (Single task with subtasks):
- Break parent task into parallel subtasks
- Launch agents equal to number of subtasks (up to max_agents limit)
- Coordinate subtask integration and dependencies
- Ensure subtask agents communicate for consistency

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

🧠 ADVANCED INFERENCE MODE: Chain-of-Thought Development

EXECUTION MODE: {PARALLELIZATION_MODE}
- **TASK-LEVEL MODE**: Complete entire task independently
- **SUBTASK-LEVEL MODE**: Implement specific subtask of parent task

CONTEXT:
- Task specification: {COMPLETE_TASK_SPEC}
- Worktree path: .worktrees/task-{TASK_ID}
- Implementation requirements: {REQUIREMENTS}
- Quality standards: {QUALITY_CRITERIA}
- Confidence Target: 85%+ implementation confidence
- **IF SUBTASK MODE**: Parent task ID: {PARENT_TASK_ID}, Subtask scope: {SUBTASK_DEFINITION}

DEVELOPMENT WORKFLOW WITH REASONING:

1. Create and switch to dedicated git worktree in .worktrees/task-{TASK_ID}
   📊 Causal Reasoning: Isolation → No conflicts → Higher success rate
   🔮 Counterfactual: Without isolation → 65% conflict probability
   
   🗂️ File Tracking Setup:
   - Initialize task file tracking: Create mental list of files to be modified
   - Track file operations: CREATE, MODIFY, DELETE for task-specific files only
   - Maintain staging discipline: Never use `git add .` during development
   
2. Analyze task specification with meta-cognitive awareness
   🔍 Multi-Layer Analysis Protocol:
   - **Semantic Parsing**: Extract requirements with NLP techniques
   - **Importance Weighting**: Bayesian prior on requirement criticality
   - **Edge Case Discovery**: Use adversarial thinking + fuzzing mindset
   - **Dependency Graph**: Build causal model of component interactions
   - **Understanding Confidence**: Self-assessed comprehension (0-100%)
   - **Meta-Check**: "Am I missing something?" reflection loop
   - **IF SUBTASK MODE**: Focus analysis on subtask scope and parent task integration points
   
3. Generate implementation approach using ensemble methods
   🌳 Advanced Tree-of-Thoughts Protocol:
   - **Branch Generation**: Create 5+ diverse implementation strategies
   - **Evaluation Criteria**: 
     * Maintainability Score (weighted 30%)
     * Performance Impact (weighted 25%)
     * Complexity Cost (weighted 20%)
     * Future Extensibility (weighted 25%)
   - **Monte Carlo Selection**: Simulate outcomes for each approach
   - **Decision Justification**: Explicit reasoning chain for selection
   - **Counterfactual Documentation**: "Why not X?" for each rejected path
   - **IF SUBTASK MODE**: Ensure approach aligns with parent task architecture and other subtasks
   
4. Implement solution following specifications exactly
   ✅ Self-Consistency Checks:
   - Does implementation match ALL requirements?
   - Are edge cases handled properly?
   - Is code idiomatic and maintainable?
   - **IF SUBTASK MODE**: Does implementation integrate properly with parent task?
   - Implementation Confidence: ____%
   
5. Create comprehensive tests for implementation
   🧪 Test Strategy Reasoning:
   - Unit tests for isolated logic
   - Integration tests for dependencies
   - Edge case coverage analysis
   - **IF SUBTASK MODE**: Integration tests with other subtasks
   - Test Coverage Target: 80%+
   
6. Document implementation decisions and approach (production docs only)
   📝 Documentation includes:
   - Why this approach was chosen
   - Key design decisions
   - Future maintenance considerations
   
7. Quality Self-Assessment before signaling completion
   🎯 Final Confidence Score: ____%
   - Requirements Met: ___/___
   - Tests Passing: ___/___
   - Code Quality: ___/10
   
8. Prepare selective staging of task files
   📂 Task File Staging Protocol:
   - List all files created, modified, or removed during task implementation
   - Verify each file is directly related to task requirements
   - Use selective `git add <file>` for each task-relevant file
   - AVOID `git add .` to prevent staging unrelated changes
   - Example staging commands:
     ```
     git add src/feature/implementation.js
     git add tests/feature/implementation.test.js
     git add docs/feature/README.md
     ```

IMPORTANT: Only implement production code. Do NOT create:
- AI specification documents
- Task planning files
- Agent coordination artifacts
- Any files in .taskmaster directory

DELIVERABLE: Complete implementation ready for review validation (production code only)
FINAL CONFIDENCE: Report implementation confidence percentage
```

**Review Agent Coordination:**
```
TASK: Review implementation of Task #{TASK_ID}

You are Review Agent for task #{TASK_ID} implementation.

🔍 ADVANCED INFERENCE MODE: Deep Review Analysis

CONTEXT:
- Original task specification: {TASK_SPEC}
- Implementation location: .worktrees/task-{TASK_ID}
- Review mode: {REVIEW_MODE}
- Quality checklist: {REVIEW_CRITERIA}
- Developer Confidence: {DEV_CONFIDENCE}%

REVIEW PROCESS WITH CHAIN-OF-THOUGHT:

1. Specification Compliance Analysis
   📋 Reasoning Steps:
   - List each requirement from spec
   - Check implementation for each requirement
   - Rate compliance: 0-100% per requirement
   - Overall Specification Match: ____%
   
2. Implementation Quality Assessment
   🎯 Multi-Factor Analysis:
   - Code readability and clarity (0-10)
   - Architectural alignment (0-10)
   - Performance considerations (0-10)
   - Security best practices (0-10)
   - Overall Quality Score: ___/40
   
3. Test Coverage Verification
   🧪 Test Analysis Protocol:
   - Unit test coverage: ____%
   - Integration test presence: Yes/No
   - Edge case handling: ____%
   - Test quality assessment: ___/10
   
4. Self-Consistency Validation
   🔄 Cross-Check Protocol:
   - Does implementation solve the stated problem?
   - Are there logical inconsistencies?
   - Would another developer understand this?
   - Consistency Score: ____%
   
5. Generate Review Decision
   🤔 Decision Tree:
   IF all scores >= threshold THEN "OK"
   ELSE generate prioritized correction list:
      - Critical issues (blocks functionality)
      - Major issues (violates requirements)
      - Minor issues (quality improvements)
   
   Review Confidence: ____%

REVIEW METRICS:
- Specification Compliance: ____%
- Code Quality: ___/40
- Test Coverage: ____%
- Overall Confidence: ____%

DELIVERABLE: Review result (OK | CORRECTIONS_NEEDED with detailed list)
INCLUDE: Confidence scores and reasoning for each decision
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

✅ ADVANCED INFERENCE MODE: Intelligent Finalization

CONTEXT:
- Approved implementation: .worktrees/task-{TASK_ID}
- Branch pattern: {BRANCH_PATTERN}
- Task ID: {TASK_ID}
- Integration requirements: {INTEGRATION_SPECS}
- Review Confidence: {REVIEW_CONFIDENCE}%
- Implementation Quality Score: {QUALITY_SCORE}

FINALIZATION WORKFLOW WITH VALIDATION BRANCH:

1. Create validation branch (NOT a worktree) for manual review
   🌿 Advanced Branch Strategy:
   - Branch name: validation/task-{TASK_ID}
   - Purpose: Human validation checkpoint before merge
   - **Source**: Created from task worktree commit (NOT from main/master)
   - Causal Chain: Task worktree commit → Validation branch → Manual review → Quality assurance → Safe merge
   - Repository Benefit: No worktree clutter, cleaner git history, preserves exact task implementation state
   - Bayesian Prior: 95% of validated branches merge successfully
   
2. Prepare implementation for validation
   📦 Pre-validation Steps:
   - Run final lint checks
   - Verify all tests pass
   - Check for uncommitted changes
   - **Validate staging integrity**: Ensure only task-related files are staged
   - **Staging verification protocol**:
     ```
     git status --porcelain  # Review all changes
     git diff --cached       # Review staged changes
     ```
   - **Selective staging execution**:
     ```
     # Stage only task-specific files (NEVER git add .)
     git add path/to/created/file.js
     git add path/to/modified/file.js
     git rm path/to/deleted/file.js  # if applicable
     ```
   - Confidence Check: ____%
   
3. Copy implementation to validation branch with verification
   🔄 Intelligent Transfer Protocol:
   - **Extract commit from task worktree**:
     ```
     git worktree list --porcelain | grep ".worktrees/task-{TASK_ID}" -A 2 | grep "HEAD" | cut -d' ' -f2
     ```
   - **Create validation branch from task commit**:
     ```
     COMMIT_FROM_TASK_WORKTREE=$(git worktree list --porcelain | grep ".worktrees/task-{TASK_ID}" -A 2 | grep "HEAD" | cut -d' ' -f2)
     git branch validation/task-{TASK_ID} $COMMIT_FROM_TASK_WORKTREE
     git checkout validation/task-{TASK_ID}
     ```
   - Selective copy from .worktrees/task-{TASK_ID} (production code only)
   - **Integrity Verification**: Hash comparison pre/post transfer
   - **Semantic Validation**: Ensure code logic preserved
   - **No worktree link**: Maintains repository hygiene
   - **Meta-Check**: "Is this exactly what was implemented?"
   
4. Create validation commit
   💾 Commit Strategy:
   - Clear commit message with task reference
   - Include implementation confidence scores
   - Add review metrics in commit body
   - Push to remote for manual review
   
5. IF merge conflicts detected during branch creation:
   ⚠️ Conflict Analysis:
   - Analyze conflict complexity
   - Prepare conflict summary for user
   - Execute MERGE CONFLICT RESOLUTION PROTOCOL
   - Update validation branch post-resolution
   
6. Notify for manual validation
   📢 Notification Content:
   - Task ID and description
   - Validation branch: validation/task-{TASK_ID}
   - Implementation confidence: ____%
   - Review scores summary
   - "Ready for manual validation and merge"
   
7. Update Task Master status
   📊 Status Update:
   - Mark as "pending_validation"
   - Include validation branch reference
   - Add confidence metrics
   
8. Clean up worktree (after validation confirmation)
   🧹 Cleanup Protocol:
   - Remove .worktrees/task-{TASK_ID} directory
   - Verify no uncommitted changes lost
   - Log cleanup completion
   
9. Generate completion report
   📋 Report includes:
   - Implementation summary
   - Confidence scores throughout process
   - Validation branch location
   - Manual merge instructions

IMPORTANT: Do NOT commit AI-related specifications or artifacts:
- Exclude .taskmaster directory
- Exclude any AI specification documents (*.spec.md, *.ai.md)
- Exclude task planning artifacts
- Only commit actual code implementation files

VALIDATION BRANCH BENEFITS:
- No worktree clutter in repository
- Human validation before final merge
- Easy rollback if issues found
- Clear audit trail of implementations

DELIVERABLE: Task ready for manual validation in branch validation/task-{TASK_ID}
FINAL STATE: Awaiting human validation and manual merge
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

4. Stage resolved files using selective staging (`git add <specific-resolved-file>`)
   - Apply selective staging principles: stage only conflict-resolved files
   - Example: `git add src/conflicted-file.js` (NOT `git add .` )
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

**TASK-LEVEL MODE:**
- Create `.worktrees` directory structure for task isolation
- Launch Development Agents for each selected task simultaneously in separate worktrees
- Each agent works independently on complete task lifecycle
- Minimal inter-agent coordination required

**SUBTASK-LEVEL MODE:**
- Create single `.worktrees/task-{PARENT_ID}` structure with subtask organization
- Launch Development Agents for each subtask in shared parent context
- Coordinate subtask integration and consistency across agents
- Manage subtask dependencies and communication protocols

**COMMON ORCHESTRATION:**
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

**Adaptive Continuous Processing Cycle:**
```
IF task_ids provided:
    1. Validate all specified task IDs and their readiness
    2. DETERMINE PARALLELIZATION MODE:
       - IF multiple independent tasks -> TASK-LEVEL PARALLELIZATION
       - IF single task -> SUBTASK-LEVEL PARALLELIZATION
    3. Process using selected mode
ELSE:
    WHILE available_tasks > 0 AND system_capacity > threshold:
        1. Query Task Master for pending tasks without dependencies
        2. INTELLIGENT MODE SELECTION:
           - Count independent tasks available
           - IF count > 1 -> TASK-LEVEL PARALLELIZATION
           - IF count = 1 -> SUBTASK-LEVEL PARALLELIZATION
        3. Select optimal task/subtask set for parallel processing
        
THEN:
    4. Allocate worktrees according to parallelization mode
    5. Launch Development Agents with appropriate context
    6. Monitor development progress and completion
    7. Deploy Review Agents for completed implementations
    8. Handle review outcomes (OK -> Finalization, Corrections -> Correction Agent)
    9. Process Correction Agent outputs through re-review cycle
    10. Execute Finalization Agents for approved implementations
    11. **IF merge conflicts detected -> Execute Merge Conflict Resolution Protocol**
    12. **Handle user interaction and apply conflict resolution decisions**
    13. Update Task Master status and clean up resources
    14. Assess system capacity for next wave (if automatic mode)
```

**Quality Assurance Pipeline:**
- **Development Quality**: Code standards, test coverage, documentation
- **Review Quality**: Specification compliance, architectural alignment
- **Integration Quality**: Branch management, merge conflict resolution
- **Process Quality**: Agent coordination, resource management

**Adaptive Resource Management Strategy:**

**TASK-LEVEL PARALLELIZATION:**
- Dynamic worktree allocation: `.worktrees/task-{TASK_ID}` per independent task
- Isolated resource management per task
- Independent agent lifecycle management
- Minimal cross-task coordination overhead

**SUBTASK-LEVEL PARALLELIZATION:**
- Shared worktree context: `.worktrees/task-{PARENT_ID}` with subtask organization  
- Coordinated resource sharing among subtask agents
- Managed subtask integration points
- Enhanced inter-agent communication protocols

**COMMON RESOURCE MANAGEMENT:**
- Memory and processing optimization across parallel streams
- Git repository integrity and branch management
- Automatic cleanup of unused worktree instances
- Adaptive resource allocation based on parallelization mode

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

**Git Staging Strategy:**
- NEVER use `git add .` or `git add -A` to stage all files indiscriminately
- ALWAYS stage only files that were created, modified, or removed during the specific task
- Use selective staging: `git add <specific-file-path>` for each relevant file
- Maintain a tracking list of task-related files during development
- Before staging, verify each file is directly related to the task being implemented
- Use `git status` to review changes before staging
- Example selective staging:
  ```
  git add src/components/NewFeature.js
  git add src/tests/NewFeature.test.js
  git add src/utils/helper.js
  # NOT: git add .
  ```

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
- **TASK-LEVEL**: Independent worktree management (`.worktrees/task-{ID}`) preventing resource exhaustion
- **SUBTASK-LEVEL**: Coordinated worktree sharing (`.worktrees/task-{PARENT_ID}`) with subtask organization
- Parallel processing balanced with system capabilities and selected parallelization mode
- Efficient agent deployment and lifecycle management adapted to execution mode
- Clean resource cleanup and state management
- Organized worktree structure optimized for parallelization strategy

**ULTRA-THINKING DIRECTIVE WITH METACOGNITION:**
Before beginning orchestration, engage in deep, reflective thinking using advanced inference:

**Metacognitive Framework:**
- **Level 1**: Think about the task
- **Level 2**: Think about how you're thinking about the task
- **Level 3**: Evaluate the quality of your thinking process
- **Adjustment**: Refine approach based on metacognitive insights

Consider:

**Task Master Strategy with Probabilistic Reasoning:**
- **Task Identification**: Use Bayesian networks to model task suitability
  * P(success|parallel) vs P(success|sequential) comparison
  * Confidence intervals for task complexity estimates
- **Dependency Validation**: Build causal graphs of task relationships
  * Forward inference: "If I do X, what breaks?"
  * Backward inference: "What must be done before Y?"
- **Optimal Selection**: Multi-armed bandit approach for task combinations
  * Exploration vs exploitation tradeoff
  * Thompson sampling for uncertainty handling
- **Specification Integrity**: Semantic hashing + version vectors
  * Detect specification drift using embedding similarity
  * Maintain consistency scores across handoffs
- **Change Management**: Adaptive reasoning for spec updates
  * Impact analysis using counterfactual reasoning
  * Ripple effect prediction with confidence bounds

**Multi-Agent Coordination:**
- Optimal agent specialization and role distribution for maximum efficiency
- Communication protocols preventing agent conflicts and resource contention
- Error handling and recovery mechanisms for failed agent executions
- Load balancing strategies for varying task complexity and agent capabilities

**Adaptive Git Worktree Management:**

**TASK-LEVEL PARALLELIZATION:**
- Independent worktree creation: `.worktrees/task-{TASK_ID}` per task
- Isolated branch management preventing cross-task conflicts
- Simple integration patterns for independent task streams

**SUBTASK-LEVEL PARALLELIZATION:**
- Coordinated worktree management: `.worktrees/task-{PARENT_ID}` with subtask organization
- Shared context with managed subtask integration points
- Complex merge patterns for subtask consolidation

**COMMON WORKTREE PRINCIPLES:**
- Repository integrity maintenance with multiple concurrent modifications
- Centralized worktree location for better organization and cleanup
- Adaptive cleanup strategies based on parallelization mode

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