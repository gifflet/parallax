**MONITORING DASHBOARD MODULE**

Provides real-time monitoring dashboard with metrics, alerts, and system insights.

**PURPOSE:** Unified monitoring interface for all system components with live updates

**IMPORTS:**
```
- ./event-bus.md             # For real-time events
- ./distributed-state.md     # For state monitoring
- ./performance-optimizer.md # For performance metrics
- ./error-handler.md         # For error tracking
- ./health-monitor.md        # For health status
- ./config-manager.md        # For configuration
```

**DASHBOARD COMPONENTS:**
```
DASHBOARD_SECTIONS = {
    overview: "System Overview",
    tasks: "Task Progress",
    agents: "Agent Activity",
    performance: "Performance Metrics",
    errors: "Error Tracking",
    health: "Health Status",
    resources: "Resource Usage"
}

UPDATE_INTERVALS = {
    real_time: 1000,      # 1 second
    frequent: 5000,       # 5 seconds
    normal: 30000,        # 30 seconds
    slow: 60000          # 1 minute
}
```

**CORE DASHBOARD:**

```
CLASS MonitoringDashboard {
    constructor(options = {}) {
        this.mode = options.mode || "auto"  # auto, compact, detailed, json
        this.refresh_rate = options.refresh_rate || UPDATE_INTERVALS.frequent
        this.enabled_sections = options.sections || Object.keys(DASHBOARD_SECTIONS)
        this.output_stream = options.output_stream || process.stdout
        
        this.data = {}
        this.subscribers = new Map()
        this.update_timers = new Map()
        this.running = false
    }
    
    async start() {
        IF this.running:
            RETURN
        END
        
        this.running = true
        
        # Initial data collection
        await this.collect_all_data()
        
        # Subscribe to real-time events
        this.setup_event_subscriptions()
        
        # Start update cycles
        this.start_update_cycles()
        
        # Initial render
        this.render()
        
        emit("dashboard.started")
    }
    
    stop() {
        this.running = false
        
        # Clear timers
        FOR [section, timer] IN this.update_timers:
            clearInterval(timer)
        END
        this.update_timers.clear()
        
        # Unsubscribe from events
        FOR [event, handler_id] IN this.subscribers:
            unsubscribe(handler_id)
        END
        this.subscribers.clear()
        
        emit("dashboard.stopped")
    }
    
    setup_event_subscriptions() {
        # Real-time task updates
        this.subscribe("task.*", (event) => {
            this.update_section_data("tasks", async () => {
                RETURN await this.collect_task_data()
            })
        })
        
        # Agent activity
        this.subscribe("agent.*", (event) => {
            this.update_section_data("agents", async () => {
                RETURN await this.collect_agent_data()
            })
        })
        
        # Error events
        this.subscribe("error.*", (event) => {
            this.update_section_data("errors", async () => {
                RETURN await this.collect_error_data()
            })
        })
        
        # Performance metrics
        this.subscribe("performance.*", (event) => {
            this.update_section_data("performance", async () => {
                RETURN await this.collect_performance_data()
            })
        })
        
        # Health updates
        this.subscribe("health.*", (event) => {
            this.update_section_data("health", async () => {
                RETURN await this.collect_health_data()
            })
        })
    }
    
    subscribe(event_pattern, handler) {
        wrapped_handler = (event) => {
            handler(event)
            this.schedule_render()
        }
        
        handler_id = subscribe(event_pattern, wrapped_handler)
        this.subscribers.set(event_pattern, handler_id)
    }
    
    start_update_cycles() {
        # Different update frequencies for different data
        update_schedules = {
            overview: UPDATE_INTERVALS.frequent,
            tasks: UPDATE_INTERVALS.real_time,
            agents: UPDATE_INTERVALS.real_time,
            performance: UPDATE_INTERVALS.frequent,
            errors: UPDATE_INTERVALS.frequent,
            health: UPDATE_INTERVALS.normal,
            resources: UPDATE_INTERVALS.frequent
        }
        
        FOR [section, interval] IN update_schedules:
            IF this.enabled_sections.includes(section):
                timer = setInterval(async () => {
                    await this.update_section(section)
                }, interval)
                
                this.update_timers.set(section, timer)
            END
        END
    }
    
    async collect_all_data() {
        collection_tasks = []
        
        IF this.enabled_sections.includes("overview"):
            collection_tasks.push(this.collect_overview_data())
        END
        
        IF this.enabled_sections.includes("tasks"):
            collection_tasks.push(this.collect_task_data())
        END
        
        IF this.enabled_sections.includes("agents"):
            collection_tasks.push(this.collect_agent_data())
        END
        
        IF this.enabled_sections.includes("performance"):
            collection_tasks.push(this.collect_performance_data())
        END
        
        IF this.enabled_sections.includes("errors"):
            collection_tasks.push(this.collect_error_data())
        END
        
        IF this.enabled_sections.includes("health"):
            collection_tasks.push(this.collect_health_data())
        END
        
        IF this.enabled_sections.includes("resources"):
            collection_tasks.push(this.collect_resource_data())
        END
        
        results = await Promise.all(collection_tasks)
        
        # Update data
        index = 0
        FOR section IN this.enabled_sections:
            this.data[section] = results[index++]
        END
    }
    
    async collect_overview_data() {
        sessions = await get_active_sessions()
        current_session = sessions[0]
        
        IF not current_session:
            RETURN {
                status: "No active session",
                uptime: "N/A",
                progress: 0,
                eta: "N/A"
            }
        END
        
        all_task_states = {}
        FOR task_id IN current_session.task_ids:
            state = await get_task_state(task_id)
            IF state:
                all_task_states[task_id] = state
            END
        END
        
        completed = Object.values(all_task_states).filter(t => t.status == "completed").length
        total = current_session.task_ids.length
        progress = total > 0 ? (completed / total) * 100 : 0
        
        # Calculate ETA
        elapsed = Date.now() - Date.parse(current_session.started_at)
        IF completed > 0:
            avg_time_per_task = elapsed / completed
            remaining_tasks = total - completed
            eta_ms = avg_time_per_task * remaining_tasks
            eta = format_duration(eta_ms)
        ELSE:
            eta = "Calculating..."
        END
        
        RETURN {
            session_id: current_session.id,
            status: current_session.status,
            uptime: format_duration(elapsed),
            progress: progress,
            completed: completed,
            total: total,
            eta: eta
        }
    }
    
    async collect_task_data() {
        sessions = await get_active_sessions()
        IF sessions.length == 0:
            RETURN { tasks: [] }
        END
        
        current_session = sessions[0]
        tasks = []
        
        FOR task_id IN current_session.task_ids:
            state = await get_task_state(task_id)
            IF state:
                tasks.push({
                    id: task_id,
                    status: state.status,
                    progress: state.progress || 0,
                    current_phase: state.current_phase,
                    duration: state.started_at ? 
                        Date.now() - Date.parse(state.started_at) : 0,
                    agent: state.current_agent,
                    error: state.error
                })
            END
        END
        
        RETURN {
            tasks: tasks,
            by_status: group_by(tasks, "status"),
            active_count: tasks.filter(t => t.status == "in_progress").length
        }
    }
    
    async collect_agent_data() {
        agent_states = await get_all_agent_states()
        
        agents = agent_states.map(agent => ({
            id: agent.id,
            type: agent.type,
            task_id: agent.task_id,
            status: agent.status,
            duration: agent.started_at ? 
                Date.now() - Date.parse(agent.started_at) : 0,
            last_progress: agent.last_progress,
            resource_usage: agent.resource_usage
        }))
        
        RETURN {
            agents: agents,
            by_type: group_by(agents, "type"),
            active_count: agents.filter(a => a.status == "running").length,
            utilization: calculate_agent_utilization(agents)
        }
    }
    
    async collect_performance_data() {
        perf_report = get_performance_report()
        
        RETURN {
            operations: perf_report.operations,
            cache_hit_rate: perf_report.summary.cache_hit_rate,
            pool_utilization: perf_report.summary.pool_utilization,
            slow_operations: Object.entries(perf_report.operations)
                .filter(([op, stats]) => stats.avg_duration > 1000)
                .map(([op, stats]) => ({
                    operation: op,
                    avg_duration: stats.avg_duration,
                    count: stats.count
                }))
        }
    }
    
    async collect_error_data() {
        error_dashboard = get_error_dashboard_data()
        
        RETURN {
            summary: error_dashboard.summary,
            recent_errors: error_dashboard.recent_errors.slice(0, 10),
            error_rate: error_dashboard.summary.error_rate_per_minute,
            critical_count: error_dashboard.summary.critical_errors,
            top_errors: error_dashboard.summary.top_errors.slice(0, 5)
        }
    }
    
    async collect_health_data() {
        health_dashboard = get_health_dashboard()
        
        RETURN {
            overall_status: health_dashboard.overview.status,
            components: health_dashboard.components,
            alerts: health_dashboard.alerts,
            uptime_percent: health_dashboard.overview.uptime_percent
        }
    }
    
    async collect_resource_data() {
        RETURN {
            memory: get_memory_info(),
            cpu: await get_cpu_info(),
            disk: await get_disk_info(),
            processes: get_process_info(),
            git_worktrees: await count_git_worktrees()
        }
    }
    
    render() {
        IF not this.running:
            RETURN
        END
        
        # Clear screen for terminal output
        IF this.output_stream == process.stdout:
            this.clear_screen()
        END
        
        # Select render mode
        CASE this.mode:
            "compact":
                this.render_compact()
            "detailed":
                this.render_detailed()
            "json":
                this.render_json()
            "auto":
                # Choose based on terminal size
                IF get_terminal_height() < 30:
                    this.render_compact()
                ELSE:
                    this.render_detailed()
                END
        END
    }
    
    render_compact() {
        output = []
        
        # Header
        output.push(this.render_header("PARALLEL DEV MONITOR", "compact"))
        
        # Overview
        IF this.data.overview:
            o = this.data.overview
            output.push(`Session: ${o.session_id} | Status: ${o.status}`)
            output.push(`Progress: ${this.render_progress_bar(o.progress)} ${o.completed}/${o.total}`)
            output.push(`Uptime: ${o.uptime} | ETA: ${o.eta}`)
            output.push("")
        END
        
        # Active tasks
        IF this.data.tasks:
            active_tasks = this.data.tasks.tasks.filter(t => t.status == "in_progress")
            IF active_tasks.length > 0:
                output.push("Active Tasks:")
                FOR task IN active_tasks.slice(0, 5):
                    output.push(`  Task ${task.id}: ${task.current_phase} (${task.progress}%)`)
                END
                output.push("")
            END
        END
        
        # Health status
        IF this.data.health:
            output.push(`Health: ${get_status_emoji(this.data.health.overall_status)} ${this.data.health.overall_status.toUpperCase()}`)
            IF this.data.health.alerts.length > 0:
                output.push("Alerts: " + this.data.health.alerts.length)
            END
        END
        
        this.output_stream.write(output.join("\n"))
    }
    
    render_detailed() {
        output = []
        
        # Header
        output.push(this.render_header("PARALLEL DEVELOPMENT MONITORING DASHBOARD", "detailed"))
        
        # Overview section
        IF this.data.overview:
            output.push(this.render_section("OVERVIEW", () => {
                o = this.data.overview
                RETURN [
                    `Session ID: ${o.session_id}`,
                    `Status: ${this.colorize(o.status, this.get_status_color(o.status))}`,
                    `Progress: ${this.render_progress_bar(o.progress, 30)} ${o.progress.toFixed(1)}% (${o.completed}/${o.total})`,
                    `Uptime: ${o.uptime} | ETA: ${o.eta}`
                ]
            }))
        END
        
        # Task progress section
        IF this.data.tasks:
            output.push(this.render_section("TASK PROGRESS", () => {
                lines = []
                
                # Summary by status
                status_summary = []
                FOR [status, tasks] IN this.data.tasks.by_status:
                    status_summary.push(`${status}: ${tasks.length}`)
                END
                lines.push(status_summary.join(" | "))
                lines.push("")
                
                # Active tasks detail
                active_tasks = this.data.tasks.tasks.filter(t => t.status == "in_progress")
                IF active_tasks.length > 0:
                    lines.push("Active Tasks:")
                    FOR task IN active_tasks:
                        agent_info = task.agent ? ` [${task.agent}]` : ""
                        lines.push(`  Task ${task.id}: ${task.current_phase}${agent_info}`)
                        lines.push(`    Progress: ${this.render_progress_bar(task.progress, 20)} ${task.progress}%`)
                        lines.push(`    Duration: ${format_duration(task.duration)}`)
                    END
                END
                
                RETURN lines
            }))
        END
        
        # Agent activity section
        IF this.data.agents:
            output.push(this.render_section("AGENT ACTIVITY", () => {
                lines = []
                
                lines.push(`Active Agents: ${this.data.agents.active_count} | Utilization: ${(this.data.agents.utilization * 100).toFixed(1)}%`)
                lines.push("")
                
                # Agents by type
                FOR [type, agents] IN this.data.agents.by_type:
                    active = agents.filter(a => a.status == "running").length
                    lines.push(`${type}: ${active}/${agents.length} active`)
                END
                
                RETURN lines
            }))
        END
        
        # Performance metrics section
        IF this.data.performance:
            output.push(this.render_section("PERFORMANCE", () => {
                lines = []
                
                lines.push(`Cache Hit Rate: ${(this.data.performance.cache_hit_rate * 100).toFixed(1)}%`)
                
                # Pool utilization
                pools = []
                FOR [pool, util] IN this.data.performance.pool_utilization:
                    pools.push(`${pool}: ${(util * 100).toFixed(1)}%`)
                END
                lines.push("Resource Pools: " + pools.join(", "))
                
                # Slow operations
                IF this.data.performance.slow_operations.length > 0:
                    lines.push("")
                    lines.push("Slow Operations:")
                    FOR op IN this.data.performance.slow_operations.slice(0, 3):
                        lines.push(`  ${op.operation}: ${op.avg_duration.toFixed(0)}ms (${op.count} calls)`)
                    END
                END
                
                RETURN lines
            }))
        END
        
        # Error tracking section
        IF this.data.errors AND this.data.errors.summary.total_errors_1h > 0:
            output.push(this.render_section("ERRORS", () => {
                lines = []
                
                lines.push(`Error Rate: ${this.data.errors.error_rate.toFixed(2)}/min | Last Hour: ${this.data.errors.summary.total_errors_1h}`)
                
                IF this.data.errors.critical_count > 0:
                    lines.push(this.colorize(`CRITICAL ERRORS: ${this.data.errors.critical_count}`, "red"))
                END
                
                # Top errors
                IF this.data.errors.top_errors.length > 0:
                    lines.push("")
                    lines.push("Top Errors:")
                    FOR error IN this.data.errors.top_errors.slice(0, 3):
                        lines.push(`  ${error.error}: ${error.count} occurrences`)
                    END
                END
                
                RETURN lines
            }))
        END
        
        # Health status section
        IF this.data.health:
            output.push(this.render_section("HEALTH STATUS", () => {
                lines = []
                
                status_line = `Overall: ${get_status_emoji(this.data.health.overall_status)} ${this.data.health.overall_status.toUpperCase()}`
                lines.push(this.colorize(status_line, this.get_health_color(this.data.health.overall_status)))
                lines.push(`Uptime: ${this.data.health.uptime_percent.toFixed(1)}%`)
                
                # Component health
                lines.push("")
                lines.push("Components:")
                FOR [component, data] IN this.data.health.components:
                    status_icon = data.status == "healthy" ? "✓" : "✗"
                    color = this.get_health_color(data.status)
                    lines.push(this.colorize(`  ${status_icon} ${component}: ${data.status}`, color))
                END
                
                # Active alerts
                IF this.data.health.alerts.length > 0:
                    lines.push("")
                    lines.push("Active Alerts:")
                    FOR alert IN this.data.health.alerts.slice(0, 3):
                        lines.push(`  ⚠️  ${alert.message}`)
                    END
                END
                
                RETURN lines
            }))
        END
        
        # Resource usage section
        IF this.data.resources:
            output.push(this.render_section("RESOURCES", () => {
                lines = []
                
                r = this.data.resources
                
                # Memory
                mem_percent = (r.memory.used / r.memory.total) * 100
                lines.push(`Memory: ${this.render_usage_bar(mem_percent)} ${mem_percent.toFixed(1)}% (${format_bytes(r.memory.used)}/${format_bytes(r.memory.total)})`)
                
                # CPU
                lines.push(`CPU: ${this.render_usage_bar(r.cpu.usage)} ${r.cpu.usage.toFixed(1)}% (${r.cpu.cores} cores)`)
                
                # Disk
                disk_percent = (r.disk.used / r.disk.total) * 100
                lines.push(`Disk: ${this.render_usage_bar(disk_percent)} ${disk_percent.toFixed(1)}% (${format_bytes(r.disk.free)} free)`)
                
                # Git worktrees
                lines.push(`Git Worktrees: ${r.git_worktrees}`)
                
                RETURN lines
            }))
        END
        
        # Footer
        output.push(this.render_footer())
        
        this.output_stream.write(output.join("\n"))
    }
    
    render_json() {
        json_output = {
            timestamp: iso_timestamp(),
            data: this.data
        }
        
        this.output_stream.write(JSON.stringify(json_output, null, 2))
    }
    
    render_header(title, mode) {
        width = mode == "compact" ? 50 : 80
        border = "═".repeat(width - 2)
        
        lines = [
            `╔${border}╗`,
            `║${center_text(title, width - 2)}║`,
            `║${center_text(format_timestamp(new Date()), width - 2)}║`,
            `╠${border}╣`
        ]
        
        RETURN lines.join("\n")
    }
    
    render_section(title, content_fn) {
        lines = [`║ ${title}`]
        lines.push("║ " + "─".repeat(title.length))
        
        content_lines = content_fn()
        FOR line IN content_lines:
            lines.push("║ " + line)
        END
        
        lines.push("║")
        RETURN lines.join("\n")
    }
    
    render_footer() {
        width = 80
        border = "═".repeat(width - 2)
        
        shortcuts = "R: Refresh | M: Mode | Q: Quit"
        
        RETURN [
            `╠${border}╣`,
            `║${center_text(shortcuts, width - 2)}║`,
            `╚${border}╝`
        ].join("\n")
    }
    
    render_progress_bar(percentage, width = 20) {
        filled = Math.round((percentage / 100) * width)
        empty = width - filled
        
        RETURN `[${this.colorize("█".repeat(filled), "green")}${"░".repeat(empty)}]`
    }
    
    render_usage_bar(percentage, width = 15) {
        filled = Math.round((percentage / 100) * width)
        empty = width - filled
        
        color = "green"
        IF percentage > 80:
            color = "red"
        ELIF percentage > 60:
            color = "yellow"
        END
        
        RETURN `[${this.colorize("▓".repeat(filled), color)}${"░".repeat(empty)}]`
    }
    
    colorize(text, color) {
        colors = {
            red: "\033[31m",
            green: "\033[32m",
            yellow: "\033[33m",
            blue: "\033[34m",
            magenta: "\033[35m",
            cyan: "\033[36m",
            white: "\033[37m",
            reset: "\033[0m"
        }
        
        RETURN `${colors[color] || ""}${text}${colors.reset}`
    }
    
    get_status_color(status) {
        CASE status:
            "active", "completed": RETURN "green"
            "failed": RETURN "red"
            "in_progress": RETURN "cyan"
            DEFAULT: RETURN "white"
        END
    }
    
    get_health_color(status) {
        CASE status:
            "healthy": RETURN "green"
            "degraded": RETURN "yellow"
            "unhealthy": RETURN "red"
            "critical": RETURN "magenta"
            DEFAULT: RETURN "white"
        END
    }
    
    clear_screen() {
        this.output_stream.write("\033[2J\033[H")
    }
    
    schedule_render() {
        # Debounce renders
        IF this.render_timeout:
            clearTimeout(this.render_timeout)
        END
        
        this.render_timeout = setTimeout(() => {
            this.render()
        }, 100)
    }
    
    async update_section(section) {
        TRY:
            method_name = `collect_${section}_data`
            IF this[method_name]:
                this.data[section] = await this[method_name]()
            END
        CATCH error:
            LOG_ERROR: `Failed to update section ${section}`, { error: error.message }
        END
    }
    
    async update_section_data(section, collector_fn) {
        TRY:
            this.data[section] = await collector_fn()
        CATCH error:
            LOG_ERROR: `Failed to update section ${section}`, { error: error.message }
        END
    }
}

# Global dashboard instance
MONITORING_DASHBOARD = null

FUNCTION start_monitoring_dashboard(options = {}) {
    IF MONITORING_DASHBOARD:
        MONITORING_DASHBOARD.stop()
    END
    
    MONITORING_DASHBOARD = new MonitoringDashboard(options)
    MONITORING_DASHBOARD.start()
    
    RETURN MONITORING_DASHBOARD
}

FUNCTION stop_monitoring_dashboard() {
    IF MONITORING_DASHBOARD:
        MONITORING_DASHBOARD.stop()
        MONITORING_DASHBOARD = null
    END
}
```

