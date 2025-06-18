**DISTRIBUTED STATE MODULE**

Provides distributed, fault-tolerant state management with task isolation and optimistic locking.

**CORE CONCEPTS:**
- Distributed state with file-based persistence
- Optimistic locking for concurrent access
- State versioning and history
- Checkpoint and recovery support

**STATE STRUCTURE:**
```
STATE_DIRECTORY: .worktrees/.state/
├── sessions/           # Session metadata
├── tasks/             # Task-specific states
│   └── task-{id}/
│       ├── state.json
│       ├── state.lock
│       └── history/
└── checkpoints/       # Recovery points
```

**MAIN FUNCTIONS:**

```
FUNCTION init_distributed_state():
    # Create directory structure
    create_state_directories()
    
    # Migrate legacy state if exists
    IF file_exists(".worktrees/.parallel-dev-state.json"):
        migrate_legacy_state()
    END
    
    # Start state monitor
    start_state_monitor()
END

FUNCTION get_task_state(task_id, options = {}):
    state_path = get_task_state_path(task_id)
    
    # Acquire read lock
    lock = await acquire_lock(state_path + ".lock", "read", options.timeout)
    
    TRY:
        IF not file_exists(state_path):
            RETURN null
        END
        
        state = load_json(state_path)
        
        # Verify integrity
        IF not verify_state_integrity(state):
            state = recover_from_history(task_id)
        END
        
        RETURN state
    FINALLY:
        release_lock(lock)
    END
END

FUNCTION update_task_state(task_id, updates, options = {}):
    state_path = get_task_state_path(task_id)
    
    retry_count = 0
    WHILE retry_count < 3:
        TRY:
            # Acquire write lock
            lock = await acquire_lock(state_path + ".lock", "write")
            
            TRY:
                # Load current state
                current = file_exists(state_path) ? load_json(state_path) : {}
                
                # Check version for optimistic locking
                IF options.expected_version AND current.version != options.expected_version:
                    throw new Error("Version mismatch")
                END
                
                # Apply updates
                new_state = {
                    ...current,
                    ...updates,
                    task_id: task_id,
                    version: (current.version || 0) + 1,
                    updated_at: iso_timestamp()
                }
                
                # Archive current state
                IF current.version:
                    archive_state(task_id, current)
                END
                
                # Write atomically
                write_atomic(state_path, new_state)
                
                emit("state.updated", { task_id, version: new_state.version })
                RETURN new_state
            FINALLY:
                release_lock(lock)
            END
        CATCH error:
            retry_count++
            IF retry_count >= 3:
                throw error
            END
            await sleep(Math.pow(2, retry_count) * 100)
        END
    END
END

FUNCTION acquire_lock(lock_path, mode = "write", timeout = 5000):
    lock_info = {
        id: generate_lock_id(),
        mode: mode,
        owner: process.pid,
        acquired_at: Date.now(),
        expires_at: Date.now() + timeout
    }
    
    start_time = Date.now()
    
    WHILE Date.now() - start_time < timeout:
        IF mode == "read":
            # Multiple read locks allowed
            existing = get_active_locks(lock_path)
            IF not has_write_lock(existing):
                append_lock(lock_path, lock_info)
                RETURN lock_info
            END
        ELSE:
            # Exclusive write lock
            IF not file_exists(lock_path) OR is_lock_expired(lock_path):
                write_atomic(lock_path, [lock_info])
                RETURN lock_info
            END
        END
        
        await sleep(50)
    END
    
    throw new Error("Lock acquisition timeout")
END

FUNCTION create_session(options, tasks):
    session = {
        id: generate_session_id(),
        status: "active",
        started_at: iso_timestamp(),
        options: options,
        task_ids: tasks.map(t => t.id),
        metrics: {
            total_tasks: tasks.length,
            completed: 0,
            in_progress: 0,
            failed: 0
        }
    }
    
    write_atomic(get_session_path(session.id), session)
    emit("session.created", { session_id: session.id })
    
    RETURN session
END

FUNCTION create_checkpoint(session_id, reason = "manual"):
    checkpoint = {
        id: generate_checkpoint_id(),
        session_id: session_id,
        timestamp: iso_timestamp(),
        reason: reason,
        session_state: load_session(session_id),
        task_states: {}
    }
    
    # Capture all task states
    FOR task_id IN checkpoint.session_state.task_ids:
        checkpoint.task_states[task_id] = get_task_state(task_id)
    END
    
    write_atomic(get_checkpoint_path(checkpoint.id), checkpoint)
    emit("checkpoint.created", { checkpoint_id: checkpoint.id })
    
    RETURN checkpoint.id
END

FUNCTION restore_checkpoint(checkpoint_id):
    checkpoint = load_json(get_checkpoint_path(checkpoint_id))
    
    # Restore session
    write_atomic(get_session_path(checkpoint.session_id), checkpoint.session_state)
    
    # Restore task states
    FOR task_id, state IN checkpoint.task_states:
        update_task_state(task_id, state, { force: true })
    END
    
    emit("checkpoint.restored", { checkpoint_id })
    RETURN checkpoint
END
```

**UTILITY FUNCTIONS:**
```
FUNCTION write_atomic(path, data):
    temp_path = path + ".tmp"
    write_json(temp_path, data)
    rename_file(temp_path, path)
END

FUNCTION archive_state(task_id, state):
    history_dir = get_task_history_dir(task_id)
    create_dir_if_not_exists(history_dir)
    
    archive_path = history_dir + "/v" + state.version + ".json"
    write_json(archive_path, state)
    
    # Keep only last 10 versions
    cleanup_history(history_dir, 10)
END

FUNCTION verify_state_integrity(state):
    RETURN state AND 
           state.version !== undefined AND 
           state.updated_at AND 
           is_valid_timestamp(state.updated_at)
END

FUNCTION start_state_monitor():
    # Clean expired locks every 30s
    setInterval(30000, cleanup_expired_locks)
    
    # Verify state integrity every 60s
    setInterval(60000, verify_all_states)
    
    # Auto-checkpoint active sessions every 5m
    setInterval(300000, auto_checkpoint_sessions)
END
```

**EXPORTS:**
- init_distributed_state() -> Initialize system
- get_task_state(task_id, options?) -> Task state
- update_task_state(task_id, updates, options?) -> Updated state
- create_session(options, tasks) -> Session
- update_session(id, updates) -> Updated session
- get_active_sessions() -> Active sessions list
- create_checkpoint(session_id, reason?) -> Checkpoint ID
- restore_checkpoint(checkpoint_id) -> Restored checkpoint
- acquire_lock(path, mode, timeout?) -> Lock info
- release_lock(lock_info) -> Release lock