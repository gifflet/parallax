**PERFORMANCE OPTIMIZER MODULE**

Provides caching, resource pooling, and performance monitoring for optimal execution.

**PURPOSE:** Optimize system performance through intelligent caching, connection pooling, and resource management

**IMPORTS:**
```
- ./event-bus.md         # For performance events
- ./error-handler.md     # For error handling
```

**CACHE IMPLEMENTATION:**

```
# Multi-tier cache with TTL and LRU eviction
CLASS Cache {
    constructor(name, options = {}) {
        this.name = name
        this.max_size = options.max_size || 1000
        this.default_ttl = options.default_ttl || 300000  # 5 minutes
        this.enable_persistence = options.enable_persistence || false
        this.persistence_file = ".worktrees/.cache/" + name + ".json"
        
        this.store = new Map()
        this.access_order = []
        this.hit_count = 0
        this.miss_count = 0
        
        IF this.enable_persistence:
            this.load_from_disk()
        END
    }
    
    get(key, fetch_fn = null) {
        entry = this.store.get(key)
        
        IF entry:
            # Check TTL
            IF entry.expires_at && Date.now() > entry.expires_at:
                this.delete(key)
                entry = null
            ELSE:
                # Update access order for LRU
                this.update_access_order(key)
                this.hit_count++
                
                emit_aggregated("cache.hit", {
                    cache: this.name,
                    key: key
                })
                
                RETURN entry.value
            END
        END
        
        this.miss_count++
        emit_aggregated("cache.miss", {
            cache: this.name,
            key: key
        })
        
        # Fetch if function provided
        IF fetch_fn:
            value = fetch_fn()
            this.set(key, value)
            RETURN value
        END
        
        RETURN null
    }
    
    async get_async(key, fetch_fn = null) {
        cached = this.get(key)
        IF cached !== null:
            RETURN cached
        END
        
        IF fetch_fn:
            value = await fetch_fn()
            this.set(key, value)
            RETURN value
        END
        
        RETURN null
    }
    
    set(key, value, ttl = null) {
        # Enforce size limit
        IF this.store.size >= this.max_size AND not this.store.has(key):
            this.evict_lru()
        END
        
        expires_at = null
        IF ttl !== null OR this.default_ttl:
            expires_at = Date.now() + (ttl || this.default_ttl)
        END
        
        this.store.set(key, {
            value: value,
            expires_at: expires_at,
            created_at: Date.now()
        })
        
        this.update_access_order(key)
        
        IF this.enable_persistence:
            this.persist_async()
        END
    }
    
    delete(key) {
        this.store.delete(key)
        this.access_order = this.access_order.filter(k => k !== key)
    }
    
    clear() {
        this.store.clear()
        this.access_order = []
        this.hit_count = 0
        this.miss_count = 0
    }
    
    evict_lru() {
        IF this.access_order.length > 0:
            lru_key = this.access_order.shift()
            this.store.delete(lru_key)
            
            emit_aggregated("cache.eviction", {
                cache: this.name,
                key: lru_key,
                reason: "lru"
            })
        END
    }
    
    update_access_order(key) {
        # Remove from current position
        this.access_order = this.access_order.filter(k => k !== key)
        # Add to end (most recently used)
        this.access_order.push(key)
    }
    
    get_stats() {
        total_requests = this.hit_count + this.miss_count
        hit_rate = total_requests > 0 ? this.hit_count / total_requests : 0
        
        RETURN {
            name: this.name,
            size: this.store.size,
            max_size: this.max_size,
            hit_count: this.hit_count,
            miss_count: this.miss_count,
            hit_rate: hit_rate,
            memory_usage: this.estimate_memory_usage()
        }
    }
    
    estimate_memory_usage() {
        # Rough estimation
        total_size = 0
        FOR [key, entry] IN this.store:
            total_size += key.length
            total_size += JSON.stringify(entry.value).length
        END
        RETURN total_size
    }
    
    persist_async() {
        # Debounced persistence
        IF this.persist_timeout:
            clearTimeout(this.persist_timeout)
        END
        
        this.persist_timeout = setTimeout(() => {
            this.save_to_disk()
        }, 1000)
    }
    
    save_to_disk() {
        CREATE_DIR_IF_NOT_EXISTS(".worktrees/.cache")
        
        serializable = {
            entries: Array.from(this.store.entries()),
            access_order: this.access_order,
            stats: {
                hit_count: this.hit_count,
                miss_count: this.miss_count
            }
        }
        
        write_json(this.persistence_file, serializable)
    }
    
    load_from_disk() {
        IF file_exists(this.persistence_file):
            TRY:
                data = load_json(this.persistence_file)
                
                # Restore entries that haven't expired
                current_time = Date.now()
                FOR [key, entry] IN data.entries:
                    IF not entry.expires_at OR entry.expires_at > current_time:
                        this.store.set(key, entry)
                    END
                END
                
                this.access_order = data.access_order || []
                this.hit_count = data.stats?.hit_count || 0
                this.miss_count = data.stats?.miss_count || 0
            CATCH:
                LOG_WARN: "Failed to load cache from disk", { cache: this.name }
            END
        END
    }
}

# Global cache instances
CACHES = {
    task_specs: new Cache("task_specs", { max_size: 100, default_ttl: 600000 }),
    git_info: new Cache("git_info", { max_size: 500, default_ttl: 60000 }),
    dependencies: new Cache("dependencies", { max_size: 200, default_ttl: 3600000 }),
    compilation: new Cache("compilation", { max_size: 50, default_ttl: 1800000 })
}

FUNCTION get_cache(name) {
    RETURN CACHES[name] || null
END
```

