**CONFIGURATION MANAGEMENT MODULE**

Provides dynamic configuration loading with validation, hot-reloading, and environment-specific overrides.

**PURPOSE:** Manage configuration with schema validation, inheritance, and runtime updates

**IMPORTS:**
```
- ./event-bus.md    # For configuration change events
```

**CONFIGURATION STRUCTURE:**
```
CONFIG_SCHEMA = {
    version: { type: "string", required: true },
    defaults: {
        type: "object",
        properties: {
            max_agents: { type: "number", min: 1, max: 50 },
            review_mode: { type: "string", enum: ["strict", "balanced", "lenient"] },
            branch_pattern: { type: "string", pattern: ".*{id}.*" },
            worktree_dir: { type: "string" }
        }
    },
    features: {
        type: "object",
        properties: {
            auto_cleanup: { type: "boolean" },
            progress_display: { type: "boolean" },
            state_persistence: { type: "boolean" },
            interactive_conflicts: { type: "boolean" },
            auto_retry: { type: "boolean" },
            notifications: { type: "boolean" }
        }
    },
    timeouts: {
        type: "object",
        properties: {
            agent_idle: { type: "number", min: 60 },
            task_max: { type: "number", min: 300 },
            review_max: { type: "number", min: 60 },
            correction_max: { type: "number", min: 300 }
        }
    }
}
```

**CORE IMPLEMENTATION:**