**UTILITY FUNCTIONS:**

```
FUNCTION format_duration(ms) {
    seconds = Math.floor(ms / 1000)
    minutes = Math.floor(seconds / 60)
    hours = Math.floor(minutes / 60)
    
    IF hours > 0:
        RETURN `${hours}h ${minutes % 60}m`
    ELIF minutes > 0:
        RETURN `${minutes}m ${seconds % 60}s`
    ELSE:
        RETURN `${seconds}s`
    END
}

FUNCTION format_bytes(bytes) {
    units = ["B", "KB", "MB", "GB", "TB"]
    index = 0
    
    WHILE bytes >= 1024 AND index < units.length - 1:
        bytes /= 1024
        index++
    END
    
    RETURN `${bytes.toFixed(1)} ${units[index]}`
}

FUNCTION format_timestamp(date) {
    RETURN date.toLocaleString()
}

FUNCTION center_text(text, width) {
    padding = Math.max(0, width - text.length)
    left = Math.floor(padding / 2)
    right = padding - left
    
    RETURN " ".repeat(left) + text + " ".repeat(right)
}

FUNCTION group_by(array, key) {
    grouped = {}
    
    FOR item IN array:
        value = item[key]
        IF not grouped[value]:
            grouped[value] = []
        END
        grouped[value].push(item)
    END
    
    RETURN grouped
}

FUNCTION calculate_agent_utilization(agents) {
    IF agents.length == 0:
        RETURN 0
    END
    
    active = agents.filter(a => a.status == "running").length
    RETURN active / agents.length
}

FUNCTION get_terminal_height() {
    RETURN process.stdout.rows || 24
}

FUNCTION get_all_agent_states() {
    # Mock implementation - would query actual agent states
    RETURN []
}

FUNCTION count_git_worktrees() {
    # Count .worktrees subdirectories
    RETURN list_directories(".worktrees").length
}
```

