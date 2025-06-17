**MERGE CONFLICT RESOLUTION PROTOCOL**

Handles git merge conflicts during development with intelligent resolution strategies.

**PURPOSE:** Resolve merge conflicts efficiently while preserving intended changes

**TRIGGER CONDITIONS:**
- Rebase/merge operations in any agent
- Pull request merge conflicts
- Branch synchronization conflicts

**CONFLICT TYPES:**
```
1. CONTENT CONFLICTS - Different changes to same lines
2. RENAME CONFLICTS - File moved/renamed differently  
3. DELETE CONFLICTS - File deleted vs modified
4. BINARY CONFLICTS - Binary file conflicts
5. SUBMODULE CONFLICTS - Submodule pointer differences
```

**RESOLUTION WORKFLOW:**

**STEP 1: Conflict Detection**
```
FUNCTION detect_conflicts():
    git_status = git status --porcelain
    conflicts = []
    
    FOR line IN git_status:
        IF line.startsWith("UU"):  # Both modified
            conflicts.push({
                type: "content",
                file: line.substring(3)
            })
        ELIF line.startsWith("AA"):  # Both added
            conflicts.push({
                type: "add",
                file: line.substring(3)
            })
        ELIF line.startsWith("DD"):  # Both deleted
            conflicts.push({
                type: "delete", 
                file: line.substring(3)
            })
        ELIF line.startsWith("DU") OR line.startsWith("UD"):
            conflicts.push({
                type: "delete_modify",
                file: line.substring(3)
            })
        END
    END
    
    RETURN conflicts
END
```

**STEP 2: Conflict Analysis**
```
FUNCTION analyze_conflict(conflict):
    analysis = {
        file: conflict.file,
        type: conflict.type,
        our_changes: [],
        their_changes: [],
        conflict_blocks: []
    }
    
    IF conflict.type == "content":
        # Parse conflict markers
        content = read_file(conflict.file)
        blocks = parse_conflict_blocks(content)
        
        FOR block IN blocks:
            analysis.conflict_blocks.push({
                start_line: block.start,
                end_line: block.end,
                our_content: block.ours,
                their_content: block.theirs,
                context: extract_context(block)
            })
        END
    END
    
    # Get change summaries
    analysis.our_changes = git diff --ours {conflict.file}
    analysis.their_changes = git diff --theirs {conflict.file}
    
    RETURN analysis
END
```

**STEP 3: Automatic Resolution Strategies**
```
FUNCTION attempt_auto_resolution(conflict, strategy = "smart"):
    CASE strategy:
        "ours":
            # Keep our version
            git checkout --ours {conflict.file}
            git add {conflict.file}
            RETURN true
            
        "theirs":
            # Keep their version
            git checkout --theirs {conflict.file}
            git add {conflict.file}
            RETURN true
            
        "smart":
            # Try intelligent resolution
            RETURN smart_resolve(conflict)
            
        "union":
            # Keep both changes if possible
            RETURN union_merge(conflict)
    END
END

FUNCTION smart_resolve(conflict):
    # Smart resolution for common patterns
    
    IF is_import_conflict(conflict):
        # Merge both import lists
        resolved = merge_imports(conflict)
        write_file(conflict.file, resolved)
        git add {conflict.file}
        RETURN true
    END
    
    IF is_version_conflict(conflict):
        # Take higher version
        resolved = resolve_version_conflict(conflict)
        write_file(conflict.file, resolved)
        git add {conflict.file}
        RETURN true
    END
    
    IF is_formatting_only(conflict):
        # Run formatter to standardize
        run_formatter(conflict.file)
        git add {conflict.file}
        RETURN true
    END
    
    # Cannot auto-resolve
    RETURN false
END
```

