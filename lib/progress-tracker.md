**PROGRESS TRACKING MODULE**

Real-time visual progress monitoring for parallel development execution.

**PURPOSE:** Provide clear, formatted progress feedback during multi-agent execution

**IMPORTS:**
```
- ./state-manager.md    # For retrieving task states
```

**DISPLAY FORMATS:**

**COMPACT MODE (Default):**
```
╭─ PARALLEL DEV STATUS ─────────────────────────╮
│ Session: 2024-06-17-001    Uptime: 00:15:32  │
│ Tasks:   [████████░░] 80% (8/10)             │
│ Agents:  [██████████] 3/3 active             │
│                                               │
│ Task 8:  ✅ Completed (PR #12 merged)        │
│ Task 9:  🔄 In Review (agent-2) [12:45]      │
│ Task 14: 🚀 Development (75%) [08:30]        │
│ Task 10: ⏳ Queued (waiting for agents)      │
│                                               │
│ Avg time: 18 min | Est. remaining: 25 min    │
╰───────────────────────────────────────────────╯
```

**DETAILED MODE:**
```
╭─ PARALLEL DEVELOPMENT ORCHESTRATION ──────────────────────────────╮
│ Session ID: 2024-06-17-001                                        │
│ Profile: balanced | Max Agents: 5 | Review: balanced              │
├───────────────────────────────────────────────────────────────────┤
│ OVERALL PROGRESS                                                  │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│ Tasks:     [████████████████░░░░░░] 60% (6/10)                  │
│ Time:      00:45:23 elapsed | ~00:30:00 remaining                │
│ Agents:    ████ Dev (2) | ██ Review (1) | ░░ Idle (2)          │
├───────────────────────────────────────────────────────────────────┤
│ TASK DETAILS                                                      │
│                                                                   │
│ ✅ Task 1: User Authentication      [00:12:45] PR #8  merged     │
│ ✅ Task 3: Database Models          [00:18:32] PR #9  merged     │
│ ✅ Task 8: Average Price Calc       [00:20:15] PR #12 merged     │
│                                                                   │
│ 🔄 Task 9: Monthly Profit Calc      [00:15:00] Review pending    │
│    └─ Agent: reviewer-1 | Files: 5 changed, +450 -20            │
│                                                                   │
│ 🚀 Task 14: Financial Reports       [00:08:30] Dev 75% complete  │
│    └─ Agent: developer-2 | Branch: feature/task-14              │
│    └─ Progress: Models ✓ | Logic ✓ | Tests ▶ | Docs ░          │
│                                                                   │
│ 🛠️  Task 10: MCP Integration        [00:05:15] Dev 40% complete  │
│    └─ Agent: developer-3 | Branch: feature/task-10              │
│                                                                   │
│ ⏳ Task 11: RESTful API             Queued (depends on 9,10)     │
│ ⏳ Task 12: CSV Upload              Queued (depends on 11)       │
│ ⏳ Task 13: OpenAI Integration      Queued (low priority)        │
├───────────────────────────────────────────────────────────────────┤
│ RESOURCE USAGE                                                    │
│ CPU: [████████░░] 78% | Memory: [██████░░░░] 62% | Disk: 145MB  │
│ Git Worktrees: 3 active | Branches: 3 active | PRs: 1 open      │
╰───────────────────────────────────────────────────────────────────╯
```

**PROGRESS UPDATE FUNCTIONS:**