**RESOURCE POOLING:**

```
CLASS ResourcePool {
    constructor(name, factory, options = {}) {
        this.name = name
        this.factory = factory
        this.min_size = options.min_size || 1
        this.max_size = options.max_size || 10
        this.acquire_timeout = options.acquire_timeout || 5000
        this.idle_timeout = options.idle_timeout || 300000
        this.validation_fn = options.validation_fn || (() => true)
        
        this.available = []
        this.in_use = new Map()
        this.total_created = 0
        this.waiting_queue = []
        
        # Pre-warm pool
        this.ensure_minimum()
    }
    
    async acquire() {
        start_time = Date.now()
        
        WHILE true:
            # Try to get available resource
            resource = this.get_available_resource()
            
            IF resource:
                # Validate resource
                IF await this.validate_resource(resource):
                    this.mark_in_use(resource)
                    RETURN resource
                ELSE:
                    # Invalid resource, destroy it
                    await this.destroy_resource(resource)
                    CONTINUE
                END
            END
            
            # Create new resource if under limit
            IF this.total_created < this.max_size:
                resource = await this.create_resource()
                this.mark_in_use(resource)
                RETURN resource
            END
            
            # Wait for resource to become available
            IF Date.now() - start_time > this.acquire_timeout:
                THROW create_error("Resource pool acquire timeout", {
                    code: "POOL_TIMEOUT",
                    context: { pool: this.name }
                })
            END
            
            await this.wait_for_available()
        END
    }
    
    release(resource) {
        IF not this.in_use.has(resource.id):
            LOG_WARN: "Attempting to release unknown resource", {
                pool: this.name,
                resource_id: resource.id
            }
            RETURN
        END
        
        this.in_use.delete(resource.id)
        
        # Check if anyone waiting
        IF this.waiting_queue.length > 0:
            waiter = this.waiting_queue.shift()
            waiter.resolve(resource)
            RETURN
        END
        
        # Return to available pool
        resource.last_used = Date.now()
        this.available.push(resource)
        
        # Schedule idle check
        this.schedule_idle_check()
    }
    
    async destroy(resource) {
        # Remove from pools
        this.in_use.delete(resource.id)
        this.available = this.available.filter(r => r.id !== resource.id)
        
        # Destroy resource
        await this.destroy_resource(resource)
        this.total_created--
        
        # Ensure minimum
        this.ensure_minimum()
    }
    
    get_available_resource() {
        WHILE this.available.length > 0:
            resource = this.available.shift()
            
            # Check if idle too long
            IF Date.now() - resource.last_used > this.idle_timeout:
                this.destroy_resource(resource)
                this.total_created--
                CONTINUE
            END
            
            RETURN resource
        END
        
        RETURN null
    }
    
    async create_resource() {
        resource = await this.factory()
        resource.id = generate_resource_id()
        resource.created_at = Date.now()
        resource.last_used = Date.now()
        this.total_created++
        
        emit("pool.resource_created", {
            pool: this.name,
            resource_id: resource.id
        })
        
        RETURN resource
    }
    
    async destroy_resource(resource) {
        TRY:
            IF resource.destroy:
                await resource.destroy()
            ELIF resource.close:
                await resource.close()
            END
        CATCH error:
            LOG_ERROR: "Error destroying resource", {
                pool: this.name,
                resource_id: resource.id,
                error: error.message
            }
        END
        
        emit("pool.resource_destroyed", {
            pool: this.name,
            resource_id: resource.id
        })
    }
    
    async validate_resource(resource) {
        TRY:
            RETURN await this.validation_fn(resource)
        CATCH:
            RETURN false
        END
    }
    
    mark_in_use(resource) {
        resource.last_used = Date.now()
        this.in_use.set(resource.id, resource)
    }
    
    wait_for_available() {
        RETURN new Promise((resolve) => {
            this.waiting_queue.push({ resolve: resolve })
        })
    }
    
    async ensure_minimum() {
        WHILE this.total_created < this.min_size:
            resource = await this.create_resource()
            this.available.push(resource)
        END
    }
    
    schedule_idle_check() {
        IF this.idle_check_timeout:
            clearTimeout(this.idle_check_timeout)
        END
        
        this.idle_check_timeout = setTimeout(() => {
            this.cleanup_idle_resources()
        }, 60000)  # Check every minute
    }
    
    cleanup_idle_resources() {
        current_time = Date.now()
        resources_to_destroy = []
        
        FOR resource IN this.available:
            IF current_time - resource.last_used > this.idle_timeout:
                IF this.total_created > this.min_size:
                    resources_to_destroy.push(resource)
                END
            END
        END
        
        FOR resource IN resources_to_destroy:
            this.destroy(resource)
        END
    }
    
    get_stats() {
        RETURN {
            name: this.name,
            total_created: this.total_created,
            available: this.available.length,
            in_use: this.in_use.size,
            waiting: this.waiting_queue.length,
            utilization: this.in_use.size / this.max_size
        }
    }
}

# Resource pool instances
RESOURCE_POOLS = {
    git_worktrees: new ResourcePool("git_worktrees", 
        async () => {
            worktree_id = generate_worktree_id()
            worktree_path = ".worktrees/pool-" + worktree_id
            await execute_command("git worktree add " + worktree_path)
            RETURN {
                path: worktree_path,
                destroy: async () => {
                    await execute_command("git worktree remove " + worktree_path)
                }
            }
        },
        { min_size: 2, max_size: 10, idle_timeout: 600000 }
    ),
    
    process_executors: new ResourcePool("process_executors",
        async () => {
            RETURN {
                execute: async (command) => {
                    RETURN await execute_command(command)
                }
            }
        },
        { min_size: 3, max_size: 20 }
    )
}

FUNCTION get_pool(name) {
    RETURN RESOURCE_POOLS[name] || null
END
```

