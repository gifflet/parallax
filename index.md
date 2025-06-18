**PARALLEL DEVELOPMENT ORCHESTRATION**

Execute multiple development tasks concurrently using specialized agents and isolated git worktrees.

**MODULE REFERENCES:**
- Core libs: ./parallel-dev/lib/{argument-parser,state-manager}.md
- Agents: ./parallel-dev/agents/{developer,reviewer,corrector,finalizer}.md
- Protocols: ./parallel-dev/protocols/*.md (load as needed)

**MAIN EXECUTION:**

**Step 1: Parse Arguments**
```
# See ./parallel-dev/lib/argument-parser.md
options = parse_arguments($ARGUMENTS)
IF options.help: show_help() && EXIT
IF options.status: display_status_check() && EXIT
```

**Step 2: Initialize System**
```
# Initialize core components (see ./parallel-dev/lib/state-manager.md)
init_system_components()

IF options.resume:
    session = recover_latest_session()
ELSE:
    tasks = options.task_ids ? 
        validate_task_ids(options.task_ids) : 
        get_available_tasks()
    IF !tasks: ERROR: "No tasks available"
    session = create_session(options, tasks)
END

config = load_configuration(options.profile || "balanced")
```

**Step 3: Initialize Progress**
```
# See ./parallel-dev/lib/progress-tracker.md for display modes
init_progress_display(session, config.display_mode || "compact")
```

**Step 4: Execute Development Cycle**
```
# Setup agent workflow (see ./parallel-dev/lib/event-bus.md)
setup_agent_pipeline(session, {
    workflow: "developer" -> "reviewer" -> "corrector"? -> "finalizer",
    error_handler: "./parallel-dev/protocols/error-recovery.md"
})

# Process tasks in batches
WHILE has_pending_tasks(session):
    batch = get_task_batch(session, options.max_agents)
    
    FOR task IN batch:
        # Launch developer agent for task
        # Agent specs in ./parallel-dev/agents/{type}.md
        launch_agent("developer", task.id, {
            worktree: ".worktrees/task-" + task.id,
            branch: options.branch_pattern.replace("{id}", task.id)
        })
    END
    
    await wait_for_batch_completion(batch)
END
```

**Step 5: Finalize**
```
render_final_report(session)
finalize_session(session)
```

**ERROR HANDLING:**
```
# See ./parallel-dev/lib/error-handler.md for full implementation
ON error:
    handle_error(error, {
        recoverable: -> retry with exponential backoff
        critical: -> checkpoint session and exit
        default: -> log and mark task failed
    })
END
```

**CORE FUNCTIONS:**
```
FUNCTION launch_agent(type, task_id, context):
    # Load agent specification from ./parallel-dev/agents/{type}.md
    # Create isolated worktree and launch with context
    TASK: Execute {type} agent for task {task_id} in {context.worktree}
END

FUNCTION setup_agent_pipeline(session, config):
    # Configure event-driven agent workflow
    # See ./parallel-dev/lib/event-bus.md for event handling
    ON "agent.completed": route_to_next_agent(event)
    ON "agent.error": handle_with_recovery(event)
END

FUNCTION get_task_batch(session, max_size):
    # Get pending tasks respecting dependencies and resource limits
    # See ./parallel-dev/lib/distributed-state.md for details
    RETURN filter_executable_tasks(session.pending_tasks, max_size)
END
```