```
CLASS ConfigurationManager {
    constructor() {
        this.config_dir = "./parallel-dev/config"
        this.default_file = this.config_dir + "/defaults.yaml"
        this.profiles_dir = this.config_dir + "/profiles"
        this.env_prefix = "PARALLEL_DEV_"
        
        this.current_config = null
        this.active_profile = null
        this.watchers = new Map()
        this.validators = new Map()
        this.transformers = []
    }
    
    async load_configuration(profile_name = null) {
        TRY:
            # Load base defaults
            defaults = await this.load_yaml_file(this.default_file)
            
            # Validate defaults
            this.validate_config(defaults, "defaults")
            
            # Start with defaults
            config = deep_clone(defaults)
            
            # Apply profile if specified
            IF profile_name:
                profile_config = await this.load_profile(profile_name)
                config = this.merge_configs(config, profile_config)
                this.active_profile = profile_name
            END
            
            # Apply environment overrides
            env_overrides = this.load_env_overrides()
            config = this.merge_configs(config, env_overrides)
            
            # Apply transformers
            FOR transformer IN this.transformers:
                config = transformer(config)
            END
            
            # Final validation
            this.validate_config(config, "final")
            
            # Update current config
            previous_config = this.current_config
            this.current_config = config
            
            # Emit change event if config changed
            IF previous_config AND not deep_equals(previous_config, config):
                changes = calculate_config_diff(previous_config, config)
                emit("config.changed", {
                    previous: previous_config,
                    current: config,
                    changes: changes
                })
            END
            
            RETURN config
        CATCH error:
            LOG_ERROR: "Failed to load configuration", {
                profile: profile_name,
                error: error.message
            }
            
            # Fall back to cached config if available
            IF this.current_config:
                LOG_WARN: "Using cached configuration"
                RETURN this.current_config
            END
            
            THROW error
        END
    }
    
    async load_profile(profile_name) {
        profile_file = this.profiles_dir + "/" + profile_name + ".yaml"
        
        IF not await file_exists(profile_file):
            THROW new Error("Profile not found: " + profile_name)
        END
        
        profile_config = await this.load_yaml_file(profile_file)
        
        # Handle profile inheritance
        IF profile_config.extends:
            parent_config = await this.load_profile(profile_config.extends)
            profile_config = this.merge_configs(parent_config, profile_config)
            delete profile_config.extends
        END
        
        RETURN profile_config
    }
    
    load_env_overrides() {
        overrides = {}
        
        FOR [key, value] IN process.env:
            IF key.startsWith(this.env_prefix):
                config_path = key
                    .substring(this.env_prefix.length)
                    .toLowerCase()
                    .split("__")  # Double underscore for nesting
                
                # Convert to nested object
                set_nested_value(overrides, config_path, parse_env_value(value))
            END
        END
        
        RETURN overrides
    }
    
    merge_configs(base, override) {
        result = deep_clone(base)
        
        FUNCTION merge_recursive(target, source):
            FOR [key, value] IN source:
                IF value === null OR value === undefined:
                    # Skip null/undefined values
                    CONTINUE
                ELIF is_object(value) AND is_object(target[key]):
                    # Recursively merge objects
                    merge_recursive(target[key], value)
                ELIF key.endsWith("+"):
                    # Array append mode
                    actual_key = key.slice(0, -1)
                    IF is_array(target[actual_key]):
                        target[actual_key].push(...value)
                    END
                ELSE:
                    # Direct override
                    target[key] = deep_clone(value)
                END
            END
        END
        
        merge_recursive(result, override)
        RETURN result
    }
    
    validate_config(config, context = "config") {
        errors = []
        
        # Schema validation
        validate_against_schema(config, CONFIG_SCHEMA, errors, "")
        
        # Custom validators
        FOR [path, validator] IN this.validators:
            value = get_nested_value(config, path.split("."))
            IF value !== undefined:
                validation_result = validator(value, config)
                IF validation_result !== true:
                    errors.push({
                        path: path,
                        error: validation_result
                    })
                END
            END
        END
        
        # Semantic validation
        errors.push(...this.validate_semantic_rules(config))
        
        IF errors.length > 0:
            error_message = "Configuration validation failed for " + context + ":\n"
            FOR error IN errors:
                error_message += "  - " + error.path + ": " + error.error + "\n"
            END
            THROW new Error(error_message)
        END
    }
    
    validate_semantic_rules(config) {
        errors = []
        
        # Max agents should not exceed available CPU cores
        cpu_count = get_cpu_count()
        IF config.defaults?.max_agents > cpu_count * 2:
            errors.push({
                path: "defaults.max_agents",
                error: `Value ${config.defaults.max_agents} exceeds recommended limit of ${cpu_count * 2}`
            })
        END
        
        # Timeout validations
        IF config.timeouts?.task_max < config.timeouts?.review_max * 2:
            errors.push({
                path: "timeouts.task_max",
                error: "Task timeout should be at least 2x review timeout"
            })
        END
        
        RETURN errors
    }
    
    register_validator(path, validator_fn) {
        this.validators.set(path, validator_fn)
    }
    
    register_transformer(transformer_fn) {
        this.transformers.push(transformer_fn)
    }
    
    async start_hot_reload() {
        # Watch default config
        this.watch_file(this.default_file, async () => {
            LOG_INFO: "Default configuration changed, reloading..."
            await this.reload_configuration()
        })
        
        # Watch active profile
        IF this.active_profile:
            profile_file = this.profiles_dir + "/" + this.active_profile + ".yaml"
            this.watch_file(profile_file, async () => {
                LOG_INFO: "Profile configuration changed, reloading..."
                await this.reload_configuration()
            })
        END
        
        # Watch for new profiles
        this.watch_directory(this.profiles_dir, async (event) => {
            IF event.type == "add":
                emit("config.profile_added", { profile: event.filename })
            END
        })
    }
    
    stop_hot_reload() {
        FOR [path, watcher] IN this.watchers:
            watcher.close()
        END
        this.watchers.clear()
    }
    
    async reload_configuration() {
        TRY:
            new_config = await this.load_configuration(this.active_profile)
            
            LOG_INFO: "Configuration reloaded successfully"
            
            # Emit reload event
            emit("config.reloaded", {
                profile: this.active_profile,
                config: new_config
            })
        CATCH error:
            LOG_ERROR: "Failed to reload configuration", {
                error: error.message
            }
            
            emit("config.reload_failed", {
                error: error.message,
                profile: this.active_profile
            })
        END
    }
    
    watch_file(file_path, callback) {
        IF this.watchers.has(file_path):
            RETURN
        END
        
        watcher = fs.watch(file_path, { persistent: false }, (event) => {
            IF event == "change":
                # Debounce to avoid multiple triggers
                IF watcher.timeout:
                    clearTimeout(watcher.timeout)
                END
                
                watcher.timeout = setTimeout(() => {
                    callback()
                }, 500)
            END
        })
        
        this.watchers.set(file_path, watcher)
    }
    
    watch_directory(dir_path, callback) {
        IF this.watchers.has(dir_path):
            RETURN
        END
        
        watcher = fs.watch(dir_path, { persistent: false }, (event, filename) => {
            callback({ type: event, filename: filename })
        })
        
        this.watchers.set(dir_path, watcher)
    }
    
    get(path, default_value = undefined) {
        IF not this.current_config:
            THROW new Error("Configuration not loaded")
        END
        
        RETURN get_nested_value(this.current_config, path.split(".")) || default_value
    }
    
    set(path, value) {
        IF not this.current_config:
            THROW new Error("Configuration not loaded")
        END
        
        # Create deep copy to maintain immutability
        new_config = deep_clone(this.current_config)
        set_nested_value(new_config, path.split("."), value)
        
        # Validate the change
        this.validate_config(new_config, "runtime_update")
        
        # Apply the change
        old_value = this.get(path)
        this.current_config = new_config
        
        # Emit change event
        emit("config.value_changed", {
            path: path,
            old_value: old_value,
            new_value: value
        })
    }
    
    get_profile_list() {
        files = list_files(this.profiles_dir)
        RETURN files
            .filter(f => f.endsWith(".yaml"))
            .map(f => f.replace(".yaml", ""))
    }
    
    export_config(include_defaults = false) {
        config = deep_clone(this.current_config)
        
        IF not include_defaults:
            # Remove values that match defaults
            defaults = this.load_yaml_file_sync(this.default_file)
            config = remove_default_values(config, defaults)
        END
        
        RETURN config
    }
}

# Global instance
CONFIG_MANAGER = new ConfigurationManager()
```