**BATCH PROCESSING:**

```
CLASS BatchProcessor {
    constructor(name, processor_fn, options = {}) {
        this.name = name
        this.processor_fn = processor_fn
        this.batch_size = options.batch_size || 10
        this.batch_timeout = options.batch_timeout || 100
        this.max_concurrent = options.max_concurrent || 1
        
        this.queue = []
        this.processing = false
        this.timeout_handle = null
    }
    
    add(item) {
        this.queue.push({
            item: item,
            added_at: Date.now(),
            promise: new Promise((resolve, reject) => {
                item._resolve = resolve
                item._reject = reject
            })
        })
        
        # Process if batch size reached
        IF this.queue.length >= this.batch_size:
            this.process_batch()
        ELSE:
            # Schedule timeout processing
            this.schedule_timeout_processing()
        END
        
        RETURN item.promise
    }
    
    schedule_timeout_processing() {
        IF this.timeout_handle:
            clearTimeout(this.timeout_handle)
        END
        
        this.timeout_handle = setTimeout(() => {
            this.process_batch()
        }, this.batch_timeout)
    }
    
    async process_batch() {
        IF this.processing OR this.queue.length == 0:
            RETURN
        END
        
        this.processing = true
        
        # Clear timeout
        IF this.timeout_handle:
            clearTimeout(this.timeout_handle)
            this.timeout_handle = null
        END
        
        # Get batch
        batch = this.queue.splice(0, this.batch_size)
        items = batch.map(b => b.item)
        
        TRY:
            # Process batch
            results = await this.processor_fn(items)
            
            # Resolve promises
            FOR i IN range(batch.length):
                batch[i].item._resolve(results[i])
            END
        CATCH error:
            # Reject all promises
            FOR entry IN batch:
                entry.item._reject(error)
            END
        END
        
        this.processing = false
        
        # Process next batch if items remain
        IF this.queue.length > 0:
            setImmediate(() => this.process_batch())
        END
    }
    
    get_stats() {
        RETURN {
            name: this.name,
            queue_size: this.queue.length,
            processing: this.processing
        }
    }
}

# Batch processor instances
BATCH_PROCESSORS = {
    state_updates: new BatchProcessor("state_updates",
        async (updates) => {
            # Batch state updates
            grouped = {}
            FOR update IN updates:
                IF not grouped[update.task_id]:
                    grouped[update.task_id] = []
                END
                grouped[update.task_id].push(update)
            END
            
            results = []
            FOR task_id, task_updates IN grouped:
                merged_update = merge_updates(task_updates)
                result = await update_task_state(task_id, merged_update)
                results.push(result)
            END
            
            RETURN results
        },
        { batch_size: 20, batch_timeout: 200 }
    )
}
```

