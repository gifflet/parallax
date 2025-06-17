**ARGUMENT PARSING MODULE**

Parse CLI arguments with validation, type checking, and default values.

**INPUT:** $ARGUMENTS string from command invocation

**PARSING LOGIC:**

```
FUNCTION parse_arguments(args_string):
    # Initialize defaults
    options = {
        max_agents: 5,
        review_mode: "balanced",
        profile: null,
        branch_pattern: "feature/task-{id}",
        status: false,
        resume: false,
        dry_run: false,
        help: false,
        task_ids: []
    }
    
    # Parse arguments
    args = split_arguments(args_string)
    i = 0
    
    WHILE i < length(args):
        arg = args[i]
        
        # Long options
        IF arg.startsWith("--"):
            IF arg == "--help":
                options.help = true
            ELIF arg == "--status":
                options.status = true
            ELIF arg == "--resume":
                options.resume = true
            ELIF arg == "--dry-run":
                options.dry_run = true
            ELIF arg.contains("="):
                [key, value] = arg.split("=")
                CASE key:
                    "--max-agents": options.max_agents = validate_number(value, 1, 20)
                    "--review": options.review_mode = validate_review_mode(value)
                    "--profile": options.profile = value
                    "--branch": options.branch_pattern = value
            ELSE:
                # Handle --option value format
                next_arg = args[i+1] if exists
                CASE arg:
                    "--max-agents", "-m": 
                        options.max_agents = validate_number(next_arg, 1, 20)
                        i += 1
                    "--review", "-r":
                        options.review_mode = validate_review_mode(next_arg)
                        i += 1
                    "--profile", "-p":
                        options.profile = next_arg
                        i += 1
                    "--branch", "-b":
                        options.branch_pattern = next_arg
                        i += 1
        
        # Short options
        ELIF arg.startsWith("-") AND not arg.isNumber():
            CASE arg:
                "-h": options.help = true
                "-m": 
                    options.max_agents = validate_number(args[i+1], 1, 20)
                    i += 1
                "-r":
                    options.review_mode = validate_review_mode(args[i+1])
                    i += 1
                "-p":
                    options.profile = args[i+1]
                    i += 1
                "-b":
                    options.branch_pattern = args[i+1]
                    i += 1
        
        # Positional arguments (task IDs)
        ELIF arg.isNumber():
            options.task_ids.push(parseInt(arg))
        
        # Comma-separated task IDs
        ELIF arg.matches(/^\d+(,\d+)*$/):
            task_list = arg.split(",").map(parseInt)
            options.task_ids.extend(task_list)
        
        i += 1
    END
    
    RETURN options
END
```

**VALIDATION FUNCTIONS:**

```
FUNCTION validate_number(value, min, max):
    num = parseInt(value)
    IF isNaN(num):
        ERROR: "Invalid number: {value}"
    IF num < min OR num > max:
        ERROR: "Value must be between {min} and {max}"
    RETURN num
END

FUNCTION validate_review_mode(mode):
    valid_modes = ["strict", "balanced", "lenient"]
    IF mode NOT IN valid_modes:
        ERROR: "Invalid review mode: {mode}. Must be one of: {valid_modes}"
    RETURN mode
END
```

**HELP TEXT GENERATION:**

```
FUNCTION show_help():
    DISPLAY: """
PARALLEL DEVELOPMENT ORCHESTRATION

USAGE:
  parallel-dev [OPTIONS] [TASK_IDS...]

OPTIONS:
  -h, --help              Show this help message
  -m, --max-agents <N>    Maximum parallel agents (1-20, default: 5)
  -r, --review <MODE>     Review mode: strict|balanced|lenient (default: balanced)
  -p, --profile <NAME>    Load configuration profile
  -b, --branch <PATTERN>  Branch pattern with {id} placeholder (default: feature/task-{id})
      --status            Show current execution status
      --resume            Resume previous session
      --dry-run           Preview execution without running

EXAMPLES:
  parallel-dev                    # Auto-select available tasks
  parallel-dev 8 9 14            # Develop specific tasks
  parallel-dev 8,9,14            # Comma-separated task IDs
  parallel-dev -m 3 -r strict    # Limit to 3 agents, strict review
  parallel-dev --profile=ci      # Use CI configuration profile
  parallel-dev --status          # Check current execution status
  parallel-dev --resume          # Continue from last session

CONFIGURATION PROFILES:
  conservative  - 2 agents, strict review, extensive testing
  balanced      - 5 agents, balanced review, standard testing
  aggressive    - 10 agents, lenient review, minimal testing
  ci            - Optimized for CI/CD environments
"""
END
```

**ERROR HANDLING:**

```
ON parsing_error:
    DISPLAY: "Error: {error_message}"
    DISPLAY: "Try 'parallel-dev --help' for usage information"
    EXIT: with_error_code(1)
END
```

**EXPORTS:**
- `parse_arguments(args_string)` -> parsed options object
- `show_help()` -> display help text
- `validate_task_ids(ids)` -> validated task ID array