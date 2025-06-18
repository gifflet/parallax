**PROGRESS TRACKER MODULE**

Real-time visual progress monitoring for parallel development execution.

**CORE CONCEPTS:**
- Multiple display modes (compact, detailed, json)
- Real-time task status updates
- Resource usage monitoring
- Final report generation

**DISPLAY MODES:**

```
COMPACT MODE (Default):
╭─ PARALLEL DEV STATUS ─────────────────────────╮
│ Session: 2024-06-17-001    Uptime: 00:15:32  │
│ Tasks:   [████████░░] 80% (8/10)             │
│ Agents:  [██████████] 3/3 active             │
│                                               │
│ Task 8:  ✅ Completed (PR #12 merged)        │
│ Task 9:  🔄 In Review (agent-2) [12:45]      │
│ Task 14: 🚀 Development (75%) [08:30]        │
╰───────────────────────────────────────────────╯
```

**MAIN FUNCTIONS:**

```
FUNCTION init_progress_display(session, mode = "compact"):
    clear_screen()
    
    config = {
        mode: mode,
        session_id: session.id,
        start_time: Date.now(),
        update_interval: 1000,
        last_update: 0
    }
    
    # Start update loop
    start_progress_updates(config, session)
    
    RETURN config
END

FUNCTION update_progress(config, session_state):
    # Throttle updates
    IF Date.now() - config.last_update < config.update_interval:
        RETURN
    END
    
    # Render based on mode
    IF config.mode == "compact":
        render_compact(session_state)
    ELIF config.mode == "detailed":
        render_detailed(session_state)
    ELIF config.mode == "json":
        render_json(session_state)
    END
    
    config.last_update = Date.now()
END

FUNCTION render_compact(state):
    lines = []
    
    # Header
    elapsed = format_duration(Date.now() - state.started_at)
    lines.push("╭─ PARALLEL DEV STATUS ─────────────────────────╮")
    lines.push("│ Session: " + state.id + "    Uptime: " + elapsed + "  │")
    
    # Progress bars
    task_progress = render_progress_bar(state.metrics.completed, state.metrics.total_tasks)
    agent_status = render_agent_bar(state.agents)
    lines.push("│ Tasks:   " + task_progress + "             │")
    lines.push("│ Agents:  " + agent_status + "             │")
    lines.push("│                                               │")
    
    # Active tasks
    FOR task IN get_active_tasks(state):
        lines.push("│ " + render_task_line(task) + " │")
    END
    
    # Footer
    lines.push("╰───────────────────────────────────────────────╯")
    
    # Clear and print
    restore_cursor()
    print(lines.join("\n"))
END

FUNCTION render_task_line(task):
    icons = {
        completed: "✅",
        in_progress: "🚀",
        reviewing: "🔄",
        failed: "❌",
        pending: "⏳"
    }
    
    icon = icons[task.status]
    text = "Task " + task.id + ": "
    
    IF task.status == "completed":
        text += "Completed"
        IF task.pr_number:
            text += " (PR #" + task.pr_number + ")"
        END
    ELIF task.status == "in_progress":
        text += "Development"
        IF task.progress:
            text += " (" + task.progress + "%)"
        END
        IF task.elapsed:
            text += " [" + format_duration(task.elapsed) + "]"
        END
    ELIF task.status == "reviewing":
        text += "In Review"
        IF task.current_agent:
            text += " (" + task.current_agent + ")"
        END
    END
    
    RETURN pad_right(icon + " " + text, 45)
END

FUNCTION render_final_report(session):
    duration = format_duration(Date.now() - session.started_at)
    success_rate = Math.round((session.metrics.completed / session.metrics.total_tasks) * 100)
    
    report = [
        "╔════════════════════════════════════════════════╗",
        "║           EXECUTION COMPLETE                   ║",
        "╠════════════════════════════════════════════════╣",
        "║ Session: " + session.id,
        "║ Duration: " + duration,
        "║",
        "║ RESULTS:",
        "║ ✅ Completed: " + session.metrics.completed + " tasks",
        "║ ❌ Failed: " + session.metrics.failed + " tasks",
        "║ 📊 Success Rate: " + success_rate + "%",
        "║",
        "║ PULL REQUESTS:",
    ]
    
    # Add PR list
    pr_count = 0
    FOR task IN session.tasks:
        IF task.pr_number:
            status = task.merged ? "merged" : "open"
            report.push("║ - PR #" + task.pr_number + ": Task " + task.id + " (" + status + ")")
            pr_count++
        END
    END
    
    IF pr_count == 0:
        report.push("║ No pull requests created")
    END
    
    report.push("╚════════════════════════════════════════════════╝")
    
    print(report.join("\n"))
END
```

**UTILITY FUNCTIONS:**
```
FUNCTION render_progress_bar(completed, total):
    percentage = Math.round((completed / total) * 100)
    filled = Math.round((completed / total) * 10)
    bar = "[" + "█".repeat(filled) + "░".repeat(10 - filled) + "]"
    RETURN bar + " " + percentage + "% (" + completed + "/" + total + ")"
END

FUNCTION format_duration(ms):
    hours = Math.floor(ms / 3600000)
    minutes = Math.floor((ms % 3600000) / 60000)
    seconds = Math.floor((ms % 60000) / 1000)
    
    parts = []
    IF hours > 0: parts.push(hours + "h")
    IF minutes > 0: parts.push(minutes + "m")
    IF seconds > 0 OR parts.length == 0: parts.push(seconds + "s")
    
    RETURN parts.join(" ")
END
```

**EXPORTS:**
- init_progress_display(session, mode?) -> Config
- update_progress(config, state) -> Update display
- render_final_report(session) -> Show final summary
- notify_task_complete(task) -> Show completion
- notify_error(task, error) -> Show error
- display_status_check() -> Current status
- show_help() -> Display help
- format_duration(ms) -> Formatted time string