**PERFORMANCE MONITORING:**

```
PERFORMANCE_METRICS = {
    operations: {},
    intervals: {}
}

FUNCTION measure_performance(operation_name, fn) {
    RETURN async (...args) => {
        start_time = performance.now()
        start_memory = get_memory_usage()
        
        TRY:
            result = await fn(...args)
            
            end_time = performance.now()
            end_memory = get_memory_usage()
            
            # Record metrics
            record_performance_metric(operation_name, {
                duration: end_time - start_time,
                memory_delta: end_memory - start_memory,
                success: true
            })
            
            RETURN result
        CATCH error:
            end_time = performance.now()
            
            record_performance_metric(operation_name, {
                duration: end_time - start_time,
                success: false,
                error: error.message
            })
            
            THROW error
        END
    }
END

FUNCTION record_performance_metric(operation_name, metric) {
    IF not PERFORMANCE_METRICS.operations[operation_name]:
        PERFORMANCE_METRICS.operations[operation_name] = {
            count: 0,
            total_duration: 0,
            min_duration: Infinity,
            max_duration: 0,
            avg_duration: 0,
            success_count: 0,
            error_count: 0,
            memory_deltas: []
        }
    END
    
    stats = PERFORMANCE_METRICS.operations[operation_name]
    stats.count++
    stats.total_duration += metric.duration
    stats.min_duration = Math.min(stats.min_duration, metric.duration)
    stats.max_duration = Math.max(stats.max_duration, metric.duration)
    stats.avg_duration = stats.total_duration / stats.count
    
    IF metric.success:
        stats.success_count++
    ELSE:
        stats.error_count++
    END
    
    IF metric.memory_delta !== undefined:
        stats.memory_deltas.push(metric.memory_delta)
        # Keep only last 100 memory deltas
        IF stats.memory_deltas.length > 100:
            stats.memory_deltas.shift()
        END
    END
    
    # Emit performance event for monitoring
    emit_aggregated("performance.metric", {
        operation: operation_name,
        duration: metric.duration,
        success: metric.success
    }, {
        aggregation_key: "performance.metrics",
        aggregation_interval: 5000
    })
END

FUNCTION get_performance_report() {
    report = {
        summary: {
            total_operations: 0,
            total_duration: 0,
            cache_hit_rate: 0,
            pool_utilization: {},
            batch_queue_sizes: {}
        },
        operations: {},
        caches: {},
        pools: {},
        batches: {}
    }
    
    # Operation metrics
    FOR name, stats IN PERFORMANCE_METRICS.operations:
        report.operations[name] = {
            ...stats,
            success_rate: stats.count > 0 ? stats.success_count / stats.count : 0,
            avg_memory_delta: stats.memory_deltas.length > 0 ? 
                average(stats.memory_deltas) : 0
        }
        report.summary.total_operations += stats.count
        report.summary.total_duration += stats.total_duration
    END
    
    # Cache metrics
    total_hits = 0
    total_requests = 0
    FOR name, cache IN CACHES:
        cache_stats = cache.get_stats()
        report.caches[name] = cache_stats
        total_hits += cache_stats.hit_count
        total_requests += cache_stats.hit_count + cache_stats.miss_count
    END
    report.summary.cache_hit_rate = total_requests > 0 ? total_hits / total_requests : 0
    
    # Pool metrics
    FOR name, pool IN RESOURCE_POOLS:
        pool_stats = pool.get_stats()
        report.pools[name] = pool_stats
        report.summary.pool_utilization[name] = pool_stats.utilization
    END
    
    # Batch processor metrics
    FOR name, processor IN BATCH_PROCESSORS:
        batch_stats = processor.get_stats()
        report.batches[name] = batch_stats
        report.summary.batch_queue_sizes[name] = batch_stats.queue_size
    END
    
    RETURN report
END
```