**INTERACTIVE MODE:**

```
FUNCTION enable_interactive_mode(dashboard) {
    readline = require("readline")
    
    # Set up key handler
    readline.emitKeypressEvents(process.stdin)
    process.stdin.setRawMode(true)
    
    process.stdin.on("keypress", (str, key) => {
        IF key.ctrl AND key.name == "c":
            # Exit
            dashboard.stop()
            process.exit(0)
        END
        
        CASE key.name:
            "r":
                # Force refresh
                dashboard.collect_all_data()
                dashboard.render()
                
            "m":
                # Toggle mode
                modes = ["compact", "detailed", "json"]
                current_index = modes.indexOf(dashboard.mode)
                dashboard.mode = modes[(current_index + 1) % modes.length]
                dashboard.render()
                
            "q":
                # Quit
                dashboard.stop()
                process.exit(0)
                
            "h":
                # Show help
                show_interactive_help()
        END
    })
}

FUNCTION show_interactive_help() {
    help_text = """
    
    MONITORING DASHBOARD CONTROLS
    ============================
    
    R - Refresh display
    M - Toggle display mode (compact/detailed/json)
    H - Show this help
    Q - Quit dashboard
    
    Ctrl+C - Force quit
    
    Press any key to continue...
    """
    
    console.log(help_text)
}
```

**EXPORTS:**
- `start_monitoring_dashboard(options)` -> Start monitoring dashboard
- `stop_monitoring_dashboard()` -> Stop monitoring dashboard
- `MonitoringDashboard` -> Dashboard class
- `enable_interactive_mode(dashboard)` -> Enable keyboard controls