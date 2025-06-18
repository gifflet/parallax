**DISTRIBUTED STATE MANAGEMENT MODULE**

Provides distributed, fault-tolerant state management with task isolation and optimistic locking.

**PURPOSE:** Replace monolithic state file with distributed task-specific states for better performance and reliability

**IMPORTS:**
```
- ./event-bus.md    # For state change notifications
```

**STATE STRUCTURE:**
```
STATE_DIRECTORY: .worktrees/.state/
├── sessions/
│   ├── 2024-06-17-001.json     # Session metadata
│   └── 2024-06-17-002.json
├── tasks/
│   ├── task-8/
│   │   ├── state.json           # Task state
│   │   ├── state.lock           # Lock file
│   │   └── history/             # State history
│   │       ├── v1.json
│   │       └── v2.json
│   └── task-9/
│       └── ...
└── checkpoints/
    └── checkpoint-2024-06-17-100523.json
```

**CORE IMPLEMENTATION:**

```
# State paths configuration
STATE_CONFIG = {
    root: ".worktrees/.state",
    sessions: ".worktrees/.state/sessions",
    tasks: ".worktrees/.state/tasks",
    checkpoints: ".worktrees/.state/checkpoints",
    max_history: 10,
    lock_timeout: 5000,
    retry_attempts: 3
}

FUNCTION init_distributed_state():
    # Create directory structure
    FOR path IN [STATE_CONFIG.root, STATE_CONFIG.sessions, STATE_CONFIG.tasks, STATE_CONFIG.checkpoints]:
        CREATE_DIR_IF_NOT_EXISTS(path)
    END
    
    # Migrate from old state if exists
    migrate_legacy_state()
    
    # Start state monitor
    start_state_monitor()
END

FUNCTION get_task_state(task_id, options = {}):
    task_dir = STATE_CONFIG.tasks + "/task-" + task_id
    state_file = task_dir + "/state.json"
    lock_file = task_dir + "/state.lock"
    
    # Acquire read lock
    lock = acquire_lock(lock_file, "read", options.timeout)
    
    TRY:
        IF not file_exists(state_file):
            RETURN null
        END
        
        state = load_json_with_validation(state_file)
        
        # Verify integrity
        IF not verify_state_integrity(state):
            state = recover_task_state(task_id)
        END
        
        RETURN state
    FINALLY:
        release_lock(lock)
    END
END

FUNCTION update_task_state(task_id, updates, options = {}):
    task_dir = STATE_CONFIG.tasks + "/task-" + task_id
    state_file = task_dir + "/state.json"
    lock_file = task_dir + "/state.lock"
    
    CREATE_DIR_IF_NOT_EXISTS(task_dir)
    
    retry_count = 0
    WHILE retry_count < STATE_CONFIG.retry_attempts:
        TRY:
            # Acquire write lock
            lock = acquire_lock(lock_file, "write", options.timeout)
            
            TRY:
                # Load current state
                current_state = {}
                IF file_exists(state_file):
                    current_state = load_json_with_validation(state_file)
                END
                
                # Check version for optimistic locking
                IF options.expected_version AND current_state.version != options.expected_version:
                    THROW new Error("Version mismatch - state was modified")
                END
                
                # Apply updates
                new_state = {
                    ...current_state,
                    ...updates,
                    task_id: task_id,
                    version: (current_state.version || 0) + 1,
                    updated_at: iso_timestamp()
                }
                
                # Archive current state
                IF current_state.version:
                    archive_state(task_id, current_state)
                END
                
                # Write atomically
                write_atomic(state_file, new_state)
                
                # Emit state change event
                emit("state.task_updated", {
                    task_id: task_id,
                    previous_version: current_state.version || 0,
                    new_version: new_state.version,
                    changes: calculate_diff(current_state, new_state)
                })
                
                RETURN new_state
            FINALLY:
                release_lock(lock)
            END
        CATCH error:
            retry_count += 1
            IF retry_count >= STATE_CONFIG.retry_attempts:
                THROW error
            END
            
            # Exponential backoff
            SLEEP(Math.pow(2, retry_count) * 100)
        END
    END
END
```

