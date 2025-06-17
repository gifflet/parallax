**SAFE CLEANUP PROTOCOL**

Ensures safe resource cleanup with comprehensive verification to prevent data loss.

**PURPOSE:** Safely remove worktrees and branches only after verifying successful integration

**TRIGGER CONDITIONS:**
- Task marked as completed
- Finalization agent success
- Manual cleanup request
- Session termination

**IMPORTS:**
```
- ../lib/state-manager.md    # For state updates
```

**VERIFICATION WORKFLOW:**

**STEP 1: Identify Resources**
```
FUNCTION identify_cleanup_candidates(task_id):
    resources = {
        worktree: ".worktrees/task-" + task_id,
        branch: "feature/task-" + task_id,
        temp_files: [],
        pr_number: null
    }
    
    # Get additional info from state
    task_state = get_task_state(task_id)
    IF task_state.pr_number:
        resources.pr_number = task_state.pr_number
    END
    
    # Check for temp files
    temp_patterns = [
        "*.tmp",
        "*.log",
        ".*.swp"
    ]
    
    FOR pattern IN temp_patterns:
        files = find_files(resources.worktree, pattern)
        resources.temp_files.extend(files)
    END
    
    RETURN resources
END
```

**STEP 2: Merge Verification**
```
FUNCTION verify_merge_status(branch_name):
    # Identify main branch
    main_branch = detect_main_branch()  # main, master, or develop
    
    # Update branch info
    git fetch origin
    
    # Check if branch is merged
    merged_branches = git branch --merged {main_branch}
    is_merged = branch_name IN merged_branches
    
    # Additional verification
    IF is_merged:
        # Verify all commits are in main
        commits_not_in_main = git log {main_branch}..{branch_name} --oneline
        IF commits_not_in_main.length > 0:
            is_merged = false
            reason = "Commits not in main: " + commits_not_in_main
        END
    END
    
    RETURN {
        merged: is_merged,
        main_branch: main_branch,
        reason: reason || "Branch fully merged"
    }
END
```

**STEP 3: Worktree Status Check**
```
FUNCTION check_worktree_status(worktree_path):
    status = {
        exists: directory_exists(worktree_path),
        has_changes: false,
        uncommitted_files: [],
        untracked_files: []
    }
    
    IF not status.exists:
        RETURN status
    END
    
    # Check for uncommitted changes
    cd {worktree_path}
    git_status = git status --porcelain
    
    FOR line IN git_status:
        IF line.startsWith("M ") OR line.startsWith("A "):
            status.has_changes = true
            status.uncommitted_files.push(line)
        ELIF line.startsWith("??"):
            status.untracked_files.push(line)
        END
    END
    
    RETURN status
END
```

**STEP 4: Cleanup Decision Matrix**
```
FUNCTION determine_cleanup_action(task_id, resources):
    merge_status = verify_merge_status(resources.branch)
    worktree_status = check_worktree_status(resources.worktree)
    
    decision = {
        action: "none",
        reason: "",
        safe_to_cleanup: false,
        warnings: []
    }
    
    # Scenario A: Fully merged and clean
    IF merge_status.merged AND not worktree_status.has_changes:
        decision.action = "full_cleanup"
        decision.reason = "Branch merged and worktree clean"
        decision.safe_to_cleanup = true
        
    # Scenario B: Merged but has uncommitted changes
    ELIF merge_status.merged AND worktree_status.has_changes:
        decision.action = "preserve"
        decision.reason = "Uncommitted changes in worktree"
        decision.warnings.push("Worktree has uncommitted changes")
        
    # Scenario C: Not merged
    ELIF not merge_status.merged:
        decision.action = "preserve"
        decision.reason = merge_status.reason
        
        # Check if PR exists and is open
        IF resources.pr_number:
            pr_status = check_pr_status(resources.pr_number)
            IF pr_status.state == "open":
                decision.warnings.push("PR #" + resources.pr_number + " is still open")
            END
        END
    END
    
    # Add warnings for untracked files
    IF worktree_status.untracked_files.length > 0:
        decision.warnings.push("Untracked files: " + worktree_status.untracked_files.length)
    END
    
    RETURN decision
END
```

