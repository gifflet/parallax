**ERROR RECOVERY PROTOCOL**

Comprehensive error handling and recovery strategies for parallel development execution.

**PURPOSE:** Gracefully handle errors and provide recovery paths to minimize disruption

**ERROR CATEGORIES:**
```
1. AGENT_ERRORS - Agent execution failures
2. GIT_ERRORS - Version control issues  
3. SYSTEM_ERRORS - Resource/environment problems
4. NETWORK_ERRORS - Connectivity issues
5. DEPENDENCY_ERRORS - Missing or incompatible dependencies
6. STATE_ERRORS - Corrupted or invalid state
```

**ERROR DETECTION:**

```
FUNCTION detect_error_type(error):
    error_patterns = {
        agent: [
            /agent.*failed/i,
            /task.*timeout/i,
            /execution.*error/i
        ],
        git: [
            /fatal:/,
            /error:.*git/i,
            /merge.*conflict/,
            /permission.*denied.*git/
        ],
        system: [
            /ENOSPC/,
            /ENOMEM/,
            /EACCES/,
            /disk.*full/i
        ],
        network: [
            /ECONNREFUSED/,
            /ETIMEDOUT/,
            /getaddrinfo/,
            /fetch.*failed/
        ],
        dependency: [
            /module.*not found/i,
            /cannot find package/,
            /unresolved.*dependency/
        ]
    }
    
    FOR category, patterns IN error_patterns:
        FOR pattern IN patterns:
            IF error.message.match(pattern):
                RETURN category
            END
        END
    END
    
    RETURN "unknown"
END
```

**RECOVERY STRATEGIES:**

**Agent Error Recovery:**
```
FUNCTION recover_agent_error(error, context):
    recovery_plan = {
        actions: [],
        can_retry: true,
        requires_intervention: false
    }
    
    CASE error.subtype:
        "timeout":
            recovery_plan.actions = [
                "Check if agent is still running",
                "Kill zombie processes",
                "Reset task state",
                "Retry with extended timeout"
            ]
            
        "compilation_failure":
            recovery_plan.actions = [
                "Analyze compilation errors",
                "Check for missing dependencies",
                "Verify environment setup",
                "Retry with diagnostic mode"
            ]
            
        "test_failure":
            recovery_plan.actions = [
                "Run tests in isolation",
                "Check test environment",
                "Skip flaky tests temporarily",
                "Continue with warnings"
            ]
            
        "out_of_memory":
            recovery_plan.actions = [
                "Free up memory",
                "Reduce parallel agents",
                "Increase swap space",
                "Retry with lower concurrency"
            ]
    END
    
    RETURN execute_recovery_plan(recovery_plan, context)
END
```

**Git Error Recovery:**
```
FUNCTION recover_git_error(error, context):
    CASE error.subtype:
        "merge_conflict":
            # Delegate to merge conflict protocol
            RETURN execute_merge_conflict_resolution()
            
        "dirty_worktree":
            # Clean up uncommitted changes
            actions = [
                save_uncommitted_changes(),
                reset_worktree(),
                reapply_saved_changes()
            ]
            
        "branch_exists":
            # Handle existing branch
            branch = error.branch_name
            new_branch = branch + "-" + timestamp()
            git checkout -b {new_branch}
            
        "permission_denied":
            # Fix git permissions
            fix_git_permissions(context.worktree_path)
            
        "corrupted_repo":
            # Attempt repair
            git fsck --full
            git gc --aggressive
            IF still_corrupted():
                # Re-clone worktree
                recreate_worktree(context)
            END
    END
END
```

**System Error Recovery:**
```
FUNCTION recover_system_error(error, context):
    CASE error.subtype:
        "disk_full":
            # Free up disk space
            cleanup_actions = [
                clean_temp_files(),
                prune_old_worktrees(),
                clear_build_caches(),
                compress_logs()
            ]
            
            space_freed = execute_cleanup_actions(cleanup_actions)
            IF space_freed < required_space:
                ERROR: "Insufficient disk space after cleanup"
            END
            
        "out_of_memory":
            # Reduce memory usage
            memory_actions = [
                kill_orphan_processes(),
                clear_memory_caches(),
                reduce_agent_count(context),
                enable_swap_if_available()
            ]
            
        "permission_error":
            # Fix permissions
            fix_file_permissions(error.path)
            IF still_fails():
                use_sudo_if_appropriate(error.operation)
            END
    END
END
```

