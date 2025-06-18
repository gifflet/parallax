**EVENT BUS MODULE**

Provides event-driven communication between agents and components for improved performance and decoupling.

**PURPOSE:** Enable asynchronous, non-blocking communication with event subscriptions and emissions

**EVENT TYPES:**
```
AGENT_EVENTS:
  - agent.started
  - agent.progress
  - agent.completed
  - agent.failed
  - agent.heartbeat

TASK_EVENTS:
  - task.created
  - task.assigned
  - task.status_changed
  - task.completed
  - task.failed

STATE_EVENTS:
  - state.updated
  - state.corrupted
  - state.recovered

SYSTEM_EVENTS:
  - system.resource_warning
  - system.cleanup_needed
  - system.health_check
```

**CORE IMPLEMENTATION:**

```
# Event registry and handlers
EVENT_HANDLERS = {}
EVENT_QUEUE = []
PROCESSING = false

FUNCTION subscribe(event_pattern, handler, options = {}):
    handler_id = generate_handler_id()
    
    handler_config = {
        id: handler_id,
        pattern: compile_pattern(event_pattern),
        callback: handler,
        priority: options.priority || 0,
        max_retries: options.max_retries || 3,
        timeout: options.timeout || 5000,
        filter: options.filter || null
    }
    
    IF not EVENT_HANDLERS[event_pattern]:
        EVENT_HANDLERS[event_pattern] = []
    END
    
    # Insert by priority
    EVENT_HANDLERS[event_pattern].push(handler_config)
    EVENT_HANDLERS[event_pattern].sort_by(h => h.priority)
    
    RETURN handler_id
END

FUNCTION unsubscribe(handler_id):
    FOR pattern, handlers IN EVENT_HANDLERS:
        EVENT_HANDLERS[pattern] = handlers.filter(h => h.id != handler_id)
    END
END

FUNCTION emit(event_type, data = {}, options = {}):
    event = {
        id: generate_event_id(),
        type: event_type,
        data: data,
        timestamp: iso_timestamp(),
        source: options.source || "unknown",
        correlation_id: options.correlation_id || null,
        metadata: options.metadata || {}
    }
    
    # Add to queue for async processing
    EVENT_QUEUE.push(event)
    
    # Process queue if not already processing
    IF not PROCESSING:
        process_event_queue()
    END
    
    RETURN event.id
END

FUNCTION emit_sync(event_type, data = {}, options = {}):
    # Synchronous emission for critical events
    event = {
        id: generate_event_id(),
        type: event_type,
        data: data,
        timestamp: iso_timestamp(),
        source: options.source || "system",
        correlation_id: options.correlation_id || null
    }
    
    results = process_event(event)
    RETURN results
END
```

**EVENT PROCESSING:**

```
FUNCTION process_event_queue():
    PROCESSING = true
    
    WHILE EVENT_QUEUE.length > 0:
        event = EVENT_QUEUE.shift()
        
        TRY:
            process_event(event)
        CATCH error:
            LOG_ERROR: "Event processing failed", {
                event: event,
                error: error.message
            }
            
            # Re-queue if retriable
            IF event.retry_count < 3:
                event.retry_count = (event.retry_count || 0) + 1
                EVENT_QUEUE.push(event)
            END
        END
    END
    
    PROCESSING = false
END

FUNCTION process_event(event):
    matched_handlers = []
    
    # Find all matching handlers
    FOR pattern, handlers IN EVENT_HANDLERS:
        IF matches_pattern(event.type, pattern):
            matched_handlers.extend(handlers)
        END
    END
    
    # Sort by priority
    matched_handlers.sort_by(h => h.priority)
    
    results = []
    FOR handler IN matched_handlers:
        # Apply filter if specified
        IF handler.filter AND not handler.filter(event):
            CONTINUE
        END
        
        # Execute handler with timeout
        result = execute_with_timeout(handler, event)
        results.push(result)
        
        # Stop propagation if requested
        IF result.stop_propagation:
            BREAK
        END
    END
    
    RETURN results
END

FUNCTION execute_with_timeout(handler, event):
    result = {
        handler_id: handler.id,
        success: false,
        error: null,
        stop_propagation: false
    }
    
    TRY:
        WITH_TIMEOUT(handler.timeout):
            response = handler.callback(event)
            result.success = true
            result.response = response
            result.stop_propagation = response?.stop_propagation || false
        END
    CATCH timeout_error:
        result.error = "Handler timeout after " + handler.timeout + "ms"
        LOG_ERROR: "Handler timeout", {
            handler_id: handler.id,
            event_type: event.type
        }
    CATCH error:
        result.error = error.message
        LOG_ERROR: "Handler error", {
            handler_id: handler.id,
            event_type: event.type,
            error: error.message
        }
    END
    
    RETURN result
END
```

**PATTERN MATCHING:**