**LAZY LOADING:**

```
CLASS LazyLoader {
    constructor(loader_fn) {
        this.loader_fn = loader_fn
        this.loaded = false
        this.loading = false
        this.value = null
        this.error = null
        this.load_promise = null
    }
    
    async get() {
        IF this.loaded:
            IF this.error:
                THROW this.error
            END
            RETURN this.value
        END
        
        IF this.loading:
            RETURN await this.load_promise
        END
        
        this.loading = true
        this.load_promise = this.load()
        
        TRY:
            await this.load_promise
            RETURN this.value
        FINALLY:
            this.loading = false
        END
    }
    
    async load() {
        TRY:
            this.value = await this.loader_fn()
            this.loaded = true
            RETURN this.value
        CATCH error:
            this.error = error
            this.loaded = true
            THROW error
        END
    }
    
    reset() {
        this.loaded = false
        this.loading = false
        this.value = null
        this.error = null
        this.load_promise = null
    }
}

FUNCTION create_lazy_loader(loader_fn) {
    RETURN new LazyLoader(loader_fn)
END
```

**EXPORTS:**
- `get_cache(name)` -> Get cache instance
- `get_pool(name)` -> Get resource pool
- `measure_performance(name, fn)` -> Wrap function with performance measurement
- `get_performance_report()` -> Get performance metrics report
- `create_lazy_loader(loader_fn)` -> Create lazy loader
- `BatchProcessor` -> Batch processor class
- `Cache` -> Cache class
- `ResourcePool` -> Resource pool class