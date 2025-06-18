**ENHANCED ERROR HANDLER MODULE**

Provides structured error handling with correlation IDs, circuit breakers, and automatic recovery.

**PURPOSE:** Improve error handling with better context, recovery mechanisms, and observability

**IMPORTS:**
```
- ./event-bus.md           # For error event notifications
- ./distributed-state.md   # For error state persistence
```

**ERROR CLASSIFICATION:**
```
ERROR_SEVERITY = {
    CRITICAL: "critical",    # System cannot continue
    HIGH: "high",           # Task will fail without intervention
    MEDIUM: "medium",       # Recoverable with retry
    LOW: "low"             # Warning/informational
}

ERROR_CATEGORIES = {
    SYSTEM: "system",
    AGENT: "agent",
    GIT: "git",
    NETWORK: "network",
    STATE: "state",
    DEPENDENCY: "dependency",
    RESOURCE: "resource",
    VALIDATION: "validation"
}
```

**STRUCTURED ERROR:**

```
CLASS StructuredError extends Error {
    constructor(message, options = {}) {
        super(message)
        this.name = "StructuredError"
        this.id = generate_error_id()
        this.timestamp = iso_timestamp()
        this.category = options.category || ERROR_CATEGORIES.SYSTEM
        this.severity = options.severity || ERROR_SEVERITY.MEDIUM
        this.code = options.code || "UNKNOWN"
        this.context = options.context || {}
        this.correlation_id = options.correlation_id || null
        this.retry_count = options.retry_count || 0
        this.max_retries = options.max_retries || 3
        this.recoverable = options.recoverable !== false
        this.stack_trace = this.stack
        this.cause = options.cause || null
    }
    
    to_json() {
        RETURN {
            id: this.id,
            name: this.name,
            message: this.message,
            timestamp: this.timestamp,
            category: this.category,
            severity: this.severity,
            code: this.code,
            context: this.context,
            correlation_id: this.correlation_id,
            retry_count: this.retry_count,
            recoverable: this.recoverable,
            stack_trace: this.stack_trace,
            cause: this.cause ? this.cause.to_json() : null
        }
    }
}

FUNCTION create_error(message, options = {}) {
    RETURN new StructuredError(message, options)
}
```

**ERROR CONTEXT ENRICHMENT:**

```
# Thread-local context storage
CONTEXT_STACK = []

FUNCTION with_error_context(context, fn) {
    CONTEXT_STACK.push(context)
    
    TRY:
        result = fn()
        RETURN result
    FINALLY:
        CONTEXT_STACK.pop()
    END
END

FUNCTION get_current_context() {
    combined_context = {}
    
    FOR context IN CONTEXT_STACK:
        combined_context = { ...combined_context, ...context }
    END
    
    # Add system context
    combined_context.system = {
        memory_usage: get_memory_usage(),
        cpu_usage: get_cpu_usage(),
        uptime: get_process_uptime()
    }
    
    RETURN combined_context
END

FUNCTION enrich_error(error) {
    IF error instanceof StructuredError:
        error.context = { ...error.context, ...get_current_context() }
    ELSE:
        # Convert to structured error
        error = create_error(error.message || String(error), {
            context: get_current_context(),
            cause: error
        })
    END
    
    RETURN error
END
```

**CIRCUIT BREAKER:**

