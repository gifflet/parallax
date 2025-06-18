**ERROR HANDLER MODULE**

Provides structured error handling with correlation IDs, circuit breakers, and automatic recovery.

**CORE CONCEPTS:**
- Error classification by severity and category
- Context enrichment with correlation IDs
- Recovery strategies with retry logic
- Circuit breaker pattern for fault tolerance

**ERROR STRUCTURE:**
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
    RESOURCE: "resource",
    VALIDATION: "validation"
}
```

**MAIN FUNCTIONS:**

```
FUNCTION create_error(message, options = {}):
    RETURN {
        id: generate_error_id(),
        message: message,
        timestamp: iso_timestamp(),
        category: options.category || ERROR_CATEGORIES.SYSTEM,
        severity: options.severity || ERROR_SEVERITY.MEDIUM,
        code: options.code || "UNKNOWN",
        context: options.context || {},
        correlation_id: options.correlation_id || get_current_correlation_id(),
        recoverable: options.recoverable !== false,
        retry_count: 0,
        cause: options.cause || null
    }
END

FUNCTION with_error_context(context, fn):
    # Push context to stack
    push_context(context)
    TRY:
        RETURN fn()
    CATCH error:
        throw enrich_error(error)
    FINALLY:
        pop_context()
    END
END

FUNCTION enrich_error(error):
    # Convert to structured error if needed
    structured = error instanceof StructuredError ? error : create_error(error.message, {
        cause: error,
        context: get_current_context()
    })
    
    # Add system context
    structured.context.system = {
        memory_usage: get_memory_usage(),
        uptime: get_process_uptime()
    }
    
    RETURN structured
END

FUNCTION with_recovery(fn, strategy = "exponential_backoff", options = {}):
    strategies = {
        exponential_backoff: async (fn, options) => {
            FOR attempt IN range(1, options.max_retries + 1):
                TRY:
                    RETURN await fn()
                CATCH error:
                    IF attempt == options.max_retries:
                        throw error
                    END
                    delay = Math.min(
                        options.base_delay * Math.pow(2, attempt - 1),
                        options.max_delay
                    )
                    await sleep(delay)
                END
            END
        },
        
        circuit_breaker: async (fn, options) => {
            breaker = get_circuit_breaker(options.breaker_name)
            RETURN await breaker.execute(fn)
        },
        
        fallback: async (fn, options) => {
            TRY:
                RETURN await fn()
            CATCH error:
                RETURN options.fallback()
            END
        }
    }
    
    RETURN strategies[strategy](fn, {
        max_retries: 3,
        base_delay: 1000,
        max_delay: 30000,
        ...options
    })
END

FUNCTION get_circuit_breaker(name):
    IF not CIRCUIT_BREAKERS[name]:
        CIRCUIT_BREAKERS[name] = {
            state: "CLOSED",
            failure_count: 0,
            failure_threshold: 5,
            success_threshold: 2,
            timeout: 60000,
            next_attempt: null
        }
    END
    
    breaker = CIRCUIT_BREAKERS[name]
    
    breaker.execute = async (fn) => {
        IF breaker.state == "OPEN":
            IF Date.now() < breaker.next_attempt:
                throw create_error("Circuit breaker is OPEN", {
                    code: "CIRCUIT_OPEN"
                })
            END
            breaker.state = "HALF_OPEN"
        END
        
        TRY:
            result = await fn()
            on_success(breaker)
            RETURN result
        CATCH error:
            on_failure(breaker)
            throw error
        END
    }
    
    RETURN breaker
END

FUNCTION log_error(error, additional_context = {}):
    structured_error = enrich_error(error)
    
    log_entry = {
        ...structured_error,
        context: { ...structured_error.context, ...additional_context }
    }
    
    # Write to log
    append_to_error_log(log_entry)
    
    # Emit event
    emit("error.logged", {
        error_id: structured_error.id,
        severity: structured_error.severity,
        category: structured_error.category
    })
    
    # Store critical errors
    IF structured_error.severity IN [ERROR_SEVERITY.CRITICAL, ERROR_SEVERITY.HIGH]:
        store_error_state(structured_error)
    END
    
    RETURN structured_error
END

FUNCTION with_rollback(operation_id, execute_fn, rollback_fn):
    register_rollback(operation_id, rollback_fn)
    
    TRY:
        result = await execute_fn()
        unregister_rollback(operation_id)
        RETURN result
    CATCH error:
        await rollback_fn()
        throw error
    END
END

FUNCTION attempt_error_recovery(error, task_id):
    recovery_map = {
        NETWORK: { strategy: "exponential_backoff", max_retries: 5 },
        RESOURCE: { strategy: "cleanup_and_retry", cleanup: clear_caches },
        GIT: { strategy: "conflict_resolution", resolver: auto_resolve },
        AGENT: { strategy: "restart_agent", max_restarts: 2 }
    }
    
    recovery = recovery_map[error.category]
    IF recovery:
        RETURN execute_recovery(recovery, error, task_id)
    END
    
    RETURN { success: false, reason: "No recovery strategy" }
END
```

**ERROR AGGREGATION:**
```
FUNCTION aggregate_errors(time_window = 60000):
    recent_errors = get_recent_errors(time_window)
    
    RETURN {
        total: recent_errors.length,
        by_category: group_by(recent_errors, "category"),
        by_severity: group_by(recent_errors, "severity"),
        error_rate: recent_errors.length / (time_window / 1000),
        patterns: detect_patterns(recent_errors)
    }
END
```

**EXPORTS:**
- create_error(message, options) -> Structured error
- with_error_context(context, fn) -> Execute with context
- enrich_error(error) -> Add context to error
- with_recovery(fn, strategy, options) -> Execute with recovery
- get_circuit_breaker(name) -> Circuit breaker instance
- log_error(error, context) -> Log and track error
- with_rollback(id, execute, rollback) -> Execute with rollback
- attempt_error_recovery(error, task_id) -> Try recovery
- aggregate_errors(window) -> Error statistics
- execute_all_rollbacks() -> Run pending rollbacks