```
FUNCTION init_progress_display(session, mode = "compact"):
    CLEAR_SCREEN
    SAVE_CURSOR_POSITION
    
    display_config = {
        mode: mode,
        session_id: session.id,
        start_time: now(),
        update_interval: 1000,  # 1 second
        last_update: 0
    }
    
    RETURN display_config
END

FUNCTION update_progress(display_config, session_state):
    current_time = now()
    
    # Throttle updates
    IF current_time - display_config.last_update < display_config.update_interval:
        RETURN
    END
    
    RESTORE_CURSOR_POSITION
    
    IF display_config.mode == "compact":
        render_compact_display(session_state)
    ELSE:
        render_detailed_display(session_state)
    END
    
    display_config.last_update = current_time
END

FUNCTION render_compact_display(state):
    elapsed = format_duration(now() - state.started_at)
    
    # Build display lines
    lines = [
        "╭─ PARALLEL DEV STATUS ─────────────────────────╮",
        "│ Session: " + state.id + "    Uptime: " + elapsed + "  │",
        "│ Tasks:   " + render_progress_bar(state.metrics) + "             │",
        "│ Agents:  " + render_agent_status(state.agents) + "             │",
        "│                                               │"
    ]
    
    # Add task statuses
    FOR task_id, task IN state.tasks:
        IF task.status != "pending":
            line = "│ Task " + task_id + ":  " + render_task_status(task) + "        │"
            lines.push(pad_line(line, 50))
        END
    END
    
    # Add summary
    lines.push("│                                               │")
    lines.push("│ " + render_summary(state) + "    │")
    lines.push("╰───────────────────────────────────────────────╯")
    
    PRINT: lines.join("\n")
END

FUNCTION render_progress_bar(metrics):
    total = metrics.total_tasks
    completed = metrics.completed
    percentage = Math.round((completed / total) * 100)
    
    bar_width = 10
    filled = Math.round((completed / total) * bar_width)
    empty = bar_width - filled
    
    bar = "[" + "█".repeat(filled) + "░".repeat(empty) + "]"
    
    RETURN bar + " " + percentage + "% (" + completed + "/" + total + ")"
END

FUNCTION render_task_status(task):
    status_icons = {
        completed: "✅",
        in_progress: "🚀",
        reviewing: "🔄",
        failed: "❌",
        pending: "⏳",
        corrections: "🛠️"
    }
    
    icon = status_icons[task.status] || "❓"
    
    status_text = task.title || "Task " + task.id
    
    IF task.status == "completed" AND task.pr_number:
        status_text += " (PR #" + task.pr_number
        IF task.merged:
            status_text += " merged)"
        ELSE:
            status_text += ")"
        END
    ELIF task.status IN ["in_progress", "reviewing"]:
        IF task.current_agent:
            status_text += " (" + task.current_agent + ")"
        END
        IF task.elapsed_time:
            status_text += " [" + format_duration(task.elapsed_time) + "]"
        END
        IF task.progress_percentage:
            status_text += " (" + task.progress_percentage + "%)"
        END
    END
    
    RETURN icon + " " + status_text
END
```

**AGENT STATUS RENDERING:**

```
FUNCTION render_agent_status(agents):
    active = agents.filter(a => a.status == "active").length
    total = agents.length
    
    bar_width = 10
    filled = Math.round((active / total) * bar_width)
    empty = bar_width - filled
    
    bar = "[" + "█".repeat(filled) + "░".repeat(empty) + "]"
    
    RETURN bar + " " + active + "/" + total + " active"
END

FUNCTION render_agent_details(agents):
    grouped = {
        developer: agents.filter(a => a.type == "developer"),
        reviewer: agents.filter(a => a.type == "reviewer"),
        corrector: agents.filter(a => a.type == "corrector"),
        finalizer: agents.filter(a => a.type == "finalizer")
    }
    
    parts = []
    FOR type, group IN grouped:
        IF group.length > 0:
            active = group.filter(a => a.status == "active").length
            icon = "█".repeat(active) + "░".repeat(group.length - active)
            parts.push(icon + " " + capitalize(type) + " (" + active + ")")
        END
    END
    
    RETURN parts.join(" | ")
END
```

**NOTIFICATION FUNCTIONS:**

```
FUNCTION notify_task_complete(task):
    message = "✅ Task " + task.id + " completed!"
    IF task.pr_number:
        message += " PR #" + task.pr_number + " created"
    END
    
    display_notification(message, "success")
END

FUNCTION notify_error(task, error):
    message = "❌ Task " + task.id + " failed: " + error
    display_notification(message, "error")
END

FUNCTION display_notification(message, type = "info"):
    colors = {
        success: "\033[32m",  # Green
        error: "\033[31m",    # Red
        warning: "\033[33m",  # Yellow
        info: "\033[36m"      # Cyan
    }
    
    reset = "\033[0m"
    color = colors[type] || colors.info
    
    # Save current position
    SAVE_CURSOR
    
    # Move to notification area (bottom of display)
    MOVE_CURSOR_TO(0, 25)
    CLEAR_LINE
    
    PRINT: color + "📢 " + message + reset
    
    # Restore position
    RESTORE_CURSOR
    
    # Clear notification after 5 seconds
    SET_TIMEOUT(5000, () => {
        SAVE_CURSOR
        MOVE_CURSOR_TO(0, 25)
        CLEAR_LINE
        RESTORE_CURSOR
    })
END
```

**SUMMARY CALCULATIONS:**