```
FUNCTION compile_pattern(pattern):
    # Support wildcards: agent.* matches all agent events
    regex_pattern = pattern
        .replace(".", "\\.")
        .replace("*", ".*")
        .replace("?", ".")
    
    RETURN new RegExp("^" + regex_pattern + "$")
END

FUNCTION matches_pattern(event_type, pattern):
    IF typeof pattern == "string":
        pattern = compile_pattern(pattern)
    END
    
    RETURN pattern.test(event_type)
END
```

**EVENT AGGREGATION:**

```
# Aggregate similar events to reduce noise
EVENT_AGGREGATOR = {
    buffers: {},
    intervals: {}
}

FUNCTION emit_aggregated(event_type, data, options = {}):
    aggregation_key = options.aggregation_key || event_type
    interval = options.aggregation_interval || 1000
    
    IF not EVENT_AGGREGATOR.buffers[aggregation_key]:
        EVENT_AGGREGATOR.buffers[aggregation_key] = []
        
        # Set up flush interval
        EVENT_AGGREGATOR.intervals[aggregation_key] = SET_INTERVAL(interval, () => {
            flush_aggregated_events(aggregation_key)
        })
    END
    
    EVENT_AGGREGATOR.buffers[aggregation_key].push({
        type: event_type,
        data: data,
        timestamp: iso_timestamp()
    })
END

FUNCTION flush_aggregated_events(aggregation_key):
    buffer = EVENT_AGGREGATOR.buffers[aggregation_key]
    IF buffer.length == 0:
        RETURN
    END
    
    # Create aggregated event
    aggregated_event = {
        type: aggregation_key + ".aggregated",
        count: buffer.length,
        events: buffer,
        start_time: buffer[0].timestamp,
        end_time: buffer[buffer.length - 1].timestamp
    }
    
    emit(aggregated_event.type, aggregated_event)
    
    # Clear buffer
    EVENT_AGGREGATOR.buffers[aggregation_key] = []
END
```

**BUILT-IN HANDLERS:**

```
# Dead letter queue for unhandled events
subscribe("*", (event) => {
    IF get_handler_count(event.type) == 1:  # Only this handler
        LOG_WARN: "Unhandled event", {
            type: event.type,
            data: event.data
        }
    END
}, { priority: -1000 })

# Event metrics collector
subscribe("*", (event) => {
    update_event_metrics(event.type)
}, { priority: -999 })

# Health check responder
subscribe("system.health_check", (event) => {
    RETURN {
        status: "healthy",
        event_queue_size: EVENT_QUEUE.length,
        handlers_registered: count_all_handlers(),
        uptime: get_uptime()
    }
})
```

**UTILITY FUNCTIONS:**

```
FUNCTION wait_for_event(event_type, timeout = 5000, filter = null):
    RETURN new Promise((resolve, reject) => {
        timeout_handle = SET_TIMEOUT(timeout, () => {
            unsubscribe(handler_id)
            reject(new Error("Event wait timeout: " + event_type))
        })
        
        handler_id = subscribe(event_type, (event) => {
            IF not filter OR filter(event):
                CLEAR_TIMEOUT(timeout_handle)
                unsubscribe(handler_id)
                resolve(event)
            END
        })
    })
END

FUNCTION emit_and_wait(event_type, data, response_event, timeout = 5000):
    correlation_id = generate_correlation_id()
    
    # Emit with correlation ID
    emit(event_type, data, { correlation_id: correlation_id })
    
    # Wait for correlated response
    RETURN wait_for_event(response_event, timeout, (event) => {
        RETURN event.correlation_id == correlation_id
    })
END

FUNCTION create_event_stream(event_pattern):
    # Create an event stream for reactive programming
    stream = {
        handlers: [],
        
        subscribe: (handler) => {
            handler_id = subscribe(event_pattern, handler)
            stream.handlers.push(handler_id)
            RETURN () => unsubscribe(handler_id)
        },
        
        map: (transform) => {
            new_stream = create_event_stream(event_pattern)
            stream.subscribe((event) => {
                transformed = transform(event)
                new_stream._emit(transformed)
            })
            RETURN new_stream
        },
        
        filter: (predicate) => {
            new_stream = create_event_stream(event_pattern)
            stream.subscribe((event) => {
                IF predicate(event):
                    new_stream._emit(event)
                END
            })
            RETURN new_stream
        },
        
        _emit: (event) => {
            FOR handler_id IN stream.handlers:
                get_handler(handler_id)?.callback(event)
            END
        },
        
        destroy: () => {
            FOR handler_id IN stream.handlers:
                unsubscribe(handler_id)
            END
        }
    }
    
    RETURN stream
END
```

**EXPORTS:**
- `subscribe(pattern, handler, options)` -> Subscribe to events
- `unsubscribe(handler_id)` -> Unsubscribe handler
- `emit(event_type, data, options)` -> Emit event asynchronously
- `emit_sync(event_type, data, options)` -> Emit event synchronously
- `emit_aggregated(event_type, data, options)` -> Emit with aggregation
- `wait_for_event(event_type, timeout, filter)` -> Wait for specific event
- `emit_and_wait(event_type, data, response_event, timeout)` -> Request-response pattern
- `create_event_stream(pattern)` -> Create reactive event stream