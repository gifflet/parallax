**CONFIGURATION MANAGEMENT MODULE**

Provides dynamic configuration loading with validation, hot-reloading, and environment-specific overrides.

**CORE CONCEPTS:**
- Schema-based validation
- Configuration inheritance and merging
- Hot-reload with change notifications
- Environment and profile support

**CONFIGURATION SCHEMA:**
```
CONFIG_SCHEMA = {
    version: { type: "string", required: true },
    defaults: {
        max_agents: { type: "number", min: 1, max: 50, default: 5 },
        review_mode: { type: "string", enum: ["strict", "balanced", "lenient"], default: "balanced" },
        branch_pattern: { type: "string", pattern: ".*{id}.*", default: "feature/task-{id}" },
        worktree_dir: { type: "string", default: ".worktrees" }
    },
    features: {
        auto_cleanup: { type: "boolean", default: true },
        progress_display: { type: "boolean", default: true },
        state_persistence: { type: "boolean", default: true },
        notifications: { type: "boolean", default: false }
    },
    timeouts: {
        agent_idle: { type: "number", min: 60, default: 300 },
        task_max: { type: "number", min: 300, default: 3600 },
        review_max: { type: "number", min: 60, default: 900 }
    },
    resource_limits: {
        max_memory_mb: { type: "number", default: 2048 },
        max_disk_mb: { type: "number", default: 5120 },
        max_open_files: { type: "number", default: 100 }
    }
}
```

**MAIN FUNCTIONS:**

```
FUNCTION load_configuration(profile = null):
    # Load base config
    base_config = load_yaml("./config/defaults.yaml")
    
    # Load profile if specified
    profile_config = {}
    IF profile:
        profile_path = "./config/profiles/" + profile + ".yaml"
        IF file_exists(profile_path):
            profile_config = load_yaml(profile_path)
        END
    END
    
    # Load environment overrides
    env_config = load_env_config()
    
    # Merge configurations
    config = deep_merge(base_config, profile_config, env_config)
    
    # Validate against schema
    validation_result = validate_config(config, CONFIG_SCHEMA)
    IF not validation_result.valid:
        THROW new Error("Invalid configuration: " + validation_result.errors)
    END
    
    # Apply defaults
    config = apply_defaults(config, CONFIG_SCHEMA)
    
    # Watch for changes
    IF config.features.hot_reload:
        watch_config_files(config)
    END
    
    emit("config.loaded", config)
    RETURN config
END

FUNCTION validate_config(config, schema):
    errors = []
    
    FUNCTION validate_object(obj, schema_def, path = ""):
        FOR key, rule IN schema_def:
            value = obj[key]
            
            # Check required
            IF rule.required AND value === undefined:
                errors.push(path + key + " is required")
                CONTINUE
            END
            
            # Check type
            IF value !== undefined AND typeof value !== rule.type:
                errors.push(path + key + " must be " + rule.type)
            END
            
            # Check constraints
            IF rule.type == "number" AND value !== undefined:
                IF rule.min !== undefined AND value < rule.min:
                    errors.push(path + key + " must be >= " + rule.min)
                END
                IF rule.max !== undefined AND value > rule.max:
                    errors.push(path + key + " must be <= " + rule.max)
                END
            END
            
            IF rule.type == "string" AND value !== undefined:
                IF rule.enum AND not rule.enum.includes(value):
                    errors.push(path + key + " must be one of: " + rule.enum)
                END
                IF rule.pattern AND not new RegExp(rule.pattern).test(value):
                    errors.push(path + key + " must match pattern: " + rule.pattern)
                END
            END
        END
    END
    
    validate_object(config, schema)
    
    RETURN { valid: errors.length == 0, errors: errors }
END

FUNCTION deep_merge(...objects):
    result = {}
    
    FOR obj IN objects:
        FOR key, value IN obj:
            IF typeof value == "object" AND not Array.isArray(value):
                result[key] = deep_merge(result[key] || {}, value)
            ELSE:
                result[key] = value
            END
        END
    END
    
    RETURN result
END

FUNCTION watch_config_files(config):
    watched_files = [
        "./config/defaults.yaml",
        "./config/profiles/*.yaml"
    ]
    
    FOR file IN watched_files:
        watch_file(file, () => {
            TRY:
                new_config = load_configuration(config.profile)
                emit("config.changed", {
                    old: config,
                    new: new_config,
                    changed_keys: get_changed_keys(config, new_config)
                })
            CATCH error:
                emit("config.error", { error: error.message })
            END
        })
    END
END

FUNCTION get_config_value(config, path, default_value = undefined):
    # Get nested value using dot notation
    # Example: get_config_value(config, "features.auto_cleanup")
    parts = path.split(".")
    value = config
    
    FOR part IN parts:
        value = value?.[part]
        IF value === undefined:
            RETURN default_value
        END
    END
    
    RETURN value
END

FUNCTION update_config_value(config, path, value):
    # Update nested value and emit change event
    parts = path.split(".")
    obj = config
    
    FOR i IN range(parts.length - 1):
        IF not obj[parts[i]]:
            obj[parts[i]] = {}
        END
        obj = obj[parts[i]]
    END
    
    old_value = obj[parts[parts.length - 1]]
    obj[parts[parts.length - 1]] = value
    
    emit("config.value_changed", {
        path: path,
        old_value: old_value,
        new_value: value
    })
END
```

**EXPORTS:**
- load_configuration(profile?) -> Config object
- validate_config(config, schema) -> Validation result
- deep_merge(...objects) -> Merged object
- watch_config_files(config) -> Start file watching
- get_config_value(config, path, default?) -> Value at path
- update_config_value(config, path, value) -> Update value
- apply_defaults(config, schema) -> Config with defaults
- load_env_config() -> Environment overrides
- get_profile_list() -> Available profiles