**STEP 4: Interactive Resolution**
```
FUNCTION interactive_resolve(conflict, user_input = null):
    IF not user_input:
        # Display conflict details
        DISPLAY: """
        MERGE CONFLICT: {conflict.file}
        Type: {conflict.type}
        
        Our changes:
        {format_diff(conflict.our_changes)}
        
        Their changes:
        {format_diff(conflict.their_changes)}
        
        Options:
        1. Keep our version
        2. Keep their version
        3. Keep both (if possible)
        4. Custom resolution
        5. Skip this file
        """
        
        RETURN "awaiting_input"
    END
    
    # Process user choice
    CASE user_input.choice:
        1: 
            git checkout --ours {conflict.file}
            git add {conflict.file}
            
        2:
            git checkout --theirs {conflict.file}
            git add {conflict.file}
            
        3:
            IF can_merge_both(conflict):
                merged = merge_both_changes(conflict)
                write_file(conflict.file, merged)
                git add {conflict.file}
            ELSE:
                ERROR: "Cannot automatically merge both changes"
            END
            
        4:
            # Apply custom resolution
            write_file(conflict.file, user_input.custom_content)
            git add {conflict.file}
            
        5:
            # Skip resolution
            RETURN "skipped"
    END
    
    RETURN "resolved"
END
```

**STEP 5: Validation**
```
FUNCTION validate_resolution(file):
    # Check syntax is valid
    IF is_code_file(file):
        syntax_result = check_syntax(file)
        IF not syntax_result.valid:
            WARN: "Syntax error after resolution: " + syntax_result.error
            RETURN false
        END
    END
    
    # Run tests if available
    IF has_tests_for(file):
        test_result = run_related_tests(file)
        IF not test_result.passing:
            WARN: "Tests failing after resolution"
            RETURN false
        END
    END
    
    RETURN true
END
```

**CONFLICT PATTERNS:**

```
FUNCTION is_import_conflict(conflict):
    # Check if conflict is in import/require statements
    patterns = [
        /^import .* from/,
        /^const .* = require/,
        /^using /,
        /^#include/
    ]
    
    FOR block IN conflict.conflict_blocks:
        IF any_pattern_matches(patterns, block.our_content) AND
           any_pattern_matches(patterns, block.their_content):
            RETURN true
        END
    END
    
    RETURN false
END

FUNCTION merge_imports(conflict):
    our_imports = extract_imports(conflict.our_content)
    their_imports = extract_imports(conflict.their_content)
    
    # Combine and deduplicate
    all_imports = unique(our_imports.concat(their_imports))
    
    # Sort by convention
    sorted_imports = sort_imports(all_imports)
    
    # Reconstruct file
    RETURN reconstruct_with_imports(conflict.file, sorted_imports)
END
```

**RESOLUTION RECORD:**
```
FUNCTION record_resolution(conflict, resolution_method, outcome):
    record = {
        file: conflict.file,
        conflict_type: conflict.type,
        resolution_method: resolution_method,
        outcome: outcome,
        timestamp: iso_timestamp(),
        validator_results: validate_resolution(conflict.file)
    }
    
    # Append to resolution log
    append_to_log(".git/conflict-resolutions.log", record)
    
    RETURN record
END
```

**BATCH RESOLUTION:**
```
FUNCTION resolve_all_conflicts(conflicts, strategy = "smart"):
    results = {
        resolved: [],
        failed: [],
        skipped: []
    }
    
    FOR conflict IN conflicts:
        TRY:
            IF attempt_auto_resolution(conflict, strategy):
                results.resolved.push(conflict)
            ELSE:
                # Needs manual resolution
                result = interactive_resolve(conflict)
                IF result == "resolved":
                    results.resolved.push(conflict)
                ELSE:
                    results.skipped.push(conflict)
                END
            END
        CATCH error:
            results.failed.push({
                conflict: conflict,
                error: error.message
            })
        END
    END
    
    RETURN results
END
```

**EXPORTS:**
- `detect_conflicts()` -> List of current conflicts
- `analyze_conflict(conflict)` -> Detailed conflict analysis
- `attempt_auto_resolution(conflict, strategy)` -> Try automatic resolution
- `interactive_resolve(conflict, input)` -> Handle user-driven resolution
- `resolve_all_conflicts(conflicts, strategy)` -> Batch resolution
- `validate_resolution(file)` -> Check if resolution is valid