```
CIRCUIT_BREAKERS = {}

CLASS CircuitBreaker {
    constructor(name, options = {}) {
        this.name = name
        this.failure_threshold = options.failure_threshold || 5
        this.success_threshold = options.success_threshold || 2
        this.timeout = options.timeout || 60000
        this.reset_timeout = options.reset_timeout || 30000
        
        this.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
        this.failure_count = 0
        this.success_count = 0
        this.last_failure_time = null
        this.next_attempt_time = null
    }
    
    async execute(fn, fallback = null) {
        IF this.state == "OPEN":
            IF Date.now() < this.next_attempt_time:
                IF fallback:
                    RETURN fallback()
                END
                THROW create_error("Circuit breaker is OPEN", {
                    code: "CIRCUIT_OPEN",
                    context: { circuit: this.name }
                })
            ELSE:
                this.state = "HALF_OPEN"
            END
        END
        
        TRY:
            result = await with_timeout(fn(), this.timeout)
            this.on_success()
            RETURN result
        CATCH error:
            this.on_failure()
            
            IF this.state == "OPEN" AND fallback:
                RETURN fallback()
            END
            
            THROW error
        END
    }
    
    on_success() {
        this.failure_count = 0
        
        IF this.state == "HALF_OPEN":
            this.success_count++
            IF this.success_count >= this.success_threshold:
                this.state = "CLOSED"
                this.success_count = 0
                emit("circuit_breaker.closed", { name: this.name })
            END
        END
    }
    
    on_failure() {
        this.failure_count++
        this.last_failure_time = Date.now()
        
        IF this.state == "HALF_OPEN":
            this.state = "OPEN"
            this.next_attempt_time = Date.now() + this.reset_timeout
            emit("circuit_breaker.opened", { name: this.name })
        ELIF this.failure_count >= this.failure_threshold:
            this.state = "OPEN"
            this.next_attempt_time = Date.now() + this.reset_timeout
            emit("circuit_breaker.opened", { name: this.name })
        END
    }
    
    get_status() {
        RETURN {
            name: this.name,
            state: this.state,
            failure_count: this.failure_count,
            last_failure_time: this.last_failure_time,
            next_attempt_time: this.next_attempt_time
        }
    }
}

FUNCTION get_circuit_breaker(name, options = {}) {
    IF not CIRCUIT_BREAKERS[name]:
        CIRCUIT_BREAKERS[name] = new CircuitBreaker(name, options)
    END
    
    RETURN CIRCUIT_BREAKERS[name]
END
```

**ERROR RECOVERY STRATEGIES:**

```
RECOVERY_STRATEGIES = {
    exponential_backoff: {
        execute: async (fn, error, options = {}) => {
            max_retries = options.max_retries || 3
            base_delay = options.base_delay || 1000
            max_delay = options.max_delay || 30000
            
            FOR attempt IN range(1, max_retries + 1):
                TRY:
                    RETURN await fn()
                CATCH retry_error:
                    IF attempt == max_retries:
                        THROW retry_error
                    END
                    
                    delay = Math.min(base_delay * Math.pow(2, attempt - 1), max_delay)
                    delay += Math.random() * delay * 0.1  # Add jitter
                    
                    LOG_INFO: "Retrying after delay", {
                        attempt: attempt,
                        delay: delay,
                        error: retry_error.message
                    }
                    
                    await sleep(delay)
                END
            END
        }
    },
    
    circuit_breaker: {
        execute: async (fn, error, options = {}) => {
            breaker = get_circuit_breaker(options.breaker_name || "default", options)
            RETURN await breaker.execute(fn, options.fallback)
        }
    },
    
    fallback: {
        execute: async (fn, error, options = {}) => {
            TRY:
                RETURN await fn()
            CATCH:
                IF options.fallback:
                    RETURN options.fallback()
                END
                THROW error
            END
        }
    },
    
    retry_with_cleanup: {
        execute: async (fn, error, options = {}) => {
            FOR attempt IN range(1, options.max_retries + 1):
                TRY:
                    # Cleanup before retry
                    IF options.cleanup:
                        await options.cleanup()
                    END
                    
                    RETURN await fn()
                CATCH retry_error:
                    IF attempt == max_retries:
                        THROW retry_error
                    END
                END
            END
        }
    }
}

FUNCTION with_recovery(fn, strategy_name, options = {}) {
    strategy = RECOVERY_STRATEGIES[strategy_name]
    
    IF not strategy:
        THROW create_error("Unknown recovery strategy: " + strategy_name)
    END
    
    RETURN async (...args) => {
        TRY:
            RETURN await fn(...args)
        CATCH error:
            enriched_error = enrich_error(error)
            
            IF enriched_error.recoverable:
                RETURN await strategy.execute(
                    () => fn(...args),
                    enriched_error,
                    options
                )
            END
            
            THROW enriched_error
        END
    }
END
```

**ERROR AGGREGATION AND REPORTING:**