**LOCKING MECHANISM:**

```
FUNCTION acquire_lock(lock_file, mode = "write", timeout = 5000):
    lock_info = {
        id: generate_lock_id(),
        mode: mode,
        owner: get_process_id(),
        acquired_at: iso_timestamp(),
        expires_at: iso_timestamp(Date.now() + timeout)
    }
    
    start_time = Date.now()
    
    WHILE Date.now() - start_time < timeout:
        TRY:
            IF mode == "read":
                # Multiple read locks allowed
                existing_locks = get_active_locks(lock_file)
                IF not has_write_lock(existing_locks):
                    append_lock(lock_file, lock_info)
                    RETURN lock_info
                END
            ELSE:
                # Exclusive write lock
                IF not file_exists(lock_file) OR is_lock_expired(lock_file):
                    write_lock_atomic(lock_file, [lock_info])
                    RETURN lock_info
                END
            END
        CATCH:
            # Lock contention - retry
        END
        
        SLEEP(50)  # 50ms retry interval
    END
    
    THROW new Error("Failed to acquire lock within timeout")
END

FUNCTION release_lock(lock_info):
    IF not lock_info:
        RETURN
    END
    
    lock_file = lock_info.file
    
    IF lock_info.mode == "read":
        # Remove specific read lock
        locks = get_active_locks(lock_file)
        locks = locks.filter(l => l.id != lock_info.id)
        IF locks.length > 0:
            write_lock_atomic(lock_file, locks)
        ELSE:
            remove_file(lock_file)
        END
    ELSE:
        # Remove write lock
        remove_file(lock_file)
    END
END

FUNCTION is_lock_expired(lock_file):
    locks = get_active_locks(lock_file)
    current_time = Date.now()
    
    RETURN locks.every(lock => {
        expire_time = Date.parse(lock.expires_at)
        RETURN current_time > expire_time
    })
END
```

**STATE HISTORY AND RECOVERY:**

```
FUNCTION archive_state(task_id, state):
    history_dir = STATE_CONFIG.tasks + "/task-" + task_id + "/history"
    CREATE_DIR_IF_NOT_EXISTS(history_dir)
    
    archive_file = history_dir + "/v" + state.version + ".json"
    write_json(archive_file, state)
    
    # Cleanup old history
    cleanup_state_history(task_id)
END

FUNCTION cleanup_state_history(task_id):
    history_dir = STATE_CONFIG.tasks + "/task-" + task_id + "/history"
    
    IF not dir_exists(history_dir):
        RETURN
    END
    
    files = list_files(history_dir)
        .sort_by(f => extract_version(f))
        .reverse()
    
    # Keep only max_history versions
    IF files.length > STATE_CONFIG.max_history:
        FOR file IN files.slice(STATE_CONFIG.max_history):
            remove_file(history_dir + "/" + file)
        END
    END
END

FUNCTION recover_task_state(task_id):
    history_dir = STATE_CONFIG.tasks + "/task-" + task_id + "/history"
    
    IF not dir_exists(history_dir):
        RETURN null
    END
    
    # Find latest valid history
    files = list_files(history_dir)
        .sort_by(f => extract_version(f))
        .reverse()
    
    FOR file IN files:
        TRY:
            state = load_json_with_validation(history_dir + "/" + file)
            IF verify_state_integrity(state):
                LOG_INFO: "Recovered task state from history", {
                    task_id: task_id,
                    version: state.version
                }
                
                # Restore as current state
                state_file = STATE_CONFIG.tasks + "/task-" + task_id + "/state.json"
                write_atomic(state_file, state)
                
                RETURN state
            END
        CATCH:
            CONTINUE
        END
    END
    
    LOG_ERROR: "Failed to recover task state", { task_id: task_id }
    RETURN null
END
```

**SESSION MANAGEMENT:**

