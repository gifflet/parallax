**ARGUMENT PARSER MODULE**

Parse CLI arguments with validation, type checking, and default values.

**CORE CONCEPTS:**
- Flexible argument parsing (flags, options, positional)
- Type validation and constraints
- Default values and aliases
- Help text generation

**ARGUMENT STRUCTURE:**
```
OPTIONS = {
    max_agents: { type: "number", default: 5, min: 1, max: 20 },
    review_mode: { type: "string", default: "balanced", enum: ["strict", "balanced", "lenient"] },
    profile: { type: "string", default: null },
    branch_pattern: { type: "string", default: "feature/task-{id}" },
    status: { type: "boolean", default: false },
    resume: { type: "boolean", default: false },
    dry_run: { type: "boolean", default: false },
    help: { type: "boolean", default: false }
}

ALIASES = {
    "-m": "--max-agents",
    "-r": "--review",
    "-p": "--profile",
    "-b": "--branch",
    "-h": "--help"
}
```

**MAIN FUNCTION:**

```
FUNCTION parse_arguments(args_string):
    options = get_default_options()
    args = tokenize_arguments(args_string)
    task_ids = []
    
    i = 0
    WHILE i < args.length:
        arg = args[i]
        
        # Long options with value (--option=value)
        IF arg.startsWith("--") AND arg.includes("="):
            [option, value] = arg.split("=", 2)
            set_option(options, option, value)
            
        # Long options
        ELIF arg.startsWith("--"):
            option_name = arg.substring(2)
            
            # Boolean flags
            IF is_boolean_option(option_name):
                options[option_name] = true
            ELSE:
                # Next arg is the value
                i++
                IF i < args.length:
                    set_option(options, "--" + option_name, args[i])
                END
            END
            
        # Short options
        ELIF arg.startsWith("-") AND not is_number(arg):
            short_option = expand_alias(arg)
            
            IF is_boolean_option(short_option):
                options[short_option.substring(2)] = true
            ELSE:
                i++
                IF i < args.length:
                    set_option(options, short_option, args[i])
                END
            END
            
        # Positional arguments (task IDs)
        ELSE:
            IF is_number(arg):
                task_ids.push(parseInt(arg))
            END
        END
        
        i++
    END
    
    options.task_ids = task_ids
    RETURN validate_options(options)
END

FUNCTION set_option(options, option_key, value):
    # Remove -- prefix
    key = option_key.startsWith("--") ? option_key.substring(2) : option_key
    
    # Map common variations
    IF key == "review": key = "review_mode"
    IF key == "branch": key = "branch_pattern"
    IF key == "agents": key = "max_agents"
    
    # Get option definition
    option_def = OPTIONS[key]
    IF not option_def:
        throw new Error("Unknown option: " + option_key)
    END
    
    # Parse and validate value
    parsed_value = parse_value(value, option_def.type)
    validated_value = validate_value(parsed_value, option_def)
    
    options[key] = validated_value
END

FUNCTION validate_value(value, option_def):
    # Type check
    IF option_def.type == "number":
        IF not is_number(value):
            throw new Error("Expected number, got: " + value)
        END
        num = Number(value)
        IF option_def.min !== undefined AND num < option_def.min:
            throw new Error("Value must be >= " + option_def.min)
        END
        IF option_def.max !== undefined AND num > option_def.max:
            throw new Error("Value must be <= " + option_def.max)
        END
        RETURN num
        
    ELIF option_def.type == "string":
        IF option_def.enum AND not option_def.enum.includes(value):
            throw new Error("Value must be one of: " + option_def.enum.join(", "))
        END
        RETURN value
        
    ELIF option_def.type == "boolean":
        RETURN value == true || value == "true" || value == "1"
    END
    
    RETURN value
END

FUNCTION show_help():
    help_text = [
        "Usage: /parallel-dev [options] [task_ids...]",
        "",
        "Options:",
        "  -m, --max-agents <n>    Maximum concurrent agents (1-20, default: 5)",
        "  -r, --review <mode>     Review mode: strict|balanced|lenient (default: balanced)",
        "  -p, --profile <name>    Configuration profile to use",
        "  -b, --branch <pattern>  Branch naming pattern (default: feature/task-{id})",
        "  --status               Show current execution status",
        "  --resume               Resume the last active session",
        "  --dry-run              Show what would be done without executing",
        "  -h, --help             Show this help message",
        "",
        "Examples:",
        "  /parallel-dev                    # Process all available tasks",
        "  /parallel-dev 8 9 14            # Process specific tasks",
        "  /parallel-dev -m 2 -r strict    # Conservative execution",
        "  /parallel-dev --status          # Check current status",
        "  /parallel-dev --resume          # Continue previous session"
    ]
    
    print(help_text.join("\n"))
END
```

**EXPORTS:**
- parse_arguments(args_string) -> Parsed options object
- show_help() -> Display help text
- validate_options(options) -> Validated options
- get_default_options() -> Default option values