**State Recovery:**
```
FUNCTION recover_corrupted_state(error, context):
    state_file = ".worktrees/.parallel-dev-state.json"
    
    # Try to load backup
    backup_file = state_file + ".backup"
    IF file_exists(backup_file):
        TRY:
            state = load_json(backup_file)
            save_json(state_file, state)
            LOG: "Restored state from backup"
            RETURN true
        CATCH:
            LOG: "Backup also corrupted"
        END
    END
    
    # Reconstruct state from git
    reconstructed = reconstruct_state_from_git()
    IF reconstructed:
        save_json(state_file, reconstructed)
        LOG: "Reconstructed state from git"
        RETURN true
    END
    
    # Start fresh
    LOG: "Creating fresh state"
    create_new_session(context)
END
```

**RETRY LOGIC:**

```
FUNCTION retry_with_backoff(operation, max_attempts = 3):
    attempt = 0
    backoff_ms = 1000  # Start with 1 second
    
    WHILE attempt < max_attempts:
        TRY:
            result = execute_operation(operation)
            RETURN result
        CATCH error:
            attempt += 1
            
            IF attempt >= max_attempts:
                THROW error
            END
            
            LOG: "Attempt {attempt} failed, retrying in {backoff_ms}ms"
            
            # Exponential backoff
            sleep(backoff_ms)
            backoff_ms *= 2
            
            # Prepare for retry
            prepare_retry(operation, error)
        END
    END
END
```

**ERROR AGGREGATION:**

```
FUNCTION aggregate_errors(session):
    error_summary = {
        by_type: {},
        by_task: {},
        total_count: 0,
        critical_count: 0
    }
    
    FOR task_id, task IN session.tasks:
        IF task.errors:
            error_summary.by_task[task_id] = task.errors
            
            FOR error IN task.errors:
                type = detect_error_type(error)
                IF not error_summary.by_type[type]:
                    error_summary.by_type[type] = []
                END
                error_summary.by_type[type].push(error)
                
                error_summary.total_count += 1
                IF error.severity == "critical":
                    error_summary.critical_count += 1
                END
            END
        END
    END
    
    RETURN error_summary
END
```

**RECOVERY CHECKPOINT:**

```
FUNCTION create_recovery_checkpoint(context):
    checkpoint = {
        timestamp: iso_timestamp(),
        session_id: context.session_id,
        task_states: {},
        git_state: capture_git_state(),
        system_state: capture_system_state()
    }
    
    # Save task states
    FOR task_id IN context.active_tasks:
        checkpoint.task_states[task_id] = {
            worktree: get_worktree_state(task_id),
            branch: get_branch_state(task_id),
            progress: get_task_progress(task_id)
        }
    END
    
    # Save checkpoint
    checkpoint_file = ".worktrees/.recovery-checkpoint-" + timestamp() + ".json"
    save_json(checkpoint_file, checkpoint)
    
    RETURN checkpoint_file
END

FUNCTION restore_from_checkpoint(checkpoint_file):
    checkpoint = load_json(checkpoint_file)
    
    # Restore git state
    restore_git_state(checkpoint.git_state)
    
    # Restore task states
    FOR task_id, state IN checkpoint.task_states:
        restore_task_state(task_id, state)
    END
    
    LOG: "Restored from checkpoint: " + checkpoint.timestamp
END
```

**ERROR REPORTING:**

```
FUNCTION generate_error_report(error, context):
    report = """
    ERROR REPORT
    ============
    
    Error Type: {error.type}
    Severity: {error.severity}
    Time: {error.timestamp}
    
    Context:
    - Task ID: {context.task_id}
    - Agent: {context.agent_type}
    - Phase: {context.phase}
    
    Error Details:
    {error.message}
    
    Stack Trace:
    {error.stack}
    
    Recovery Attempted: {error.recovery_attempted}
    Recovery Result: {error.recovery_result}
    
    System State:
    - Memory: {get_memory_usage()}
    - Disk: {get_disk_usage()}
    - CPU: {get_cpu_usage()}
    
    Recommendations:
    {generate_recommendations(error)}
    """
    
    # Save report
    report_file = ".worktrees/error-reports/error-" + timestamp() + ".txt"
    write_file(report_file, report)
    
    RETURN report_file
END
```

**EXPORTS:**
- `detect_error_type(error)` -> Categorize error
- `recover_from_error(error, context)` -> Attempt recovery
- `retry_with_backoff(operation, max_attempts)` -> Retry with exponential backoff
- `create_recovery_checkpoint(context)` -> Save recovery point
- `restore_from_checkpoint(checkpoint_file)` -> Restore from checkpoint
- `aggregate_errors(session)` -> Summarize all errors
- `generate_error_report(error, context)` -> Create detailed error report