```
FUNCTION create_session(options):
    session = {
        id: generate_session_id(),
        status: "active",
        started_at: iso_timestamp(),
        updated_at: iso_timestamp(),
        options: options,
        task_ids: [],
        metrics: {
            total_tasks: 0,
            completed: 0,
            in_progress: 0,
            failed: 0
        }
    }
    
    session_file = STATE_CONFIG.sessions + "/" + session.id + ".json"
    write_atomic(session_file, session)
    
    emit("state.session_created", { session_id: session.id })
    
    RETURN session
END

FUNCTION update_session(session_id, updates):
    session_file = STATE_CONFIG.sessions + "/" + session_id + ".json"
    
    IF not file_exists(session_file):
        THROW new Error("Session not found: " + session_id)
    END
    
    session = load_json_with_validation(session_file)
    updated_session = {
        ...session,
        ...updates,
        updated_at: iso_timestamp()
    }
    
    write_atomic(session_file, updated_session)
    
    emit("state.session_updated", {
        session_id: session_id,
        changes: calculate_diff(session, updated_session)
    })
    
    RETURN updated_session
END

FUNCTION get_active_sessions():
    session_files = list_files(STATE_CONFIG.sessions)
    active_sessions = []
    
    FOR file IN session_files:
        session = load_json_with_validation(STATE_CONFIG.sessions + "/" + file)
        IF session.status == "active":
            active_sessions.push(session)
        END
    END
    
    RETURN active_sessions.sort_by(s => s.updated_at).reverse()
END
```

**CHECKPOINTING:**

```
FUNCTION create_checkpoint(session_id, reason = "manual"):
    checkpoint = {
        id: generate_checkpoint_id(),
        session_id: session_id,
        timestamp: iso_timestamp(),
        reason: reason,
        session_state: null,
        task_states: {}
    }
    
    # Get session state
    session_file = STATE_CONFIG.sessions + "/" + session_id + ".json"
    IF file_exists(session_file):
        checkpoint.session_state = load_json_with_validation(session_file)
    END
    
    # Get all task states for session
    IF checkpoint.session_state?.task_ids:
        FOR task_id IN checkpoint.session_state.task_ids:
            task_state = get_task_state(task_id)
            IF task_state:
                checkpoint.task_states[task_id] = task_state
            END
        END
    END
    
    # Save checkpoint
    checkpoint_file = STATE_CONFIG.checkpoints + "/checkpoint-" + checkpoint.id + ".json"
    write_atomic(checkpoint_file, checkpoint)
    
    emit("state.checkpoint_created", {
        checkpoint_id: checkpoint.id,
        session_id: session_id,
        task_count: Object.keys(checkpoint.task_states).length
    })
    
    RETURN checkpoint.id
END

FUNCTION restore_checkpoint(checkpoint_id):
    checkpoint_file = STATE_CONFIG.checkpoints + "/checkpoint-" + checkpoint_id + ".json"
    
    IF not file_exists(checkpoint_file):
        THROW new Error("Checkpoint not found: " + checkpoint_id)
    END
    
    checkpoint = load_json_with_validation(checkpoint_file)
    
    # Restore session state
    IF checkpoint.session_state:
        session_file = STATE_CONFIG.sessions + "/" + checkpoint.session_id + ".json"
        write_atomic(session_file, checkpoint.session_state)
    END
    
    # Restore task states
    FOR task_id, task_state IN checkpoint.task_states:
        update_task_state(task_id, task_state, { force: true })
    END
    
    emit("state.checkpoint_restored", {
        checkpoint_id: checkpoint_id,
        session_id: checkpoint.session_id,
        tasks_restored: Object.keys(checkpoint.task_states).length
    })
    
    RETURN checkpoint
END
```

**STATE MONITORING:**

