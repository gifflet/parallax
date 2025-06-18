**EVENT BUS MODULE**

Provides publish-subscribe event system with aggregation and async handling.

**CORE CONCEPTS:**
- Decoupled event-driven communication
- Event aggregation for performance
- Wildcard subscriptions
- Event history and replay

**EVENT STRUCTURE:**
```
EVENT = {
    name: "component.action",     # e.g., "task.completed"
    data: {},                    # Event payload
    timestamp: iso_timestamp(),
    correlation_id: string,      # For tracing related events
    source: string              # Component that emitted
}

COMMON_EVENTS = [
    "session.created", "session.completed", "session.failed",
    "task.started", "task.progress", "task.completed", "task.failed",
    "agent.started", "agent.completed", "agent.error",
    "state.updated", "state.checkpoint",
    "error.occurred", "error.recovered",
    "config.changed", "health.checked"
]
```

**MAIN FUNCTIONS:**

```
EVENT_BUS = {
    subscribers: new Map(),      # event -> [callbacks]
    event_history: [],          # Recent events
    max_history: 1000,
    aggregators: new Map()      # For batching events
}

FUNCTION emit(event_name, data = {}):
    event = {
        name: event_name,
        data: data,
        timestamp: iso_timestamp(),
        correlation_id: get_current_correlation_id(),
        source: get_caller_component()
    }
    
    # Add to history
    EVENT_BUS.event_history.push(event)
    IF EVENT_BUS.event_history.length > EVENT_BUS.max_history:
        EVENT_BUS.event_history.shift()
    END
    
    # Notify subscribers
    notify_subscribers(event)
    
    # Check aggregators
    check_aggregators(event)
END

FUNCTION subscribe(event_pattern, callback):
    IF not EVENT_BUS.subscribers.has(event_pattern):
        EVENT_BUS.subscribers.set(event_pattern, [])
    END
    
    EVENT_BUS.subscribers.get(event_pattern).push(callback)
    
    # Return unsubscribe function
    RETURN () => {
        callbacks = EVENT_BUS.subscribers.get(event_pattern)
        index = callbacks.indexOf(callback)
        IF index >= 0:
            callbacks.splice(index, 1)
        END
    }
END

FUNCTION notify_subscribers(event):
    # Direct subscribers
    callbacks = EVENT_BUS.subscribers.get(event.name) || []
    
    # Wildcard subscribers
    FOR [pattern, pattern_callbacks] IN EVENT_BUS.subscribers:
        IF pattern.includes("*") AND matches_pattern(event.name, pattern):
            callbacks.push(...pattern_callbacks)
        END
    END
    
    # Execute callbacks
    FOR callback IN callbacks:
        TRY:
            callback(event)
        CATCH error:
            console.error("Event handler error:", error)
        END
    END
END

FUNCTION emit_aggregated(event_name, data, options = {}):
    key = options.aggregation_key || event_name
    interval = options.aggregation_interval || 1000
    
    IF not EVENT_BUS.aggregators.has(key):
        EVENT_BUS.aggregators.set(key, {
            events: [],
            timer: null,
            interval: interval
        })
    END
    
    aggregator = EVENT_BUS.aggregators.get(key)
    aggregator.events.push({ name: event_name, data: data })
    
    # Reset timer
    IF aggregator.timer:
        clearTimeout(aggregator.timer)
    END
    
    aggregator.timer = setTimeout(() => {
        flush_aggregator(key)
    }, interval)
END

FUNCTION flush_aggregator(key):
    aggregator = EVENT_BUS.aggregators.get(key)
    IF not aggregator OR aggregator.events.length == 0:
        RETURN
    END
    
    # Emit aggregated event
    emit(key + ".aggregated", {
        count: aggregator.events.length,
        events: aggregator.events,
        interval: aggregator.interval
    })
    
    # Clear
    aggregator.events = []
    aggregator.timer = null
END

FUNCTION wait_for_event(event_pattern, timeout = 5000):
    RETURN new Promise((resolve, reject) => {
        timer = setTimeout(() => {
            unsubscribe()
            reject(new Error("Event timeout: " + event_pattern))
        }, timeout)
        
        unsubscribe = subscribe(event_pattern, (event) => {
            clearTimeout(timer)
            unsubscribe()
            resolve(event)
        })
    })
END

FUNCTION replay_events(filter_fn, callback):
    relevant_events = EVENT_BUS.event_history.filter(filter_fn)
    
    FOR event IN relevant_events:
        callback(event)
    END
END

FUNCTION get_event_stats(time_window = 60000):
    cutoff = Date.now() - time_window
    recent_events = EVENT_BUS.event_history.filter(e => 
        Date.parse(e.timestamp) > cutoff
    )
    
    stats = {}
    FOR event IN recent_events:
        IF not stats[event.name]:
            stats[event.name] = 0
        END
        stats[event.name]++
    END
    
    RETURN stats
END
```

**EXPORTS:**
- emit(event_name, data?) -> Emit event
- subscribe(pattern, callback) -> Unsubscribe function
- emit_aggregated(name, data, options?) -> Emit with aggregation
- wait_for_event(pattern, timeout?) -> Promise<Event>
- replay_events(filter, callback) -> Replay history
- get_event_stats(window?) -> Event statistics
- clear_history() -> Clear event history
- set_correlation_id(id) -> Set correlation context