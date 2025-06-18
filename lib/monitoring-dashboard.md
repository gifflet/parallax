**MONITORING DASHBOARD MODULE**

Provides real-time monitoring dashboard with metrics, alerts, and system insights.

**CORE CONCEPTS:**
- Real-time event aggregation
- Multi-view dashboard (overview, tasks, agents, performance, errors, health)
- Configurable update intervals and display modes

**DASHBOARD STRUCTURE:**
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

**MAIN FUNCTIONS:**

```
FUNCTION init_dashboard(options = {}):
    dashboard = {
        mode: options.mode || "auto",  # auto, compact, detailed, json
        refresh_rate: options.refresh_rate || UPDATE_INTERVALS.frequent,
        enabled_sections: options.sections || Object.keys(DASHBOARD_SECTIONS),
        data: {},
        subscribers: new Map()
    }
    
    # Subscribe to relevant events
    subscribe_to_events(dashboard)
    start_update_cycle(dashboard)
    
    RETURN dashboard
END

FUNCTION update_dashboard_data(dashboard):
    # Collect data from all sources
    dashboard.data = {
        overview: get_system_overview(),
        tasks: get_task_progress(),
        agents: get_agent_status(),
        performance: get_performance_metrics(),
        errors: get_error_summary(),
        health: get_health_status(),
        resources: get_resource_usage()
    }
    
    # Notify subscribers
    emit("dashboard.updated", dashboard.data)
END

FUNCTION render_dashboard(dashboard, output_stream):
    renderer = get_renderer(dashboard.mode)
    
    FOR section IN dashboard.enabled_sections:
        IF dashboard.data[section]:
            renderer.render_section(section, dashboard.data[section], output_stream)
        END
    END
END

FUNCTION get_system_overview():
    RETURN {
        uptime: get_process_uptime(),
        active_sessions: count_active_sessions(),
        total_tasks: count_total_tasks(),
        completion_rate: calculate_completion_rate(),
        error_rate: calculate_error_rate_per_minute(),
        resource_usage: get_current_resource_usage()
    }
END

FUNCTION create_alert(condition, message, severity = "warning"):
    alert = {
        id: generate_alert_id(),
        condition: condition,
        message: message,
        severity: severity,  # info, warning, critical
        timestamp: iso_timestamp(),
        acknowledged: false
    }
    
    # Check if condition is met
    IF evaluate_condition(condition):
        emit("alert.triggered", alert)
        store_alert(alert)
    END
    
    RETURN alert
END

FUNCTION export_dashboard_data(dashboard, format = "json"):
    exporters = {
        json: () => JSON.stringify(dashboard.data, null, 2),
        csv: () => convert_to_csv(dashboard.data),
        html: () => generate_html_report(dashboard.data)
    }
    
    exporter = exporters[format]
    RETURN exporter ? exporter() : null
END
```

**ALERT CONDITIONS:**
```
COMMON_ALERTS = [
    {
        name: "high_error_rate",
        condition: (data) => data.errors.rate_per_minute > 5,
        message: "Error rate exceeds threshold",
        severity: "critical"
    },
    {
        name: "low_completion_rate", 
        condition: (data) => data.tasks.completion_rate < 0.5,
        message: "Task completion rate below 50%",
        severity: "warning"
    },
    {
        name: "resource_exhaustion",
        condition: (data) => data.resources.cpu > 90 || data.resources.memory > 90,
        message: "System resources near capacity",
        severity: "critical"
    }
]
```

**EXPORTS:**
- init_dashboard(options) -> Dashboard instance
- update_dashboard_data(dashboard) -> Updates all dashboard data
- render_dashboard(dashboard, stream) -> Renders to output stream
- get_system_overview() -> System metrics summary
- create_alert(condition, message, severity) -> Alert instance
- export_dashboard_data(dashboard, format) -> Exported data
- subscribe_to_section(dashboard, section, callback) -> Subscription
- get_dashboard_metrics(dashboard) -> All current metrics