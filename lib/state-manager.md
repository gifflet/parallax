**STATE MANAGER MODULE**

Persist and recover execution state for reliability and resumability.

**CORE CONCEPTS:**
- Session-based state management
- Task progress tracking
- Metrics calculation
- State persistence and recovery

**STATE STRUCTURE:**
```json
{
  "version": "2.0",
  "sessions": [{
    "id": "2024-06-17-001",
    "status": "active|completed|failed",
    "started_at": "2024-06-17T10:00:00Z",
    "updated_at": "2024-06-17T10:30:00Z",
    "options": {},
    "tasks": {
      "task_id": {
        "status": "pending|in_progress|completed|failed",
        "branch": "feature/task-id",
        "worktree": ".worktrees/task-id",
        "pr_number": null,
        "phases": {},
        "metrics": {}
      }
    },
    "metrics": {
      "total_tasks": 10,
      "completed": 5,
      "in_progress": 2,
      "failed": 0,
      "average_time_minutes": 25
    }
  }]
}
```

**MAIN FUNCTIONS:**

```
STATE_FILE = ".worktrees/.parallel-dev-state.json"

FUNCTION load_state():
    IF not file_exists(STATE_FILE):
        RETURN { version: "2.0", sessions: [] }
    END
    
    TRY:
        state = JSON.parse(read_file(STATE_FILE))
        RETURN migrate_state_if_needed(state)
    CATCH:
        # Backup corrupted state
        backup_state()
        RETURN { version: "2.0", sessions: [] }
    END
END

FUNCTION save_state(state):
    ensure_directory_exists(dirname(STATE_FILE))
    write_file_atomic(STATE_FILE, JSON.stringify(state, null, 2))
END

FUNCTION create_session(options):
    state = load_state()
    
    session = {
        id: generate_session_id(),
        status: "active",
        started_at: iso_timestamp(),
        updated_at: iso_timestamp(),
        options: options,
        tasks: {},
        metrics: {
            total_tasks: 0,
            completed: 0,
            in_progress: 0,
            failed: 0,
            average_time_minutes: 0
        }
    }
    
    state.sessions.push(session)
    save_state(state)
    
    RETURN session
END

FUNCTION update_session(session_id, updates):
    state = load_state()
    session = state.sessions.find(s => s.id == session_id)
    
    IF not session:
        throw new Error("Session not found: " + session_id)
    END
    
    # Merge updates
    Object.assign(session, updates)
    session.updated_at = iso_timestamp()
    
    # Recalculate metrics
    session.metrics = calculate_session_metrics(session)
    
    save_state(state)
    RETURN session
END

FUNCTION get_active_sessions():
    state = load_state()
    RETURN state.sessions.filter(s => s.status == "active")
END

FUNCTION update_task_progress(session_id, task_id, progress):
    state = load_state()
    session = state.sessions.find(s => s.id == session_id)
    
    IF not session.tasks[task_id]:
        session.tasks[task_id] = {
            status: "pending",
            started_at: null,
            completed_at: null
        }
    END
    
    task = session.tasks[task_id]
    
    # Update task based on progress
    IF progress.status:
        task.status = progress.status
        IF progress.status == "in_progress" AND not task.started_at:
            task.started_at = iso_timestamp()
        ELIF progress.status == "completed":
            task.completed_at = iso_timestamp()
        END
    END
    
    # Merge other progress data
    Object.assign(task, progress)
    
    # Update session
    session.updated_at = iso_timestamp()
    session.metrics = calculate_session_metrics(session)
    
    save_state(state)
    RETURN task
END

FUNCTION calculate_session_metrics(session):
    tasks = Object.values(session.tasks)
    
    metrics = {
        total_tasks: tasks.length,
        completed: tasks.filter(t => t.status == "completed").length,
        in_progress: tasks.filter(t => t.status == "in_progress").length,
        failed: tasks.filter(t => t.status == "failed").length,
        average_time_minutes: 0
    }
    
    # Calculate average time
    completed_times = tasks
        .filter(t => t.completed_at && t.started_at)
        .map(t => {
            start = Date.parse(t.started_at)
            end = Date.parse(t.completed_at)
            RETURN (end - start) / 60000  # minutes
        })
    
    IF completed_times.length > 0:
        metrics.average_time_minutes = Math.round(
            completed_times.reduce((a, b) => a + b, 0) / completed_times.length
        )
    END
    
    RETURN metrics
END

FUNCTION recover_session(session_id):
    state = load_state()
    session = state.sessions.find(s => s.id == session_id)
    
    IF not session:
        throw new Error("Session not found: " + session_id)
    END
    
    # Mark as active
    session.status = "active"
    session.recovered_at = iso_timestamp()
    
    save_state(state)
    RETURN session
END

FUNCTION cleanup_old_sessions(days = 7):
    state = load_state()
    cutoff_date = Date.now() - (days * 24 * 60 * 60 * 1000)
    
    state.sessions = state.sessions.filter(s => {
        session_date = Date.parse(s.updated_at || s.started_at)
        RETURN session_date > cutoff_date || s.status == "active"
    })
    
    save_state(state)
END
```

**EXPORTS:**
- load_state() -> State object
- save_state(state) -> Save to disk
- create_session(options) -> Session
- update_session(id, updates) -> Updated session
- get_active_sessions() -> Active sessions
- update_task_progress(session_id, task_id, progress) -> Task
- calculate_session_metrics(session) -> Metrics
- recover_session(id) -> Recovered session
- cleanup_old_sessions(days) -> Clean old data