```
ERROR_AGGREGATOR = {
    errors: [],
    max_errors: 1000,
    aggregation_window: 60000  # 1 minute
}

FUNCTION aggregate_error(error) {
    structured_error = enrich_error(error)
    
    ERROR_AGGREGATOR.errors.push({
        error: structured_error.to_json(),
        timestamp: Date.now()
    })
    
    # Cleanup old errors
    cutoff_time = Date.now() - ERROR_AGGREGATOR.aggregation_window
    ERROR_AGGREGATOR.errors = ERROR_AGGREGATOR.errors.filter(e => e.timestamp > cutoff_time)
    
    # Limit size
    IF ERROR_AGGREGATOR.errors.length > ERROR_AGGREGATOR.max_errors:
        ERROR_AGGREGATOR.errors = ERROR_AGGREGATOR.errors.slice(-ERROR_AGGREGATOR.max_errors)
    END
    
    # Check for error patterns
    detect_error_patterns()
END

FUNCTION detect_error_patterns() {
    patterns = {
        repeated_errors: {},
        error_spike: false,
        critical_errors: []
    }
    
    # Count error frequencies
    FOR error_entry IN ERROR_AGGREGATOR.errors:
        error = error_entry.error
        key = error.code + ":" + error.category
        
        IF not patterns.repeated_errors[key]:
            patterns.repeated_errors[key] = 0
        END
        patterns.repeated_errors[key]++
        
        IF error.severity == ERROR_SEVERITY.CRITICAL:
            patterns.critical_errors.push(error)
        END
    END
    
    # Detect spikes
    error_rate = ERROR_AGGREGATOR.errors.length / (ERROR_AGGREGATOR.aggregation_window / 1000)
    patterns.error_spike = error_rate > 1  # More than 1 error per second
    
    # Alert on patterns
    FOR pattern, count IN patterns.repeated_errors:
        IF count > 5:
            emit("error.pattern_detected", {
                pattern: pattern,
                count: count,
                window: ERROR_AGGREGATOR.aggregation_window
            })
        END
    END
    
    IF patterns.error_spike:
        emit("error.spike_detected", {
            rate: error_rate,
            count: ERROR_AGGREGATOR.errors.length
        })
    END
    
    IF patterns.critical_errors.length > 0:
        emit("error.critical_detected", {
            errors: patterns.critical_errors
        })
    END
END
```

**ERROR LOGGING:**

```
FUNCTION log_error(error, additional_context = {}) {
    structured_error = enrich_error(error)
    
    log_entry = {
        timestamp: iso_timestamp(),
        level: "ERROR",
        error_id: structured_error.id,
        correlation_id: structured_error.correlation_id,
        message: structured_error.message,
        category: structured_error.category,
        severity: structured_error.severity,
        code: structured_error.code,
        context: { ...structured_error.context, ...additional_context },
        stack_trace: structured_error.stack_trace
    }
    
    # Write to log file
    append_to_log_file(log_entry)
    
    # Aggregate for pattern detection
    aggregate_error(structured_error)
    
    # Emit error event
    emit("error.logged", {
        error_id: structured_error.id,
        severity: structured_error.severity,
        category: structured_error.category
    })
    
    # Store in distributed state for persistence
    IF structured_error.severity IN [ERROR_SEVERITY.CRITICAL, ERROR_SEVERITY.HIGH]:
        store_error_state(structured_error)
    END
    
    RETURN structured_error
END

FUNCTION append_to_log_file(log_entry) {
    log_file = ".worktrees/.logs/errors.jsonl"
    CREATE_DIR_IF_NOT_EXISTS(".worktrees/.logs")
    
    # Rotate log if too large
    IF get_file_size(log_file) > 50 * 1024 * 1024:  # 50MB
        rotate_log_file(log_file)
    END
    
    # Append as JSON line
    append_file(log_file, JSON.stringify(log_entry) + "\n")
END
```

**AUTOMATIC ROLLBACK:**

