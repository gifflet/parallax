# Parallax Architecture

This document explains the technical architecture and design principles behind Parallax's multi-agent orchestration system.

## Overview

Parallax implements a sophisticated multi-agent architecture that enables parallel development through specialized AI agents working in isolated Git worktrees. The system coordinates these agents to implement, review, correct, and finalize multiple tasks simultaneously.

## Core Components

### 1. Orchestration Engine

The main orchestration engine manages the entire parallel development lifecycle:

```
┌─────────────────────────────────────┐
│      Orchestration Engine           │
├─────────────────────────────────────┤
│ • Task Discovery & Analysis         │
│ • Agent Deployment                  │
│ • Resource Management               │
│ • Progress Monitoring               │
│ • Conflict Resolution               │
└─────────────────────────────────────┘
```

### 2. Agent Types

Parallax employs four specialized agent types, each with distinct responsibilities:

#### Development Agent
- **Purpose**: Implement features according to task specifications
- **Workspace**: Isolated Git worktree (`.worktrees/task-{id}`)
- **Responsibilities**:
  - Analyze task requirements
  - Implement solution
  - Write tests
  - Create documentation

#### Review Agent
- **Purpose**: Validate implementations against specifications
- **Inputs**: Original task spec + implementation
- **Outputs**: Approval or detailed correction list
- **Review Criteria**:
  - Specification compliance
  - Code quality
  - Test coverage
  - Architecture alignment

#### Correction Agent
- **Purpose**: Apply review feedback
- **Triggers**: Failed review
- **Process**:
  - Parse correction requirements
  - Modify implementation
  - Re-test changes
  - Signal completion

#### Finalization Agent
- **Purpose**: Merge approved implementations
- **Responsibilities**:
  - Create feature branch
  - Merge implementation
  - Handle conflicts
  - Update task status
  - Clean up resources

## Workflow Architecture

### Parallel Execution Flow

```
Task Master
    │
    ├─→ Task Discovery
    │       │
    │       ├─→ Dependency Analysis
    │       └─→ Task Selection
    │
    ├─→ Worktree Setup
    │       │
    │       └─→ .worktrees/
    │             ├── task-1/
    │             ├── task-2/
    │             └── task-3/
    │
    ├─→ Agent Deployment (Parallel)
    │       │
    │       ├─→ Dev Agent 1 → Review → [Correction] → Finalization
    │       ├─→ Dev Agent 2 → Review → [Correction] → Finalization
    │       └─→ Dev Agent 3 → Review → [Correction] → Finalization
    │
    └─→ Cleanup & Reporting
```

### Agent Communication Protocol

Agents communicate through structured handoffs:

1. **Task Specification Transfer**
   ```
   Orchestrator → Development Agent
   {
     task_id: 123,
     specification: {...},
     worktree_path: ".worktrees/task-123",
     quality_requirements: {...}
   }
   ```

2. **Review Handoff**
   ```
   Development Agent → Review Agent
   {
     task_id: 123,
     implementation_path: ".worktrees/task-123",
     original_spec: {...},
     completion_signal: true
   }
   ```

3. **Correction Request**
   ```
   Review Agent → Correction Agent
   {
     task_id: 123,
     corrections: [
       {file: "src/api.js", issue: "Missing error handling", line: 45},
       {file: "tests/api.test.js", issue: "Incomplete test coverage"}
     ]
   }
   ```

## Git Worktree Management

### Isolation Strategy

Each task operates in a completely isolated Git worktree:

```
project-root/
├── .git/
├── .worktrees/          # Parallax worktree directory
│   ├── task-123/        # Complete project copy for task 123
│   ├── task-124/        # Complete project copy for task 124
│   └── task-125/        # Complete project copy for task 125
├── src/
├── tests/
└── package.json
```

### Benefits of Worktree Isolation

1. **No Conflicts**: Each agent works independently
2. **Parallel Commits**: Agents can commit without blocking
3. **Clean Rollback**: Failed tasks don't affect others
4. **Resource Tracking**: Easy to monitor and clean up

### Worktree Lifecycle

