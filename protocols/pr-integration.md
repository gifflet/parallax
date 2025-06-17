**PULL REQUEST INTEGRATION PROTOCOL**

Manages pull request lifecycle from creation through merge with GitHub integration.

**PURPOSE:** Streamline PR creation, monitoring, and merge processes

**DEPENDENCIES:**
- GitHub CLI (`gh`) installed and authenticated
- Repository permissions for PR operations

**PR LIFECYCLE STAGES:**
```
1. CREATION - Generate PR with proper formatting
2. VALIDATION - Ensure PR meets requirements
3. MONITORING - Track PR status and checks
4. REVIEW - Handle review feedback
5. MERGE - Complete integration to base branch
6. CLEANUP - Post-merge resource management
```

**PR CREATION:**

```
FUNCTION create_pull_request(context):
    # Prepare PR metadata
    pr_data = {
        title: format_pr_title(context),
        body: generate_pr_body(context),
        base: context.base_branch || "main",
        head: context.branch_name,
        draft: context.is_wip || false,
        labels: determine_labels(context),
        assignees: context.assignees || [],
        reviewers: context.reviewers || []
    }
    
    # Create PR using gh CLI
    command = build_gh_command(pr_data)
    result = execute_command(command)
    
    # Parse PR details
    pr_info = parse_pr_result(result)
    
    # Add additional metadata
    IF pr_data.labels.length > 0:
        add_pr_labels(pr_info.number, pr_data.labels)
    END
    
    RETURN pr_info
END

FUNCTION format_pr_title(context):
    # Follow conventional commit format
    type = determine_change_type(context.task_spec)
    scope = determine_scope(context.task_spec)
    
    title = "{type}"
    IF scope:
        title += "({scope})"
    END
    title += ": {context.task_spec.title}"
    
    # Add task reference
    title += " (#task-{context.task_id})"
    
    RETURN title
END

FUNCTION generate_pr_body(context):
    template = """
## Summary
{summary}

## What Changed
{changes}

## How to Test
{test_instructions}

## Checklist
- [ ] Tests pass locally
- [ ] Code follows style guidelines  
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No console errors

## Screenshots
{screenshots}

## Performance Impact
{performance}

## Related
- Task: #{task_id}
- Spec: {task_title}

---
<details>
<summary>Technical Details</summary>

### Files Changed
{file_changes}

### Test Coverage
- Coverage: {coverage}%
- Tests added: {tests_added}
- Tests modified: {tests_modified}

### Metrics
- Lines added: {lines_added}
- Lines removed: {lines_removed}
- Complexity change: {complexity_delta}

</details>
"""
    
    # Fill template with actual data
    RETURN fill_template(template, {
        summary: summarize_changes(context),
        changes: list_changes(context),
        test_instructions: generate_test_instructions(context),
        screenshots: context.screenshots || "N/A",
        performance: analyze_performance_impact(context),
        task_id: context.task_id,
        task_title: context.task_spec.title,
        file_changes: format_file_changes(context),
        coverage: context.test_coverage,
        tests_added: count_new_tests(context),
        tests_modified: count_modified_tests(context),
        lines_added: context.stats.additions,
        lines_removed: context.stats.deletions,
        complexity_delta: calculate_complexity_change(context)
    })
END
```

**PR VALIDATION:**

```
FUNCTION validate_pr_requirements(pr_number):
    validation_results = {
        valid: true,
        issues: [],
        warnings: []
    }
    
    pr_data = gh pr view {pr_number} --json
    
    # Check PR title format
    IF not is_valid_conventional_commit(pr_data.title):
        validation_results.issues.push("PR title must follow conventional commit format")
        validation_results.valid = false
    END
    
    # Check PR body completeness
    IF not has_required_sections(pr_data.body):
        validation_results.warnings.push("PR body missing recommended sections")
    END
    
    # Check file changes
    files = gh pr diff {pr_number} --name-only
    IF contains_forbidden_files(files):
        validation_results.issues.push("PR contains forbidden file changes")
        validation_results.valid = false
    END
    
    # Check size limits
    stats = gh pr view {pr_number} --json additions,deletions
    IF stats.additions + stats.deletions > 1000:
        validation_results.warnings.push("Large PR - consider splitting")
    END
    
    RETURN validation_results
END
```

**PR MONITORING:**

```
FUNCTION monitor_pr_status(pr_number, options = {}):
    monitoring_state = {
        pr_number: pr_number,
        start_time: now(),
        check_interval: options.interval || 30000,  # 30 seconds
        max_duration: options.timeout || 3600000,   # 1 hour
        status_history: []
    }
    
    WHILE true:
        current_status = get_pr_status(pr_number)
        monitoring_state.status_history.push({
            timestamp: now(),
            status: current_status
        })
        
        # Check completion conditions
        IF current_status.merged:
            RETURN {
                result: "merged",
                duration: now() - monitoring_state.start_time,
                merge_commit: current_status.merge_commit_sha
            }
        END
        
        IF current_status.closed:
            RETURN {
                result: "closed",
                reason: current_status.close_reason
            }
        END
        
        # Check for required actions
        IF current_status.mergeable_state == "blocked":
            handle_blocked_pr(pr_number, current_status)
        END
        
        # Check timeout
        IF now() - monitoring_state.start_time > monitoring_state.max_duration:
            RETURN {
                result: "timeout",
                last_status: current_status
            }
        END
        
        SLEEP(monitoring_state.check_interval)
    END
END

FUNCTION get_pr_status(pr_number):
    status = gh pr view {pr_number} --json state,mergeable,checks,reviews
    
    RETURN {
        state: status.state,
        merged: status.state == "MERGED",
        closed: status.state == "CLOSED",
        mergeable: status.mergeable,
        mergeable_state: determine_mergeable_state(status),
        checks: parse_check_status(status.checks),
        reviews: parse_review_status(status.reviews),
        conflicts: status.mergeable == false
    }
END
```