```
ROLLBACK_REGISTRY = {}

FUNCTION register_rollback(operation_id, rollback_fn, context = {}) {
    ROLLBACK_REGISTRY[operation_id] = {
        fn: rollback_fn,
        context: context,
        registered_at: iso_timestamp()
    }
END

FUNCTION with_rollback(operation_id, operation_fn, rollback_fn) {
    RETURN async (...args) => {
        register_rollback(operation_id, rollback_fn, { args: args })
        
        TRY:
            result = await operation_fn(...args)
            # Success - remove rollback
            delete ROLLBACK_REGISTRY[operation_id]
            RETURN result
        CATCH error:
            LOG_ERROR: "Operation failed, executing rollback", {
                operation_id: operation_id,
                error: error.message
            }
            
            TRY:
                await rollback_fn(...args)
                LOG_INFO: "Rollback completed successfully", {
                    operation_id: operation_id
                }
            CATCH rollback_error:
                LOG_ERROR: "Rollback failed", {
                    operation_id: operation_id,
                    error: rollback_error.message
                }
                
                # Create compound error
                THROW create_error("Operation failed and rollback failed", {
                    severity: ERROR_SEVERITY.CRITICAL,
                    cause: error,
                    context: {
                        rollback_error: rollback_error.message
                    }
                })
            END
            
            THROW error
        END
    }
END

FUNCTION execute_all_rollbacks(filter = null) {
    rollbacks = Object.entries(ROLLBACK_REGISTRY)
    
    IF filter:
        rollbacks = rollbacks.filter(filter)
    END
    
    results = []
    FOR [operation_id, rollback] IN rollbacks:
        TRY:
            await rollback.fn()
            results.push({
                operation_id: operation_id,
                status: "success"
            })
            delete ROLLBACK_REGISTRY[operation_id]
        CATCH error:
            results.push({
                operation_id: operation_id,
                status: "failed",
                error: error.message
            })
        END
    END
    
    RETURN results
END
```

**ERROR DASHBOARD DATA:**

```
FUNCTION get_error_dashboard_data() {
    current_time = Date.now()
    
    dashboard = {
        summary: {
            total_errors_1h: 0,
            total_errors_24h: 0,
            critical_errors: 0,
            error_rate_per_minute: 0,
            top_errors: [],
            affected_tasks: new Set(),
            circuit_breakers: []
        },
        time_series: {
            hourly: {},
            by_category: {},
            by_severity: {}
        },
        recent_errors: []
    }
    
    # Get error history
    error_history = load_error_history()
    
    # Calculate metrics
    FOR error IN error_history:
        error_time = Date.parse(error.timestamp)
        age = current_time - error_time
        
        IF age < 3600000:  # 1 hour
            dashboard.summary.total_errors_1h++
        END
        
        IF age < 86400000:  # 24 hours
            dashboard.summary.total_errors_24h++
        END
        
        IF error.severity == ERROR_SEVERITY.CRITICAL:
            dashboard.summary.critical_errors++
        END
        
        IF error.context?.task_id:
            dashboard.summary.affected_tasks.add(error.context.task_id)
        END
    END
    
    # Calculate error rate
    dashboard.summary.error_rate_per_minute = dashboard.summary.total_errors_1h / 60
    
    # Get circuit breaker status
    FOR name, breaker IN CIRCUIT_BREAKERS:
        dashboard.summary.circuit_breakers.push(breaker.get_status())
    END
    
    # Get top errors
    error_counts = {}
    FOR error IN error_history:
        key = error.code + ":" + error.message.substring(0, 50)
        error_counts[key] = (error_counts[key] || 0) + 1
    END
    
    dashboard.summary.top_errors = Object.entries(error_counts)
        .sort((a, b) => b[1] - a[1])
        .slice(0, 10)
        .map(([key, count]) => ({ error: key, count: count }))
    
    # Convert affected tasks set to array
    dashboard.summary.affected_tasks = Array.from(dashboard.summary.affected_tasks)
    
    RETURN dashboard
END
```

**EXPORTS:**
- `create_error(message, options)` -> Create structured error
- `with_error_context(context, fn)` -> Execute with error context
- `enrich_error(error)` -> Add context to error
- `get_circuit_breaker(name, options)` -> Get/create circuit breaker
- `with_recovery(fn, strategy, options)` -> Wrap function with recovery
- `log_error(error, context)` -> Log structured error
- `with_rollback(id, operation, rollback)` -> Execute with rollback
- `execute_all_rollbacks(filter)` -> Execute pending rollbacks
- `get_error_dashboard_data()` -> Get error metrics dashboard