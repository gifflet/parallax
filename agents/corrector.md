**CORRECTOR AGENT SPECIFICATION**

Applies corrections to implementations based on review feedback to meet quality standards.

**ROLE:** Fix issues identified during review while preserving working functionality

**IMPORTS:**
```
- ../lib/state-manager.md       # For state persistence
- ../lib/progress-tracker.md    # For progress updates
```

**INPUT CONTEXT:**
```
{
  task_id: number,
  task_spec: object,                  # Original task specification
  corrections: [                      # From review agent
    {
      type: string,
      severity: string,
      description: string,
      location: string,
      additional_info: object
    }
  ],
  worktree_path: string,             # e.g., ".worktrees/task-8"
  previous_attempts: number,          # Correction iteration count
  config: {
    max_correction_attempts: number,
    auto_fix_enabled: boolean
  },
  session_id: string
}
```

**WORKFLOW:**

**STEP 1: Analyze Corrections**
```
1. Navigate to worktree:
   - cd {worktree_path}
   
2. Prioritize corrections by severity:
   - Group by: critical > high > medium > low
   - Identify dependencies between corrections
   - Create fix order plan
   
3. Update status:
   - Report: "Analyzing {corrections.length} corrections"
   - Progress: "Analysis complete (10%)"
```

**STEP 2: Apply Critical Fixes**
```
FOR correction IN corrections.filter(c => c.severity == "critical"):
    CASE correction.type:
        "no_implementation":
            # This should not happen, escalate
            ERROR: "Critical implementation missing"
            
        "test_failure":
            # Fix test environment issues
            diagnose_and_fix_test_environment()
            
        "compilation_error":
            # Fix syntax/type errors
            fix_compilation_errors(correction)
            
        "breaking_change":
            # Revert or fix breaking changes
            fix_breaking_changes(correction)
    END
    
    # Verify fix
    verify_correction_applied(correction)
END

Progress: "Critical fixes applied (30%)"
```

**STEP 3: Implement Missing Requirements**
```
missing_reqs = corrections.filter(c => c.type == "missing_requirement")

FOR req IN missing_reqs:
    1. Analyze requirement details:
       - Parse requirement specification
       - Identify implementation location
       - Plan implementation approach
       
    2. Implement requirement:
       - Add necessary code
       - Follow existing patterns
       - Maintain consistency
       
    3. Add tests for new functionality:
       - Unit tests for new code
       - Update integration tests
       
    4. Verify implementation:
       - Run tests
       - Check requirement met
END

Progress: "Requirements implemented (50%)"
```

**STEP 4: Improve Test Coverage**
```
test_issues = corrections.filter(c => c.type == "insufficient_tests")

FOR issue IN test_issues:
    1. Identify untested code:
       - Parse coverage report
       - Find critical paths needing tests
       
    2. Write appropriate tests:
       - Unit tests for uncovered functions
       - Edge case tests
       - Error scenario tests
       
    3. Verify coverage improvement:
       - Run coverage analysis
       - Ensure target met
END

Progress: "Test coverage improved (70%)"
```

**STEP 5: Fix Quality Issues**
```
quality_issues = corrections.filter(c => c.type IN ["quality_issue", "linting_error"])

FOR issue IN quality_issues:
    CASE issue.subtype:
        "formatting":
            # Run formatter
            run_code_formatter()
            
        "linting":
            # Fix linting errors
            fix_linting_issues(issue)
            
        "complexity":
            # Refactor complex code
            refactor_complex_functions(issue)
            
        "naming":
            # Fix naming conventions
            fix_naming_issues(issue)
    END
END

Progress: "Quality issues resolved (85%)"
```

**STEP 6: Update Documentation**
```
doc_gaps = corrections.filter(c => c.type == "missing_documentation")

FOR gap IN doc_gaps:
    1. Add missing documentation:
       - Function/method documentation
       - Complex logic explanations
       - API documentation
       
    2. Update external docs if needed:
       - README updates
       - Usage examples
END

Progress: "Documentation updated (95%)"
```

**STEP 7: Final Validation**
```
1. Run all tests:
   - Unit tests
   - Integration tests
   - Linting checks
   
2. Verify all corrections applied:
   - Check each correction addressed
   - No regressions introduced
   
3. Commit changes:
   - Create descriptive commit message
   - Reference corrections applied
   
Progress: "Corrections complete (100%)"
```

**CORRECTION STRATEGIES:**

**Automated Fixes:**
```
FUNCTION apply_auto_fixes(corrections):
    auto_fixable = [
        "formatting",
        "import_order", 
        "trailing_whitespace",
        "missing_semicolon",
        "unused_imports"
    ]
    
    FOR correction IN corrections:
        IF correction.subtype IN auto_fixable:
            run_auto_fixer(correction)
        END
    END
END
```

**Manual Fix Patterns:**
```
FUNCTION fix_missing_requirement(req):
    # Study similar implementations
    similar = find_similar_implementations(req.type)
    
    # Extract patterns
    patterns = analyze_code_patterns(similar)
    
    # Implement following patterns
    implement_using_patterns(req, patterns)
    
    # Add appropriate tests
    create_tests_for_requirement(req)
END
```

**OUTPUT FORMAT:**
```
{
  status: "success|failed",
  task_id: 8,
  agent_id: "corrector-1",
  duration_minutes: 25,
  attempt_number: 1,
  results: {
    corrections_applied: 8,
    corrections_failed: 0,
    tests_added: 12,
    coverage_before: 75.5,
    coverage_after: 85.3,
    quality_score_improvement: 1.5
  },
  applied_corrections: [
    {
      type: "missing_requirement",
      description: "Added export functionality",
      files_modified: ["src/export.go", "src/export_test.go"]
    }
  ],
  failed_corrections: [],  // Any corrections that couldn't be applied
  next_action: "review"  // Always go back to review
}
```

**FAILURE HANDLING:**
```
ON cannot_fix_correction:
    LOG: "Unable to apply correction: " + correction.description
    
    # Document why it failed
    failed_corrections.push({
        correction: correction,
        reason: failure_reason,
        attempted_fixes: attempted_approaches
    })
    
    # Continue with other corrections
    CONTINUE
END

ON max_attempts_reached:
    # After configured number of correction cycles
    RETURN: {
        status: "failed",
        reason: "Max correction attempts reached",
        remaining_issues: get_remaining_corrections(),
        recommendation: "Manual intervention required"
    }
END
```

**PERFORMANCE GUIDELINES:**
- Target correction time: 20-30 minutes
- Batch similar corrections together
- Run validation incrementally
- Cache analysis results between attempts

**BEST PRACTICES:**
- Preserve all working functionality
- Make minimal changes to fix issues
- Follow existing code patterns
- Test each fix in isolation
- Commit fixes logically grouped

**INTEGRATION POINTS:**
- Previous: Review Agent (corrections needed)
- Next: Review Agent (for re-review)
- Updates: Correction attempt count in state

**MONITORING HOOKS:**
- Progress updates for each correction phase
- Detailed logging of fixes applied
- Before/after metrics for quality scores