**UTILITY FUNCTIONS:**

```
FUNCTION load_configuration(profile = null) {
    RETURN CONFIG_MANAGER.load_configuration(profile)
}

FUNCTION get_config(path, default_value) {
    RETURN CONFIG_MANAGER.get(path, default_value)
}

FUNCTION set_config(path, value) {
    RETURN CONFIG_MANAGER.set(path, value)
}

FUNCTION deep_clone(obj) {
    IF obj === null OR typeof obj !== "object":
        RETURN obj
    END
    
    IF obj instanceof Date:
        RETURN new Date(obj.getTime())
    END
    
    IF obj instanceof Array:
        RETURN obj.map(item => deep_clone(item))
    END
    
    cloned = {}
    FOR [key, value] IN obj:
        cloned[key] = deep_clone(value)
    END
    RETURN cloned
}

FUNCTION deep_equals(obj1, obj2) {
    IF obj1 === obj2:
        RETURN true
    END
    
    IF obj1 == null OR obj2 == null:
        RETURN false
    END
    
    IF typeof obj1 !== typeof obj2:
        RETURN false
    END
    
    IF typeof obj1 !== "object":
        RETURN obj1 === obj2
    END
    
    keys1 = Object.keys(obj1)
    keys2 = Object.keys(obj2)
    
    IF keys1.length !== keys2.length:
        RETURN false
    END
    
    FOR key IN keys1:
        IF not deep_equals(obj1[key], obj2[key]):
            RETURN false
        END
    END
    
    RETURN true
}

FUNCTION get_nested_value(obj, path) {
    result = obj
    FOR segment IN path:
        IF result == null OR typeof result !== "object":
            RETURN undefined
        END
        result = result[segment]
    END
    RETURN result
}

FUNCTION set_nested_value(obj, path, value) {
    current = obj
    FOR i IN range(path.length - 1):
        segment = path[i]
        IF not (segment IN current) OR typeof current[segment] !== "object":
            current[segment] = {}
        END
        current = current[segment]
    END
    current[path[path.length - 1]] = value
}

FUNCTION parse_env_value(value) {
    # Try to parse as JSON first
    TRY:
        RETURN JSON.parse(value)
    CATCH:
        # Not JSON
    END
    
    # Check for boolean
    IF value.toLowerCase() == "true":
        RETURN true
    ELIF value.toLowerCase() == "false":
        RETURN false
    END
    
    # Check for number
    IF /^-?\d+(\.\d+)?$/.test(value):
        RETURN parseFloat(value)
    END
    
    # Return as string
    RETURN value
}

FUNCTION validate_against_schema(obj, schema, errors, path) {
    IF schema.type:
        CASE schema.type:
            "string":
                IF typeof obj !== "string":
                    errors.push({ path: path, error: "Expected string" })
                ELIF schema.pattern AND not new RegExp(schema.pattern).test(obj):
                    errors.push({ path: path, error: "Does not match pattern: " + schema.pattern })
                END
                
            "number":
                IF typeof obj !== "number":
                    errors.push({ path: path, error: "Expected number" })
                ELIF schema.min !== undefined AND obj < schema.min:
                    errors.push({ path: path, error: "Value below minimum: " + schema.min })
                ELIF schema.max !== undefined AND obj > schema.max:
                    errors.push({ path: path, error: "Value above maximum: " + schema.max })
                END
                
            "boolean":
                IF typeof obj !== "boolean":
                    errors.push({ path: path, error: "Expected boolean" })
                END
                
            "object":
                IF typeof obj !== "object" OR obj === null:
                    errors.push({ path: path, error: "Expected object" })
                ELIF schema.properties:
                    FOR [key, prop_schema] IN schema.properties:
                        prop_path = path ? path + "." + key : key
                        IF key IN obj:
                            validate_against_schema(obj[key], prop_schema, errors, prop_path)
                        ELIF prop_schema.required:
                            errors.push({ path: prop_path, error: "Required field missing" })
                        END
                    END
                END
        END
    END
    
    IF schema.enum AND not schema.enum.includes(obj):
        errors.push({ path: path, error: "Value not in enum: " + schema.enum.join(", ") })
    END
}

FUNCTION calculate_config_diff(old_config, new_config) {
    changes = []
    
    FUNCTION compare_objects(old_obj, new_obj, path = ""):
        all_keys = new Set([...Object.keys(old_obj), ...Object.keys(new_obj)])
        
        FOR key IN all_keys:
            current_path = path ? path + "." + key : key
            
            IF not (key IN old_obj):
                changes.push({
                    type: "added",
                    path: current_path,
                    value: new_obj[key]
                })
            ELIF not (key IN new_obj):
                changes.push({
                    type: "removed",
                    path: current_path,
                    value: old_obj[key]
                })
            ELIF typeof old_obj[key] == "object" AND typeof new_obj[key] == "object":
                compare_objects(old_obj[key], new_obj[key], current_path)
            ELIF old_obj[key] !== new_obj[key]:
                changes.push({
                    type: "modified",
                    path: current_path,
                    old_value: old_obj[key],
                    new_value: new_obj[key]
                })
            END
        END
    END
    
    compare_objects(old_config, new_config)
    RETURN changes
}
```