```
Create → Checkout → Develop → Review → Merge → Cleanup
   │         │          │        │       │        │
   └─────────┴──────────┴────────┴───────┴────────┘
              Isolated Environment
```

## Conflict Resolution Architecture

### Interactive Resolution Flow

```
Conflict Detection
    │
    ├─→ Analyze Conflicts
    │     ├── File conflicts
    │     ├── Content conflicts
    │     └── Structural conflicts
    │
    ├─→ Present to User
    │     ├── Visual diff
    │     ├── Context explanation
    │     └── Resolution options
    │
    ├─→ Apply Resolution
    │     ├── User choice
    │     ├── Validation
    │     └── Commit
    │
    └─→ Continue Flow
```

### Conflict Resolution States

1. **Detection**: Identify merge conflicts during finalization
2. **Analysis**: Parse conflict markers and extract context
3. **Presentation**: Format conflicts for user decision
4. **Resolution**: Apply user's choice
5. **Validation**: Ensure resolution maintains code integrity

## Resource Management

### Agent Pool Management

```python
class AgentPool:
    def __init__(self, max_agents):
        self.max_agents = max_agents
        self.active_agents = []
        self.task_queue = []
    
    def deploy_agent(self, task):
        if len(self.active_agents) < self.max_agents:
            agent = create_agent(task)
            self.active_agents.append(agent)
        else:
            self.task_queue.append(task)
```

### Resource Optimization Strategies

1. **Wave-Based Processing**: Deploy agents in controlled waves
2. **Dynamic Scaling**: Adjust based on system resources
3. **Priority Queue**: Process high-priority tasks first
4. **Resource Pooling**: Reuse worktrees when possible

## Quality Assurance Pipeline

### Multi-Stage Validation

```
Implementation → Syntax Check → Unit Tests → Integration Tests → Review
      │              │              │               │              │
      └──────────────┴──────────────┴───────────────┴──────────────┘
                        Quality Gates
```

### Review Mode Architecture

**Strict Mode**:
- Line-by-line specification compliance
- Complete test coverage requirement
- Documentation validation
- Performance benchmarks

**Balanced Mode**:
- Core functionality verification
- Critical path testing
- Basic documentation

**Lenient Mode**:
- Basic functionality check
- Minimal test requirements
- Fast iteration focus

## Integration Points

### Task Master Integration

```
Task Master API
    │
    ├─→ GET /tasks/pending
    ├─→ GET /tasks/{id}/spec
    ├─→ PUT /tasks/{id}/status
    └─→ POST /tasks/{id}/complete
```

### Claude Code Integration

Parallax leverages Claude Code's capabilities:
- File system operations
- Git commands
- Code analysis
- Test execution

## Performance Considerations

### Scalability Patterns

1. **Horizontal Scaling**: Add more agents for linear speedup
2. **Vertical Optimization**: Improve individual agent efficiency
3. **Smart Batching**: Group related tasks
4. **Caching**: Reuse analysis results

### Memory Management

```
Per-Agent Memory = Base + Worktree + Context
Total Memory = Orchestrator + (Agents × Per-Agent Memory)
```

Optimization strategies:
- Lazy worktree creation
- Incremental file loading
- Context pruning
- Resource recycling

## Security Architecture

### Isolation Boundaries

1. **Worktree Isolation**: Each task in separate filesystem
2. **Branch Protection**: No direct main branch access
3. **Review Enforcement**: Mandatory review before merge
4. **Audit Trail**: Complete history of changes

### Safe Execution

- No execution of untrusted code
- Sandboxed test environments
- Limited system access
- Credential isolation

## Extensibility

### Plugin Architecture

Future extensibility through:
- Custom agent types
- Review rule plugins
- Integration adapters
- Workflow templates

### API Design

```typescript
interface ParallaxPlugin {
  name: string;
  version: string;
  agents?: CustomAgent[];
  reviewRules?: ReviewRule[];
  integrations?: Integration[];
}
```

## Summary

Parallax's architecture enables efficient parallel development through:
- Specialized agent roles
- Isolated execution environments
- Intelligent orchestration
- Robust conflict resolution
- Scalable resource management

This design ensures maximum development throughput while maintaining code quality and system stability.