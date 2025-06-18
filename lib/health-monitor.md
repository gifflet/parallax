**HEALTH MONITORING MODULE**

Provides comprehensive health checks and monitoring for all system components.

**PURPOSE:** Monitor system health, detect issues early, and provide observability

**IMPORTS:**
```
- ./event-bus.md            # For health events
- ./distributed-state.md    # For state health checks
- ./performance-optimizer.md # For performance metrics
- ./error-handler.md        # For error metrics
```

**HEALTH CHECK TYPES:**
```
HEALTH_STATUS = {
    HEALTHY: "healthy",
    DEGRADED: "degraded",
    UNHEALTHY: "unhealthy",
    CRITICAL: "critical"
}

HEALTH_CHECKS = {
    system: ["memory", "disk", "cpu", "processes"],
    state: ["integrity", "locks", "consistency"],
    agents: ["running", "stuck", "resources"],
    git: ["worktrees", "branches", "remotes"],
    network: ["connectivity", "latency"],
    dependencies: ["installed", "versions"]
}
```

**CORE HEALTH MONITOR:**

```
CLASS HealthMonitor {
    constructor(options = {}) {
        this.check_interval = options.check_interval || 60000  # 1 minute
        this.history_size = options.history_size || 100
        this.thresholds = {
            memory_usage_percent: 80,
            disk_usage_percent: 90,
            cpu_usage_percent: 85,
            error_rate_per_minute: 10,
            response_time_ms: 5000,
            ...options.thresholds
        }
        
        this.checks = new Map()
        this.history = []
        this.status = HEALTH_STATUS.HEALTHY
        this.monitoring = false
    }
    
    register_check(name, check_fn, options = {}) {
        this.checks.set(name, {
            fn: check_fn,
            timeout: options.timeout || 5000,
            critical: options.critical || false,
            enabled: options.enabled !== false,
            last_result: null,
            last_check: null
        })
    }
    
    async start() {
        IF this.monitoring:
            RETURN
        END
        
        this.monitoring = true
        
        # Initial check
        await this.run_all_checks()
        
        # Schedule periodic checks
        this.interval = setInterval(async () => {
            await this.run_all_checks()
        }, this.check_interval)
        
        emit("health.monitoring_started")
    }
    
    stop() {
        IF this.interval:
            clearInterval(this.interval)
        END
        this.monitoring = false
        emit("health.monitoring_stopped")
    }
    
    async run_all_checks() {
        results = {
            timestamp: iso_timestamp(),
            checks: {},
            overall_status: HEALTH_STATUS.HEALTHY,
            issues: []
        }
        
        # Run checks in parallel
        check_promises = []
        FOR [name, check] IN this.checks:
            IF check.enabled:
                check_promises.push(
                    this.run_single_check(name, check)
                        .then(result => ({ name, result }))
                )
            END
        END
        
        check_results = await Promise.allSettled(check_promises)
        
        # Process results
        FOR result IN check_results:
            IF result.status == "fulfilled":
                check_name = result.value.name
                check_result = result.value.result
                
                results.checks[check_name] = check_result
                
                # Update overall status
                IF check_result.status == HEALTH_STATUS.CRITICAL:
                    results.overall_status = HEALTH_STATUS.CRITICAL
                ELIF check_result.status == HEALTH_STATUS.UNHEALTHY AND 
                     results.overall_status != HEALTH_STATUS.CRITICAL:
                    results.overall_status = HEALTH_STATUS.UNHEALTHY
                ELIF check_result.status == HEALTH_STATUS.DEGRADED AND
                     results.overall_status == HEALTH_STATUS.HEALTHY:
                    results.overall_status = HEALTH_STATUS.DEGRADED
                END
                
                # Collect issues
                IF check_result.issues:
                    results.issues.push(...check_result.issues)
                END
            ELSE:
                # Check failed
                failed_name = result.reason.name || "unknown"
                results.checks[failed_name] = {
                    status: HEALTH_STATUS.UNHEALTHY,
                    error: result.reason.message,
                    timestamp: iso_timestamp()
                }
                results.issues.push({
                    check: failed_name,
                    message: "Health check failed: " + result.reason.message
                })
            END
        END
        
        # Update history
        this.history.push(results)
        IF this.history.length > this.history_size:
            this.history.shift()
        END
        
        # Update monitor status
        previous_status = this.status
        this.status = results.overall_status
        
        # Emit events
        emit("health.check_completed", results)
        
        IF this.status != previous_status:
            emit("health.status_changed", {
                previous: previous_status,
                current: this.status,
                issues: results.issues
            })
        END
        
        RETURN results
    }
    
    async run_single_check(name, check) {
        start_time = Date.now()
        
        TRY:
            result = await with_timeout(check.fn(), check.timeout)
            
            check.last_result = result
            check.last_check = iso_timestamp()
            
            RETURN {
                ...result,
                duration_ms: Date.now() - start_time,
                timestamp: iso_timestamp()
            }
        CATCH error:
            RETURN {
                status: check.critical ? HEALTH_STATUS.CRITICAL : HEALTH_STATUS.UNHEALTHY,
                error: error.message,
                duration_ms: Date.now() - start_time,
                timestamp: iso_timestamp()
            }
        END
    }
    
    get_status() {
        RETURN {
            status: this.status,
            monitoring: this.monitoring,
            last_check: this.history.length > 0 ? 
                this.history[this.history.length - 1].timestamp : null,
            check_count: this.checks.size,
            issues: this.get_current_issues()
        }
    }
    
    get_current_issues() {
        IF this.history.length == 0:
            RETURN []
        END
        
        RETURN this.history[this.history.length - 1].issues
    }
    
    get_health_report() {
        recent_history = this.history.slice(-10)
        
        report = {
            current_status: this.status,
            uptime: calculate_uptime(this.history),
            availability: calculate_availability(this.history),
            checks: {},
            trends: {},
            recommendations: []
        }
        
        # Aggregate check results
        FOR [name, check] IN this.checks:
            check_history = recent_history
                .map(h => h.checks[name])
                .filter(Boolean)
            
            report.checks[name] = {
                current_status: check.last_result?.status || "unknown",
                last_check: check.last_check,
                success_rate: calculate_success_rate(check_history),
                avg_duration: calculate_avg_duration(check_history)
            }
        END
        
        # Calculate trends
        report.trends = calculate_health_trends(this.history)
        
        # Generate recommendations
        report.recommendations = generate_health_recommendations(report)
        
        RETURN report
    }
}

# Global health monitor instance
HEALTH_MONITOR = new HealthMonitor()
```

