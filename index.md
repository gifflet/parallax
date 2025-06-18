**PARALLEL DEVELOPMENT ORCHESTRATION**

Execute multiple development tasks concurrently using specialized agents and isolated git worktrees.

**IMPORTS:**
```
- ./parallel-dev/lib/argument-parser.md
- ./parallel-dev/lib/state-manager.md
- ./parallel-dev/lib/distributed-state.md
- ./parallel-dev/lib/event-bus.md
- ./parallel-dev/lib/error-handler.md
- ./parallel-dev/lib/performance-optimizer.md
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

**Step 2: Initialize System Components**
```
# Initialize performance-critical components
init_distributed_state()
init_event_bus()
init_performance_monitoring()

# Set up error context
with_error_context({ phase: "initialization" }, () => {
    IF options.resume:
        sessions = get_active_sessions()
        IF sessions.length == 0:
            ERROR: "No session to resume"
        END
        session = sessions[0]  # Most recent active
        session = recover_session(session)
    ELSE:
        # Get tasks with caching
        task_cache = get_cache("task_specs")
        IF options.task_ids.length > 0:
            tasks = await task_cache.get_async("tasks:" + options.task_ids.join(","), 
                () => validate_task_ids(options.task_ids)
            )
        ELSE:
            tasks = await task_cache.get_async("tasks:available",
                () => get_available_tasks()
            )
        END
        
        IF tasks.length == 0:
            ERROR: "No tasks available for development"
        END
        
        session = create_session(options)
        session.task_ids = tasks.map(t => t.id)
        update_session(session.id, session)
    END
})

# Load configuration with hot-reload support
config = load_configuration(options.profile)
subscribe("config.changed", (event) => {
    config = event.data.config
    LOG: "Configuration reloaded"
})
```

**Step 3: Initialize Progress Display**
```
display_mode = config.display_mode || "compact"
progress_config = init_progress_display(session, display_mode)
```

**Step 4: Execute Development Cycle**
```
# Set up event-driven coordination
setup_agent_coordination(session)

# Process tasks with enhanced performance
WHILE has_pending_tasks(session):
    # Get next batch with intelligent scheduling
    batch = await get_optimized_batch(session, options.max_agents)
    
    # Launch agents with resource pooling
    agent_pool = get_pool("process_executors")
    
    FOR task IN batch:
        # Acquire resources from pool
        executor = await agent_pool.acquire()
        
        with_error_context({ task_id: task.id, phase: "development" }, async () => {
            # Launch with circuit breaker protection
            breaker = get_circuit_breaker("agent_launcher")
            await breaker.execute(async () => {
                emit("agent.launching", {
                    type: "developer",
                    task_id: task.id,
                    session_id: session.id
                })
                
                LAUNCH_AGENT:
                    type: "developer"
                    task_id: task.id
                    executor: executor
                    context: {
                        task_spec: await get_cached_task_spec(task.id),
                        worktree_path: ".worktrees/task-" + task.id,
                        branch_name: options.branch_pattern.replace("{id}", task.id),
                        config: config,
                        session_id: session.id,
                        correlation_id: generate_correlation_id()
                    }
                END
            })
        })
    END
    
    # Event-driven agent coordination (no polling)
    await coordinate_agents_async(session)
END

# Event handler setup
FUNCTION setup_agent_coordination(session):
    # Handle agent completion events
    subscribe("agent.completed", async (event) => {
        task_id = event.data.task_id
        agent_type = event.data.agent_type
        
        with_recovery(async () => {
            CASE agent_type:
                "developer":
                    await launch_next_agent(task_id, "reviewer", session)
                "reviewer":
                    IF event.data.result == "corrections_needed":
                        await launch_next_agent(task_id, "corrector", session)
                    ELSE:
                        await launch_next_agent(task_id, "finalizer", session)
                    END
                "corrector":
                    await launch_next_agent(task_id, "reviewer", session)
                "finalizer":
                    await finalize_task(task_id, session)
            END
        }, "exponential_backoff")
    })
    
    # Handle progress updates
    subscribe("agent.progress", (event) => {
        update_task_state(event.data.task_id, {
            progress: event.data.progress,
            current_phase: event.data.phase
        })
        update_progress(progress_config, session)
    })
    
    # Handle errors with automatic recovery
    subscribe("agent.error", async (event) => {
        error = enrich_error(event.data.error)
        log_error(error, { task_id: event.data.task_id })
        
        IF error.recoverable:
            await handle_recoverable_error(event.data.task_id, error)
        ELSE:
            await mark_task_failed(event.data.task_id, error)
        END
    })
END

# Optimized task batching
FUNCTION get_optimized_batch(session, max_size):
    pending_tasks = get_pending_tasks_sorted_by_priority(session)
    available_resources = await get_available_resources()
    
    # Smart batching based on resources and dependencies
    batch = []
    FOR task IN pending_tasks:
        IF batch.length >= max_size:
            BREAK
        END
        
        IF can_execute_task(task, available_resources, batch):
            batch.push(task)
            reserve_resources_for_task(task)
        END
    END
    
    RETURN batch
END