**CONFIGURATION VALIDATION PRESETS:**

```
# Register common validators
CONFIG_MANAGER.register_validator("defaults.max_agents", (value, config) => {
    IF value < 1:
        RETURN "Must be at least 1"
    END
    IF value > 50:
        RETURN "Cannot exceed 50 for system stability"
    END
    RETURN true
})

CONFIG_MANAGER.register_validator("defaults.branch_pattern", (value, config) => {
    IF not value.includes("{id}"):
        RETURN "Must contain {id} placeholder"
    END
    RETURN true
})

CONFIG_MANAGER.register_validator("resources.max_memory_mb", (value, config) => {
    available_memory = get_available_memory_mb()
    IF value > available_memory * 0.8:
        RETURN `Cannot exceed 80% of available memory (${available_memory}MB)`
    END
    RETURN true
})

# Register transformers
CONFIG_MANAGER.register_transformer((config) => {
    # Auto-adjust timeouts based on max_agents
    IF config.defaults?.max_agents > 10:
        # Increase timeouts for high concurrency
        IF config.timeouts:
            config.timeouts.agent_idle *= 1.5
            config.timeouts.task_max *= 1.5
        END
    END
    RETURN config
})

CONFIG_MANAGER.register_transformer((config) => {
    # Add computed values
    config.computed = {
        max_parallel_tasks: Math.min(
            config.defaults?.max_agents || 5,
            get_cpu_count()
        ),
        effective_timeout: Math.max(
            config.timeouts?.task_max || 3600,
            config.timeouts?.review_max || 900,
            config.timeouts?.correction_max || 1800
        )
    }
    RETURN config
})
```

**EXPORTS:**
- `CONFIG_MANAGER` -> Global configuration manager instance
- `load_configuration(profile)` -> Load configuration with profile
- `get_config(path, default)` -> Get configuration value
- `set_config(path, value)` -> Set configuration value
- `deep_clone(obj)` -> Deep clone object
- `deep_equals(obj1, obj2)` -> Deep equality check