**BUILT-IN HEALTH CHECKS:**

```
# System resource checks
HEALTH_MONITOR.register_check("system.memory", async () => {
    memory_info = get_memory_info()
    usage_percent = (memory_info.used / memory_info.total) * 100
    
    status = HEALTH_STATUS.HEALTHY
    issues = []
    
    IF usage_percent > HEALTH_MONITOR.thresholds.memory_usage_percent:
        status = HEALTH_STATUS.UNHEALTHY
        issues.push({
            type: "high_memory_usage",
            message: `Memory usage at ${usage_percent.toFixed(1)}%`,
            severity: "high"
        })
    ELIF usage_percent > HEALTH_MONITOR.thresholds.memory_usage_percent * 0.8:
        status = HEALTH_STATUS.DEGRADED
    END
    
    RETURN {
        status: status,
        metrics: {
            usage_percent: usage_percent,
            used_mb: memory_info.used / 1024 / 1024,
            total_mb: memory_info.total / 1024 / 1024
        },
        issues: issues
    }
})

HEALTH_MONITOR.register_check("system.disk", async () => {
    disk_info = await get_disk_info()
    usage_percent = (disk_info.used / disk_info.total) * 100
    
    status = HEALTH_STATUS.HEALTHY
    issues = []
    
    IF usage_percent > HEALTH_MONITOR.thresholds.disk_usage_percent:
        status = HEALTH_STATUS.CRITICAL
        issues.push({
            type: "critical_disk_space",
            message: `Disk usage at ${usage_percent.toFixed(1)}%`,
            severity: "critical"
        })
    ELIF usage_percent > HEALTH_MONITOR.thresholds.disk_usage_percent * 0.8:
        status = HEALTH_STATUS.UNHEALTHY
    ELIF usage_percent > HEALTH_MONITOR.thresholds.disk_usage_percent * 0.6:
        status = HEALTH_STATUS.DEGRADED
    END
    
    RETURN {
        status: status,
        metrics: {
            usage_percent: usage_percent,
            free_gb: disk_info.free / 1024 / 1024 / 1024,
            total_gb: disk_info.total / 1024 / 1024 / 1024
        },
        issues: issues
    }
}, { critical: true })

# State health checks
HEALTH_MONITOR.register_check("state.integrity", async () => {
    state_issues = await verify_all_states()
    
    status = HEALTH_STATUS.HEALTHY
    IF state_issues.length > 0:
        status = HEALTH_STATUS.UNHEALTHY
    END
    
    RETURN {
        status: status,
        metrics: {
            corrupted_states: state_issues.length,
            total_states: await count_all_states()
        },
        issues: state_issues.map(issue => ({
            type: "state_corruption",
            message: `State corrupted for ${issue.entity_id}`,
            severity: "high"
        }))
    }
})

HEALTH_MONITOR.register_check("state.locks", async () => {
    lock_info = await get_lock_statistics()
    
    status = HEALTH_STATUS.HEALTHY
    issues = []
    
    IF lock_info.expired_locks > 0:
        status = HEALTH_STATUS.DEGRADED
        issues.push({
            type: "expired_locks",
            message: `${lock_info.expired_locks} expired locks found`,
            severity: "medium"
        })
    END
    
    IF lock_info.stuck_locks > 0:
        status = HEALTH_STATUS.UNHEALTHY
        issues.push({
            type: "stuck_locks",
            message: `${lock_info.stuck_locks} locks held too long`,
            severity: "high"
        })
    END
    
    RETURN {
        status: status,
        metrics: lock_info,
        issues: issues
    }
})

# Agent health checks
HEALTH_MONITOR.register_check("agents.status", async () => {
    agent_states = await get_all_agent_states()
    
    status = HEALTH_STATUS.HEALTHY
    issues = []
    metrics = {
        total: 0,
        running: 0,
        stuck: 0,
        failed: 0
    }
    
    FOR agent IN agent_states:
        metrics.total++
        
        IF agent.status == "running":
            metrics.running++
            
            # Check if stuck
            IF Date.now() - agent.last_progress > 300000:  # 5 minutes
                metrics.stuck++
                issues.push({
                    type: "stuck_agent",
                    message: `Agent ${agent.id} stuck for > 5 minutes`,
                    severity: "high"
                })
            END
        ELIF agent.status == "failed":
            metrics.failed++
        END
    END
    
    IF metrics.stuck > 0:
        status = HEALTH_STATUS.UNHEALTHY
    ELIF metrics.failed > metrics.total * 0.2:
        status = HEALTH_STATUS.DEGRADED
    END
    
    RETURN {
        status: status,
        metrics: metrics,
        issues: issues
    }
})

# Performance health checks
HEALTH_MONITOR.register_check("performance.response_time", async () => {
    perf_report = get_performance_report()
    
    status = HEALTH_STATUS.HEALTHY
    issues = []
    slow_operations = []
    
    FOR [op_name, stats] IN perf_report.operations:
        IF stats.avg_duration > HEALTH_MONITOR.thresholds.response_time_ms:
            slow_operations.push({
                operation: op_name,
                avg_duration: stats.avg_duration
            })
        END
    END
    
    IF slow_operations.length > 0:
        status = HEALTH_STATUS.DEGRADED
        issues.push({
            type: "slow_operations",
            message: `${slow_operations.length} operations exceeding threshold`,
            severity: "medium",
            details: slow_operations
        })
    END
    
    RETURN {
        status: status,
        metrics: {
            slow_operation_count: slow_operations.length,
            cache_hit_rate: perf_report.summary.cache_hit_rate
        },
        issues: issues
    }
})

# Error rate health check
HEALTH_MONITOR.register_check("errors.rate", async () => {
    error_dashboard = get_error_dashboard_data()
    error_rate = error_dashboard.summary.error_rate_per_minute
    
    status = HEALTH_STATUS.HEALTHY
    issues = []
    
    IF error_rate > HEALTH_MONITOR.thresholds.error_rate_per_minute:
        status = HEALTH_STATUS.UNHEALTHY
        issues.push({
            type: "high_error_rate",
            message: `Error rate: ${error_rate.toFixed(2)}/min`,
            severity: "high"
        })
    ELIF error_rate > HEALTH_MONITOR.thresholds.error_rate_per_minute * 0.5:
        status = HEALTH_STATUS.DEGRADED
    END
    
    IF error_dashboard.summary.critical_errors > 0:
        status = HEALTH_STATUS.CRITICAL
        issues.push({
            type: "critical_errors",
            message: `${error_dashboard.summary.critical_errors} critical errors`,
            severity: "critical"
        })
    END
    
    RETURN {
        status: status,
        metrics: {
            error_rate_per_minute: error_rate,
            total_errors_1h: error_dashboard.summary.total_errors_1h,
            critical_errors: error_dashboard.summary.critical_errors
        },
        issues: issues
    }
}, { critical: true })
```

