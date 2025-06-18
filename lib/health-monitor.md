**HEALTH MONITOR MODULE**

Provides system health monitoring, threshold alerts, and automatic recovery mechanisms.

**CORE CONCEPTS:**
- Multi-level health checks (system, application, dependencies)
- Configurable thresholds and alert conditions
- Automatic recovery actions
- Health score calculation

**HEALTH CHECK STRUCTURE:**
```
HEALTH_CHECKS = {
    system: {
        cpu_usage: { threshold: 80, critical: 95 },
        memory_usage: { threshold: 75, critical: 90 },
        disk_space: { threshold: 80, critical: 95 }
    },
    application: {
        error_rate: { threshold: 5, critical: 10 },  # per minute
        response_time: { threshold: 1000, critical: 5000 },  # ms
        queue_size: { threshold: 100, critical: 500 }
    },
    dependencies: {
        git_connectivity: { required: true },
        api_availability: { required: false },
        task_server: { required: true }
    }
}

HEALTH_STATES = {
    HEALTHY: "healthy",
    WARNING: "warning", 
    CRITICAL: "critical",
    UNKNOWN: "unknown"
}
```

**MAIN FUNCTIONS:**

```
FUNCTION init_health_monitor(options = {}):
    monitor = {
        checks: options.checks || HEALTH_CHECKS,
        interval: options.interval || 30000,  # 30 seconds
        history_size: options.history_size || 100,
        recovery_enabled: options.recovery_enabled !== false,
        current_status: {},
        history: []
    }
    
    start_monitoring(monitor)
    RETURN monitor
END

FUNCTION perform_health_check(monitor):
    results = {
        timestamp: iso_timestamp(),
        checks: {},
        overall_state: HEALTH_STATES.HEALTHY,
        score: 100
    }
    
    # System checks
    results.checks.system = check_system_resources()
    
    # Application checks
    results.checks.application = check_application_health()
    
    # Dependency checks
    results.checks.dependencies = check_dependencies()
    
    # Calculate overall health
    results.overall_state = calculate_overall_state(results.checks)
    results.score = calculate_health_score(results.checks)
    
    # Update monitor
    monitor.current_status = results
    monitor.history.push(results)
    if monitor.history.length > monitor.history_size:
        monitor.history.shift()
    
    # Trigger recovery if needed
    IF monitor.recovery_enabled AND results.overall_state == HEALTH_STATES.CRITICAL:
        trigger_recovery_actions(results)
    END
    
    emit("health.checked", results)
    RETURN results
END

FUNCTION check_system_resources():
    RETURN {
        cpu: {
            value: get_cpu_usage(),
            state: evaluate_threshold(get_cpu_usage(), HEALTH_CHECKS.system.cpu_usage)
        },
        memory: {
            value: get_memory_usage(),
            state: evaluate_threshold(get_memory_usage(), HEALTH_CHECKS.system.memory_usage)
        },
        disk: {
            value: get_disk_usage(),
            state: evaluate_threshold(get_disk_usage(), HEALTH_CHECKS.system.disk_space)
        }
    }
END

FUNCTION trigger_recovery_actions(health_results):
    actions = []
    
    # High CPU - reduce concurrency
    IF health_results.checks.system.cpu.state == HEALTH_STATES.CRITICAL:
        actions.push({
            type: "reduce_concurrency",
            execute: () => reduce_max_agents()
        })
    END
    
    # High memory - clear caches
    IF health_results.checks.system.memory.state == HEALTH_STATES.CRITICAL:
        actions.push({
            type: "clear_caches",
            execute: () => clear_all_caches()
        })
    END
    
    # High error rate - enable circuit breakers
    IF health_results.checks.application.error_rate.state == HEALTH_STATES.CRITICAL:
        actions.push({
            type: "enable_circuit_breakers",
            execute: () => enable_all_circuit_breakers()
        })
    END
    
    FOR action IN actions:
        emit("health.recovery_action", action)
        action.execute()
    END
END

FUNCTION calculate_health_score(checks):
    weights = {
        system: 0.4,
        application: 0.4,
        dependencies: 0.2
    }
    
    score = 0
    FOR category, category_checks IN checks:
        category_score = 0
        check_count = 0
        
        FOR check_name, check_result IN category_checks:
            check_count++
            IF check_result.state == HEALTH_STATES.HEALTHY:
                category_score += 100
            ELIF check_result.state == HEALTH_STATES.WARNING:
                category_score += 50
            # CRITICAL adds 0
        END
        
        score += (category_score / check_count) * weights[category]
    END
    
    RETURN Math.round(score)
END

FUNCTION get_health_trends(monitor, duration = 300000):  # 5 minutes
    cutoff_time = Date.now() - duration
    relevant_history = monitor.history.filter(h => 
        Date.parse(h.timestamp) > cutoff_time
    )
    
    RETURN {
        score_trend: calculate_trend(relevant_history.map(h => h.score)),
        error_trend: calculate_trend(relevant_history.map(h => 
            h.checks.application.error_rate.value
        )),
        resource_trend: calculate_trend(relevant_history.map(h => ({
            cpu: h.checks.system.cpu.value,
            memory: h.checks.system.memory.value
        })))
    }
END
```

**EXPORTS:**
- init_health_monitor(options) -> Monitor instance
- perform_health_check(monitor) -> Health check results
- check_system_resources() -> System resource status
- check_application_health() -> Application health status
- check_dependencies() -> Dependency availability
- trigger_recovery_actions(results) -> Execute recovery
- calculate_health_score(checks) -> Overall health score (0-100)
- get_health_trends(monitor, duration) -> Trend analysis
- get_current_health(monitor) -> Current health status