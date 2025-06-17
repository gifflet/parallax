**STATE MANAGEMENT MODULE**

Persist and recover execution state for reliability and resumability.

**STATE FILE:** `.worktrees/.parallel-dev-state.json`

**STATE STRUCTURE:**
```json
{
  "version": "2.0",
  "sessions": [
    {
      "id": "2024-06-17-001",
      "status": "active|completed|failed",
      "started_at": "2024-06-17T10:00:00Z",
      "updated_at": "2024-06-17T10:30:00Z",
      "options": {
        "max_agents": 5,
        "review_mode": "balanced",
        "profile": "default"
      },
      "tasks": {
        "8": {
          "status": "completed",
          "branch": "feature/task-8",
          "worktree": ".worktrees/task-8",
          "pr_number": 12,
          "started_at": "2024-06-17T10:00:00Z",
          "completed_at": "2024-06-17T10:20:00Z",
          "agent_logs": ["dev-1", "review-1", "final-1"]
        },
        "9": {
          "status": "in-progress",
          "branch": "feature/task-9",
          "worktree": ".worktrees/task-9",
          "current_phase": "review",
          "started_at": "2024-06-17T10:15:00Z",
          "agent_logs": ["dev-2", "review-2"]
        }
      },
      "metrics": {
        "total_tasks": 3,
        "completed": 1,
        "in_progress": 1,
        "failed": 0,
        "average_time_minutes": 20
      }
    }
  ]
}
```

**CORE FUNCTIONS:**

```
FUNCTION save_state(session_data):
    state_file = ".worktrees/.parallel-dev-state.json"
    
    # Ensure directory exists
    CREATE_DIR: .worktrees if not exists
    
    # Load existing state or create new
    IF file_exists(state_file):
        state = load_json(state_file)
    ELSE:
        state = { version: "2.0", sessions: [] }
    
    # Update or add session
    session_index = find_session_index(state.sessions, session_data.id)
    IF session_index >= 0:
        state.sessions[session_index] = session_data
    ELSE:
        state.sessions.push(session_data)
    
    # Keep only last 10 sessions
    IF state.sessions.length > 10:
        state.sessions = state.sessions.slice(-10)
    
    # Write atomically
    temp_file = state_file + ".tmp"
    write_json(temp_file, state)
    rename_file(temp_file, state_file)
    
    RETURN success
END

FUNCTION load_state():
    state_file = ".worktrees/.parallel-dev-state.json"
    
    IF not file_exists(state_file):
        RETURN null
    
    state = load_json(state_file)
    
    # Find most recent active session
    active_sessions = state.sessions.filter(s => s.status == "active")
    IF active_sessions.length == 0:
        RETURN null
    
    # Return most recent
    RETURN active_sessions.sort_by(s => s.updated_at).last()
END

FUNCTION get_active_tasks():
    session = load_state()
    IF not session:
        RETURN []
    
    active_tasks = []
    FOR task_id, task_data IN session.tasks:
        IF task_data.status IN ["in-progress", "pending", "reviewing"]:
            active_tasks.push({
                id: task_id,
                ...task_data
            })
    END
    
    RETURN active_tasks
END

FUNCTION cleanup_stale_sessions():
    state_file = ".worktrees/.parallel-dev-state.json"
    IF not file_exists(state_file):
        RETURN
    
    state = load_json(state_file)
    current_time = now()
    stale_threshold = 24 * 60 * 60 * 1000  # 24 hours
    
    # Mark stale active sessions as failed
    FOR session IN state.sessions:
        IF session.status == "active":
            last_update = parse_time(session.updated_at)
            IF current_time - last_update > stale_threshold:
                session.status = "stale"
                session.stale_reason = "No activity for 24 hours"
            END
        END
    END
    
    save_json(state_file, state)
END

FUNCTION create_session(options, task_ids):
    session = {
        id: generate_session_id(),
        status: "active",
        started_at: iso_timestamp(),
        updated_at: iso_timestamp(),
        options: options,
        tasks: {},
        metrics: {
            total_tasks: task_ids.length,
            completed: 0,
            in_progress: 0,
            failed: 0
        }
    }
    
    # Initialize task entries
    FOR task_id IN task_ids:
        session.tasks[task_id] = {
            status: "pending",
            branch: options.branch_pattern.replace("{id}", task_id),
            worktree: ".worktrees/task-" + task_id
        }
    END
    
    RETURN session
END

FUNCTION update_task_status(session_id, task_id, status, additional_data = {}):
    state_file = ".worktrees/.parallel-dev-state.json"
    state = load_json(state_file)
    
    session = find_session(state.sessions, session_id)
    IF not session:
        ERROR: "Session not found: " + session_id
    
    # Update task
    IF task_id IN session.tasks:
        session.tasks[task_id] = {
            ...session.tasks[task_id],
            status: status,
            ...additional_data,
            updated_at: iso_timestamp()
        }
    END
    
    # Update metrics
    recalculate_metrics(session)
    
    # Update session timestamp
    session.updated_at = iso_timestamp()
    
    save_json(state_file, state)
END
```

**RECOVERY FUNCTIONS:**

```
FUNCTION recover_session(session):
    DISPLAY: "Recovering session: " + session.id
    DISPLAY: "Started: " + session.started_at
    
    # Verify worktrees still exist
    FOR task_id, task IN session.tasks:
        IF task.status == "in-progress":
            IF not directory_exists(task.worktree):
                WARN: "Worktree missing for task " + task_id
                task.status = "failed"
                task.failure_reason = "Worktree not found"
            END
        END
    END
    
    # Check git branches
    FOR task_id, task IN session.tasks:
        IF task.branch AND task.status != "completed":
            IF not git_branch_exists(task.branch):
                WARN: "Branch missing for task " + task_id
                task.status = "failed" 
                task.failure_reason = "Branch not found"
            END
        END
    END
    
    RETURN session
END
```

**UTILITY FUNCTIONS:**

```
FUNCTION generate_session_id():
    date = format_date(now(), "YYYY-MM-DD")
    counter = get_daily_counter()
    RETURN date + "-" + pad_number(counter, 3)
END

FUNCTION recalculate_metrics(session):
    metrics = {
        total_tasks: 0,
        completed: 0,
        in_progress: 0,
        failed: 0,
        pending: 0
    }
    
    FOR task_id, task IN session.tasks:
        metrics.total_tasks += 1
        CASE task.status:
            "completed": metrics.completed += 1
            "in-progress", "reviewing": metrics.in_progress += 1
            "failed": metrics.failed += 1
            "pending": metrics.pending += 1
        END
    END
    
    # Calculate average time
    completed_times = []
    FOR task_id, task IN session.tasks:
        IF task.status == "completed" AND task.completed_at:
            duration = parse_time(task.completed_at) - parse_time(task.started_at)
            completed_times.push(duration / 60000)  # Convert to minutes
        END
    END
    
    IF completed_times.length > 0:
        metrics.average_time_minutes = average(completed_times)
    END
    
    session.metrics = metrics
END
```

**EXPORTS:**
- `save_state(session_data)` -> Save session state
- `load_state()` -> Load active session
- `get_active_tasks()` -> Get tasks in progress
- `cleanup_stale_sessions()` -> Clean old sessions
- `create_session(options, task_ids)` -> Create new session
- `update_task_status(session_id, task_id, status, data)` -> Update task
- `recover_session(session)` -> Recover interrupted session