**HEALTH DASHBOARD:**

```
FUNCTION get_health_dashboard() {
    current_status = HEALTH_MONITOR.get_status()
    health_report = HEALTH_MONITOR.get_health_report()
    
    dashboard = {
        overview: {
            status: current_status.status,
            status_emoji: get_status_emoji(current_status.status),
            monitoring: current_status.monitoring,
            last_check: current_status.last_check,
            uptime_percent: health_report.uptime,
            issue_count: current_status.issues.length
        },
        components: {},
        metrics: {
            system: {},
            performance: {},
            errors: {}
        },
        alerts: [],
        recommendations: health_report.recommendations
    }
    
    # Component health
    FOR [check_name, check_data] IN health_report.checks:
        component = check_name.split(".")[0]
        IF not dashboard.components[component]:
            dashboard.components[component] = {
                status: HEALTH_STATUS.HEALTHY,
                checks: {}
            }
        END
        
        dashboard.components[component].checks[check_name] = check_data
        
        # Update component status
        IF check_data.current_status == HEALTH_STATUS.CRITICAL:
            dashboard.components[component].status = HEALTH_STATUS.CRITICAL
        ELIF check_data.current_status == HEALTH_STATUS.UNHEALTHY AND
             dashboard.components[component].status != HEALTH_STATUS.CRITICAL:
            dashboard.components[component].status = HEALTH_STATUS.UNHEALTHY
        END
    END
    
    # Active alerts
    FOR issue IN current_status.issues:
        IF issue.severity IN ["critical", "high"]:
            dashboard.alerts.push({
                type: issue.type,
                message: issue.message,
                severity: issue.severity,
                timestamp: iso_timestamp()
            })
        END
    END
    
    RETURN dashboard
END

FUNCTION render_health_status():
    dashboard = get_health_dashboard()
    
    status_colors = {
        healthy: "\033[32m",    # Green
        degraded: "\033[33m",   # Yellow
        unhealthy: "\033[31m",  # Red
        critical: "\033[35m"    # Magenta
    }
    
    reset = "\033[0m"
    
    DISPLAY: """
╔════════════════════════════════════════════════════════════════╗
║                      SYSTEM HEALTH STATUS                      ║
╠════════════════════════════════════════════════════════════════╣
║ Overall: {status_colors[dashboard.overview.status]}{dashboard.overview.status_emoji} {dashboard.overview.status.toUpperCase()}{reset}
║ Uptime: {dashboard.overview.uptime_percent}% | Last Check: {dashboard.overview.last_check}
║
║ COMPONENTS:
"""
    
    FOR [component, data] IN dashboard.components:
        color = status_colors[data.status]
        DISPLAY: `║ ${color}▣${reset} ${component}: ${data.status}`
        
        FOR [check, result] IN data.checks:
            IF result.current_status != HEALTH_STATUS.HEALTHY:
                DISPLAY: `║   └─ ${check}: ${result.current_status}`
            END
        END
    END
    
    IF dashboard.alerts.length > 0:
        DISPLAY: "║\n║ ACTIVE ALERTS:"
        FOR alert IN dashboard.alerts:
            DISPLAY: `║ ⚠️  ${alert.message}`
        END
    END
    
    IF dashboard.recommendations.length > 0:
        DISPLAY: "║\n║ RECOMMENDATIONS:"
        FOR rec IN dashboard.recommendations:
            DISPLAY: `║ 💡 ${rec}`
        END
    END
    
    DISPLAY: "╚════════════════════════════════════════════════════════════════╝"
END

FUNCTION get_status_emoji(status):
    CASE status:
        HEALTH_STATUS.HEALTHY: RETURN "✅"
        HEALTH_STATUS.DEGRADED: RETURN "⚡"
        HEALTH_STATUS.UNHEALTHY: RETURN "❌"
        HEALTH_STATUS.CRITICAL: RETURN "🚨"
        DEFAULT: RETURN "❓"
    END
END
```