**STEP 5: Execute Cleanup**
```
FUNCTION execute_cleanup(task_id, resources, decision):
    cleanup_log = {
        task_id: task_id,
        timestamp: iso_timestamp(),
        decision: decision,
        actions_taken: [],
        errors: []
    }
    
    IF decision.action == "full_cleanup":
        # Remove worktree
        IF directory_exists(resources.worktree):
            TRY:
                git worktree remove {resources.worktree}
                cleanup_log.actions_taken.push("Removed worktree: " + resources.worktree)
            CATCH error:
                cleanup_log.errors.push("Failed to remove worktree: " + error)
            END
        END
        
        # Delete local branch
        TRY:
            git branch -d {resources.branch}
            cleanup_log.actions_taken.push("Deleted local branch: " + resources.branch)
        CATCH error:
            # Force delete if needed (already merged)
            git branch -D {resources.branch}
            cleanup_log.actions_taken.push("Force deleted local branch: " + resources.branch)
        END
        
        # Delete remote branch
        TRY:
            git push origin --delete {resources.branch}
            cleanup_log.actions_taken.push("Deleted remote branch: " + resources.branch)
        CATCH error:
            # Remote might already be deleted
            cleanup_log.actions_taken.push("Remote branch already deleted or inaccessible")
        END
        
        # Clean temp files
        FOR file IN resources.temp_files:
            remove_file(file)
            cleanup_log.actions_taken.push("Removed temp file: " + file)
        END
        
    ELIF decision.action == "preserve":
        cleanup_log.actions_taken.push("PRESERVED: " + decision.reason)
        
        # Create preservation record
        preservation = {
            task_id: task_id,
            preserved_at: iso_timestamp(),
            reason: decision.reason,
            warnings: decision.warnings,
            resources: resources
        }
        
        save_preservation_record(preservation)
    END
    
    # Update task state
    update_task_cleanup_status(task_id, cleanup_log)
    
    RETURN cleanup_log
END
```

**PRESERVATION HANDLING:**
```
FUNCTION save_preservation_record(record):
    file = ".worktrees/.preserved-tasks.json"
    
    # Load existing records
    records = []
    IF file_exists(file):
        records = load_json(file)
    END
    
    # Add new record
    records.push(record)
    
    # Save
    save_json(file, records)
    
    # Log for visibility
    LOG: "PRESERVED TASK " + record.task_id + ": " + record.reason
END

FUNCTION list_preserved_resources():
    file = ".worktrees/.preserved-tasks.json"
    
    IF not file_exists(file):
        RETURN []
    END
    
    records = load_json(file)
    
    FOR record IN records:
        DISPLAY: "Task " + record.task_id + " preserved at " + record.preserved_at
        DISPLAY: "  Reason: " + record.reason
        DISPLAY: "  Worktree: " + record.resources.worktree
        DISPLAY: "  Branch: " + record.resources.branch
        
        IF record.warnings.length > 0:
            DISPLAY: "  Warnings:"
            FOR warning IN record.warnings:
                DISPLAY: "    - " + warning
            END
        END
    END
    
    RETURN records
END
```

**ERROR RECOVERY:**
```
ON cleanup_error:
    # Never lose data due to cleanup errors
    LOG: "Cleanup error for task " + task_id + ": " + error
    
    # Mark as needing manual intervention
    mark_for_manual_cleanup(task_id, error)
    
    # Continue with other tasks
    CONTINUE
END
```

**EXPORTS:**
- `execute_safe_cleanup(task_id)` -> Perform safe cleanup with verification
- `list_preserved_resources()` -> Show preserved tasks
- `force_cleanup(task_id)` -> Override safety checks (requires confirmation)
- `check_cleanup_safety(task_id)` -> Preview cleanup decision without executing