```
FUNCTION start_state_monitor():
    # Monitor for orphaned locks
    SET_INTERVAL(30000, () => {
        cleanup_expired_locks()
    })
    
    # Monitor for state integrity
    SET_INTERVAL(60000, () => {
        verify_all_states()
    })
    
    # Auto-checkpoint active sessions
    SET_INTERVAL(300000, () => {  # Every 5 minutes
        auto_checkpoint_sessions()
    })
END

FUNCTION cleanup_expired_locks():
    lock_files = find_files(STATE_CONFIG.root, "*.lock")
    cleaned = 0
    
    FOR lock_file IN lock_files:
        IF is_lock_expired(lock_file):
            remove_file(lock_file)
            cleaned += 1
        END
    END
    
    IF cleaned > 0:
        LOG_INFO: "Cleaned up expired locks", { count: cleaned }
    END
END

FUNCTION verify_all_states():
    issues = []
    
    # Check task states
    task_dirs = list_directories(STATE_CONFIG.tasks)
    FOR task_dir IN task_dirs:
        task_id = task_dir.replace("task-", "")
        state = get_task_state(task_id)
        
        IF state AND not verify_state_integrity(state):
            issues.push({
                type: "task_state_corrupted",
                task_id: task_id
            })
        END
    END
    
    IF issues.length > 0:
        emit("state.integrity_issues", { issues: issues })
    END
END
```

**UTILITY FUNCTIONS:**

```
FUNCTION write_atomic(file_path, data):
    temp_file = file_path + ".tmp"
    write_json(temp_file, data)
    rename_file(temp_file, file_path)
END

FUNCTION load_json_with_validation(file_path):
    content = read_file(file_path)
    
    TRY:
        data = JSON.parse(content)
        RETURN data
    CATCH error:
        LOG_ERROR: "Invalid JSON in file", {
            file: file_path,
            error: error.message
        }
        THROW error
    END
END

FUNCTION verify_state_integrity(state):
    # Basic integrity checks
    IF not state:
        RETURN false
    END
    
    IF not state.version OR not state.updated_at:
        RETURN false
    END
    
    # Check timestamp validity
    TRY:
        Date.parse(state.updated_at)
    CATCH:
        RETURN false
    END
    
    RETURN true
END

FUNCTION calculate_diff(old_obj, new_obj):
    diff = {
        added: {},
        modified: {},
        removed: {}
    }
    
    # Find added and modified
    FOR key, value IN new_obj:
        IF not (key IN old_obj):
            diff.added[key] = value
        ELIF old_obj[key] != value:
            diff.modified[key] = {
                old: old_obj[key],
                new: value
            }
        END
    END
    
    # Find removed
    FOR key IN old_obj:
        IF not (key IN new_obj):
            diff.removed[key] = old_obj[key]
        END
    END
    
    RETURN diff
END

FUNCTION migrate_legacy_state():
    legacy_file = ".worktrees/.parallel-dev-state.json"
    
    IF not file_exists(legacy_file):
        RETURN
    END
    
    LOG_INFO: "Migrating legacy state file"
    
    legacy_state = load_json_with_validation(legacy_file)
    
    FOR session IN legacy_state.sessions:
        # Create session file
        session_file = STATE_CONFIG.sessions + "/" + session.id + ".json"
        write_atomic(session_file, session)
        
        # Create task states
        FOR task_id, task_data IN session.tasks:
            update_task_state(task_id, task_data)
        END
    END
    
    # Rename legacy file
    rename_file(legacy_file, legacy_file + ".migrated")
    
    LOG_INFO: "Legacy state migration completed"
END
```

**EXPORTS:**
- `init_distributed_state()` -> Initialize distributed state system
- `get_task_state(task_id, options)` -> Get task state with locking
- `update_task_state(task_id, updates, options)` -> Update task state atomically
- `create_session(options)` -> Create new session
- `update_session(session_id, updates)` -> Update session
- `get_active_sessions()` -> Get all active sessions
- `create_checkpoint(session_id, reason)` -> Create state checkpoint
- `restore_checkpoint(checkpoint_id)` -> Restore from checkpoint
- `acquire_lock(lock_file, mode, timeout)` -> Acquire distributed lock
- `release_lock(lock_info)` -> Release lock