# Async coordination without polling
FUNCTION coordinate_agents_async(session):
    RETURN new Promise((resolve) => {
        # Set up completion handler
        completion_handler = subscribe("session.all_tasks_complete", (event) => {
            IF event.data.session_id == session.id:
                unsubscribe(completion_handler)
                resolve()
            END
        })
        
        # Emit session started event
        emit("session.started", { session_id: session.id })
    })
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
# Enhanced error handling with automatic recovery
ON agent_failure:
    structured_error = enrich_error(error)
    task_id = structured_error.context.task_id
    
    # Log with full context
    log_error(structured_error, {
        session_id: session.id,
        agent_type: structured_error.context.agent_type
    })
    
    # Attempt recovery based on error type
    recovery_result = await attempt_error_recovery(structured_error, task_id)
    
    IF recovery_result.success:
        LOG: "Error recovered successfully"
        emit("error.recovered", {
            task_id: task_id,
            error_id: structured_error.id,
            recovery_strategy: recovery_result.strategy
        })
    ELSE:
        # Update task state with detailed error
        await update_task_state(task_id, {
            status: "failed",
            error: structured_error.to_json(),
            failed_at: iso_timestamp(),
            recovery_attempted: true
        })
        
        notify_error(task_id, structured_error)
    END
END

ON critical_error:
    structured_error = enrich_error(error)
    
    # Create checkpoint before failing
    checkpoint_id = await create_checkpoint(session.id, "critical_error")
    
    # Update session with error details
    await update_session(session.id, {
        status: "failed",
        error: structured_error.to_json(),
        checkpoint_id: checkpoint_id
    })
    
    # Execute emergency cleanup
    await execute_all_rollbacks((r) => r.context.session_id == session.id)
    
    # Generate error report
    report_file = generate_error_report(structured_error, {
        session: session,
        checkpoint: checkpoint_id
    })
    
    ERROR: "Critical error: " + structured_error.message
    ERROR: "Session checkpoint created: " + checkpoint_id
    ERROR: "Error report: " + report_file
    ERROR: "Use --resume to continue from checkpoint when resolved."
END

# Recovery strategies
FUNCTION attempt_error_recovery(error, task_id):
    recovery_strategies = [
        { 
            condition: (e) => e.category == "NETWORK",
            strategy: "exponential_backoff",
            options: { max_retries: 5, base_delay: 2000 }
        },
        {
            condition: (e) => e.category == "RESOURCE" AND e.code == "ENOSPC",
            strategy: "cleanup_and_retry",
            options: { cleanup_fn: () => cleanup_disk_space() }
        },
        {
            condition: (e) => e.category == "GIT" AND e.code == "MERGE_CONFLICT",
            strategy: "automated_resolution",
            options: { resolver: "merge_conflict_protocol" }
        }
    ]
    
    FOR strategy IN recovery_strategies:
        IF strategy.condition(error):
            RETURN await execute_recovery_strategy(strategy, error, task_id)
        END
    END
    
    RETURN { success: false, reason: "No applicable recovery strategy" }
END
```

**AGENT LAUNCHER:**
```
FUNCTION LAUNCH_AGENT(type, task_id, executor, context):
    # Performance measurement
    RETURN measure_performance("agent_launch", async () => {
        agent_module = get_agent_module(type)
        
        # Add monitoring and error handling
        agent_context = {
            ...context,
            event_emitter: create_agent_event_emitter(task_id, type),
            error_handler: create_agent_error_handler(task_id, type),
            performance_tracker: create_agent_performance_tracker(task_id, type)
        }
        
        # Create agent task with enhanced context
        agent_prompt = agent_module.generate_prompt(agent_context)
        
        # Launch with timeout and monitoring
        agent_task = with_timeout(async () => {
            # Execute with rollback capability
            RETURN await with_rollback(
                "agent_" + task_id + "_" + type,
                async () => {
                    TASK: agent_prompt
                },
                async () => {
                    # Rollback on failure
                    await cleanup_agent_resources(task_id, type)
                }
            )
        }, config.agents[type].timeout || 3600000)
        
        # Track agent execution
        track_agent_enhanced(session.id, task_id, type, {
            executor: executor,
            started_at: iso_timestamp(),
            correlation_id: context.correlation_id
        })
        
        # Set up monitoring
        monitor_agent_health(task_id, type, agent_task)
        
        RETURN agent_task
    })
END

# Enhanced agent tracking
FUNCTION track_agent_enhanced(session_id, task_id, type, metadata):
    agent_state = {
        session_id: session_id,
        task_id: task_id,
        type: type,
        status: "running",
        ...metadata
    }
    
    # Update distributed state
    await update_task_state(task_id, {
        current_agent: type,
        agent_metadata: metadata
    })
    
    # Emit tracking event
    emit("agent.started", agent_state)
END

# Agent health monitoring
FUNCTION monitor_agent_health(task_id, type, agent_task):
    heartbeat_interval = 30000  # 30 seconds
    last_heartbeat = Date.now()
    
    monitor = setInterval(async () => {
        # Check if agent is still running
        IF agent_task.completed:
            clearInterval(monitor)
            RETURN
        END
        
        # Check for timeout
        IF Date.now() - last_heartbeat > heartbeat_interval * 3:
            emit("agent.unhealthy", {
                task_id: task_id,
                type: type,
                reason: "No heartbeat"
            })
        END
    }, heartbeat_interval)
    
    # Clean up monitor on completion
    agent_task.finally(() => clearInterval(monitor))
END
```