```
FUNCTION render_summary(state):
    avg_time = state.metrics.average_time_minutes || 0
    remaining_tasks = state.metrics.total_tasks - state.metrics.completed
    est_remaining = avg_time * remaining_tasks
    
    summary = "Avg time: " + Math.round(avg_time) + " min"
    
    IF remaining_tasks > 0:
        summary += " | Est. remaining: " + Math.round(est_remaining) + " min"
    END
    
    RETURN summary
END

FUNCTION calculate_task_progress(task):
    IF task.subtasks AND task.subtasks.length > 0:
        completed = task.subtasks.filter(st => st.completed).length
        total = task.subtasks.length
        RETURN Math.round((completed / total) * 100)
    END
    
    # Estimate based on phase
    phase_weights = {
        planning: 10,
        development: 50,
        testing: 20,
        review: 15,
        finalization: 5
    }
    
    completed_weight = 0
    FOR phase, weight IN phase_weights:
        IF task.completed_phases.includes(phase):
            completed_weight += weight
        ELIF task.current_phase == phase:
            completed_weight += weight * 0.5  # Half credit for in-progress
        END
    END
    
    RETURN completed_weight
END
```

**FINAL REPORT GENERATION:**
```
FUNCTION render_final_report(session):
    total_duration = format_duration(now() - session.started_at)
    success_rate = (session.metrics.completed / session.metrics.total_tasks) * 100
    
    DISPLAY: """
╔════════════════════════════════════════════════════════════════╗
║                    EXECUTION COMPLETE                          ║
╠════════════════════════════════════════════════════════════════╣
║ Session: {session.id}                                          ║
║ Duration: {total_duration}                                     ║
║                                                                ║
║ RESULTS:                                                       ║
║ ✅ Completed: {session.metrics.completed} tasks                ║
║ ❌ Failed: {session.metrics.failed} tasks                      ║
║ 📊 Success Rate: {success_rate}%                              ║
║ ⏱️  Avg Time/Task: {session.metrics.average_time_minutes} min  ║
║                                                                ║
║ PULL REQUESTS:                                                 ║
"""
    
    # List PRs created
    pr_count = 0
    FOR task_id, task IN session.tasks:
        IF task.pr_number:
            pr_count += 1
            status = task.merged ? "merged" : "open"
            DISPLAY: "║ - PR #{task.pr_number}: Task {task_id} ({status})         ║"
        END
    END
    
    IF pr_count == 0:
        DISPLAY: "║ No pull requests created                                      ║"
    END
    
    DISPLAY: "║                                                                ║"
    
    # List any preserved resources
    preserved = list_preserved_resources()
    IF preserved.length > 0:
        DISPLAY: "║ PRESERVED RESOURCES:                                           ║"
        FOR resource IN preserved:
            DISPLAY: "║ - Task {resource.task_id}: {resource.reason}              ║"
        END
    END
    
    DISPLAY: "╚════════════════════════════════════════════════════════════════╝"
END
```

**UTILITY FUNCTIONS:**
```
FUNCTION format_duration(milliseconds):
    hours = Math.floor(milliseconds / 3600000)
    minutes = Math.floor((milliseconds % 3600000) / 60000)
    seconds = Math.floor((milliseconds % 60000) / 1000)
    
    parts = []
    IF hours > 0:
        parts.push(hours + "h")
    END
    IF minutes > 0:
        parts.push(minutes + "m")
    END
    IF seconds > 0 OR parts.length == 0:
        parts.push(seconds + "s")
    END
    
    RETURN parts.join(" ")
END

FUNCTION pad_line(text, width):
    current_length = text.length
    IF current_length < width:
        RETURN text + " ".repeat(width - current_length)
    END
    RETURN text.substring(0, width)
END

FUNCTION capitalize(text):
    RETURN text.charAt(0).toUpperCase() + text.slice(1)
END
```

**ANSI TERMINAL FUNCTIONS:**
```
# Terminal control sequences for display manipulation
CLEAR_SCREEN = "\033[2J\033[H"
SAVE_CURSOR = "\033[s"
RESTORE_CURSOR = "\033[u"
CLEAR_LINE = "\033[2K"
MOVE_CURSOR_TO = (x, y) => "\033[" + y + ";" + x + "H"
```

**EXPORTS:**
- `init_progress_display(session, mode)` -> Initialize display
- `update_progress(config, state)` -> Update display
- `notify_task_complete(task)` -> Show completion notification
- `notify_error(task, error)` -> Show error notification
- `render_final_report(session)` -> Generate final summary
- `display_status_check(session_id)` -> Show current status