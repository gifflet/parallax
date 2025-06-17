**PARALLEL DEVELOPMENT ORCHESTRATION**

Execute multiple development tasks concurrently using specialized agents and isolated git worktrees.

**IMPORTS:**
```
- ./parallel-dev/lib/argument-parser.md
- ./parallel-dev/lib/state-manager.md
- ./parallel-dev/lib/progress-tracker.md
- ./parallel-dev/protocols/safe-cleanup.md
- ./parallel-dev/agents/developer.md
- ./parallel-dev/agents/reviewer.md
- ./parallel-dev/agents/corrector.md
- ./parallel-dev/agents/finalizer.md
```

**MAIN EXECUTION:**

**Step 1: Parse Arguments**
```
options = parse_arguments($ARGUMENTS)

IF options.help:
    show_help()
    EXIT
END

IF options.status:
    display_status_check()
    EXIT
END
```

**Step 2: Initialize or Resume Session**
```
IF options.resume:
    session = load_state()
    IF not session:
        ERROR: "No session to resume"
    END
    session = recover_session(session)
ELSE:
    # Get tasks from Task Master
    IF options.task_ids.length > 0:
        tasks = validate_task_ids(options.task_ids)
    ELSE:
        tasks = get_available_tasks()
    END
    
    IF tasks.length == 0:
        ERROR: "No tasks available for development"
    END
    
    session = create_session(options, tasks)
END

# Load configuration
config = load_configuration(options.profile)
```

**Step 3: Initialize Progress Display**
```
display_mode = config.display_mode || "compact"
progress_config = init_progress_display(session, display_mode)
```

**Step 4: Execute Development Cycle**
```
WHILE has_pending_tasks(session):
    # Get next batch of tasks
    batch = get_next_batch(session, options.max_agents)
    
    # Launch development agents concurrently
    FOR task IN batch:
        LAUNCH_AGENT:
            type: "developer"
            task_id: task.id
            context: {
                task_spec: get_task_spec(task.id),
                worktree_path: ".worktrees/task-" + task.id,
                branch_name: options.branch_pattern.replace("{id}", task.id),
                config: config,
                session_id: session.id
            }
        END
    END
    
    # Monitor and coordinate agent execution
    WHILE agents_active(session):
        # Update progress display
        update_progress(progress_config, session)
        
        # Check for completed development
        completed_devs = get_completed_developments(session)
        FOR task IN completed_devs:
            # Launch review agent
            LAUNCH_AGENT:
                type: "reviewer"
                task_id: task.id
                context: {
                    task_spec: get_task_spec(task.id),
                    implementation_path: task.worktree_path,
                    review_mode: options.review_mode,
                    config: config,
                    session_id: session.id
                }
            END
        END
        
        # Check for completed reviews
        reviewed_tasks = get_reviewed_tasks(session)
        FOR task IN reviewed_tasks:
            IF task.review_result == "corrections_needed":
                # Launch corrector agent
                LAUNCH_AGENT:
                    type: "corrector"
                    task_id: task.id
                    context: {
                        corrections: task.corrections,
                        worktree_path: task.worktree_path,
                        config: config,
                        session_id: session.id
                    }
                END
            ELSE:
                # Launch finalizer agent
                LAUNCH_AGENT:
                    type: "finalizer"
                    task_id: task.id
                    context: {
                        worktree_path: task.worktree_path,
                        branch_name: task.branch_name,
                        config: config,
                        session_id: session.id
                    }
                END
            END
        END
        
        # Handle finalized tasks
        finalized_tasks = get_finalized_tasks(session)
        FOR task IN finalized_tasks:
            # Execute safe cleanup
            execute_safe_cleanup(task.id)
            
            # Update Task Master
            update_task_status_in_taskmaster(task.id, "completed")
            
            # Show notification
            notify_task_complete(task)
        END
        
        # Brief pause to prevent CPU spinning
        SLEEP(1000)
    END
END
```

**Step 5: Final Report**
```
# Display execution summary
render_final_report(session)

# Mark session as completed
session.status = "completed"
save_state(session)
```

**ERROR HANDLING:**
```
ON agent_failure:
    task_id = error.task_id
    update_task_status(session.id, task_id, "failed", {
        error: error.message,
        failed_at: iso_timestamp()
    })
    notify_error(get_task(task_id), error.message)
END

ON critical_error:
    # Save current state for recovery
    session.status = "failed"
    session.error = error.message
    save_state(session)
    
    ERROR: "Critical error: " + error.message
    ERROR: "Session saved. Use --resume to continue when resolved."
END
```

**AGENT LAUNCHER:**
```
FUNCTION LAUNCH_AGENT(type, task_id, context):
    agent_module = get_agent_module(type)
    
    # Create agent task with proper context
    agent_prompt = agent_module.generate_prompt(context)
    
    # Launch as concurrent task
    TASK: agent_prompt
    
    # Track agent execution
    track_agent(session.id, task_id, type)
END
```