**AUTO-MERGE HANDLING:**

```
FUNCTION setup_auto_merge(pr_number, options = {}):
    merge_method = options.method || "squash"  # squash, merge, rebase
    
    # Enable auto-merge
    result = gh pr merge {pr_number} \
        --auto \
        --{merge_method} \
        --delete-branch={options.delete_branch || true}
    
    IF result.success:
        LOG: "Auto-merge enabled for PR #{pr_number}"
        
        # Set up monitoring
        monitor_auto_merge(pr_number)
    ELSE:
        LOG: "Failed to enable auto-merge: " + result.error
        
        # Fallback to manual merge when ready
        setup_merge_when_ready(pr_number, options)
    END
END

FUNCTION monitor_auto_merge(pr_number):
    WHILE true:
        status = get_pr_status(pr_number)
        
        IF status.merged:
            LOG: "PR #{pr_number} auto-merged successfully"
            RETURN true
        END
        
        IF status.state == "CLOSED":
            LOG: "PR #{pr_number} was closed without merging"
            RETURN false
        END
        
        IF not status.auto_merge_enabled:
            LOG: "Auto-merge was disabled for PR #{pr_number}"
            RETURN false
        END
        
        SLEEP(30000)  # Check every 30 seconds
    END
END
```

**REVIEW FEEDBACK HANDLING:**

```
FUNCTION handle_review_feedback(pr_number):
    reviews = gh pr view {pr_number} --json reviews
    
    action_items = []
    
    FOR review IN reviews:
        IF review.state == "CHANGES_REQUESTED":
            comments = get_review_comments(pr_number, review.id)
            
            FOR comment IN comments:
                IF comment.requires_action:
                    action_items.push({
                        file: comment.path,
                        line: comment.line,
                        suggestion: comment.suggestion,
                        reviewer: review.author.login
                    })
                END
            END
        END
    END
    
    IF action_items.length > 0:
        apply_review_suggestions(pr_number, action_items)
    END
    
    RETURN action_items
END

FUNCTION apply_review_suggestions(pr_number, suggestions):
    # Check out PR branch
    gh pr checkout {pr_number}
    
    applied_count = 0
    
    FOR suggestion IN suggestions:
        IF suggestion.suggestion:
            # Apply suggested change
            apply_suggestion_to_file(
                suggestion.file,
                suggestion.line,
                suggestion.suggestion
            )
            applied_count += 1
        END
    END
    
    IF applied_count > 0:
        # Commit changes
        git add -A
        git commit -m "Apply review suggestions\n\nCo-authored-by: reviewers"
        git push
        
        # Comment on PR
        gh pr comment {pr_number} --body "Applied {applied_count} review suggestions"
    END
END
```

**POST-MERGE ACTIONS:**

```
FUNCTION execute_post_merge_actions(pr_number, merge_result):
    # Get PR details
    pr_data = gh pr view {pr_number} --json
    
    # Update task status
    task_id = extract_task_id(pr_data.title)
    IF task_id:
        update_task_master_status(task_id, "merged", {
            pr_number: pr_number,
            merge_commit: merge_result.merge_commit_sha,
            merged_at: iso_timestamp()
        })
    END
    
    # Clean up resources
    IF pr_data.headRefName:
        # Delete remote branch
        git push origin --delete {pr_data.headRefName}
        
        # Clean up local branch and worktree
        cleanup_merged_branch(pr_data.headRefName)
    END
    
    # Notify relevant parties
    notify_merge_complete(pr_number, pr_data)
    
    # Trigger downstream actions
    IF has_downstream_triggers(pr_data):
        trigger_downstream_actions(pr_data)
    END
END
```

**CONFLICT RESOLUTION IN PR:**

```
FUNCTION resolve_pr_conflicts(pr_number):
    # Get PR branch
    pr_data = gh pr view {pr_number} --json headRefName,baseRefName
    
    # Checkout PR branch
    gh pr checkout {pr_number}
    
    # Attempt merge/rebase
    TRY:
        IF use_rebase_workflow():
            git rebase origin/{pr_data.baseRefName}
        ELSE:
            git merge origin/{pr_data.baseRefName}
        END
    CATCH conflict_error:
        # Use merge conflict protocol
        conflicts = detect_conflicts()
        resolution = resolve_all_conflicts(conflicts)
        
        IF resolution.failed.length > 0:
            ERROR: "Could not resolve all conflicts automatically"
        END
        
        # Commit resolution
        git add -A
        git commit -m "Resolve merge conflicts with {pr_data.baseRefName}"
    END
    
    # Push resolved branch
    git push --force-with-lease
    
    # Comment on PR
    gh pr comment {pr_number} --body "Resolved merge conflicts with base branch"
END
```

**EXPORTS:**
- `create_pull_request(context)` -> Create new PR
- `validate_pr_requirements(pr_number)` -> Validate PR meets standards
- `monitor_pr_status(pr_number, options)` -> Track PR progress
- `setup_auto_merge(pr_number, options)` -> Enable auto-merge
- `handle_review_feedback(pr_number)` -> Process review comments
- `resolve_pr_conflicts(pr_number)` -> Fix merge conflicts
- `execute_post_merge_actions(pr_number, result)` -> Post-merge cleanup