**UTILITY FUNCTIONS:**

```
FUNCTION calculate_uptime(history):
    IF history.length == 0:
        RETURN 100
    END
    
    healthy_count = history.filter(h => 
        h.overall_status == HEALTH_STATUS.HEALTHY
    ).length
    
    RETURN (healthy_count / history.length) * 100
END

FUNCTION calculate_availability(history):
    IF history.length == 0:
        RETURN 100
    END
    
    available_count = history.filter(h => 
        h.overall_status != HEALTH_STATUS.CRITICAL
    ).length
    
    RETURN (available_count / history.length) * 100
END

FUNCTION generate_health_recommendations(report):
    recommendations = []
    
    # Memory recommendations
    IF report.checks["system.memory"]?.metrics?.usage_percent > 70:
        recommendations.push("Consider increasing memory allocation or reducing concurrent agents")
    END
    
    # Disk recommendations
    IF report.checks["system.disk"]?.metrics?.free_gb < 10:
        recommendations.push("Low disk space - run cleanup protocol to free space")
    END
    
    # Performance recommendations
    IF report.checks["performance.response_time"]?.metrics?.cache_hit_rate < 0.5:
        recommendations.push("Low cache hit rate - consider increasing cache sizes")
    END
    
    # Error recommendations
    IF report.checks["errors.rate"]?.metrics?.error_rate_per_minute > 5:
        recommendations.push("High error rate detected - review error logs and consider reducing load")
    END
    
    RETURN recommendations
END

FUNCTION auto_remediate_issues():
    current_issues = HEALTH_MONITOR.get_current_issues()
    
    FOR issue IN current_issues:
        CASE issue.type:
            "expired_locks":
                cleanup_expired_locks()
            "high_memory_usage":
                trigger_memory_cleanup()
            "critical_disk_space":
                execute_emergency_disk_cleanup()
            "stuck_agent":
                restart_stuck_agent(issue.agent_id)
        END
    END
END
```

**EXPORTS:**
- `HEALTH_MONITOR` -> Global health monitor instance
- `get_health_dashboard()` -> Get health dashboard data
- `render_health_status()` -> Display health status
- `auto_remediate_issues()` -> Attempt automatic issue resolution
- `HEALTH_STATUS` -> Health status constants