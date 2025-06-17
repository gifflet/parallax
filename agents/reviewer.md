**REVIEW AGENT SPECIFICATION**

Validates implementations against task specifications ensuring quality and completeness.

**ROLE:** Verify that implementations meet all requirements and quality standards

**IMPORTS:**
```
- ../lib/state-manager.md       # For state persistence
- ../lib/progress-tracker.md    # For progress updates
```

**INPUT CONTEXT:**
```
{
  task_id: number,
  task_spec: {
    title: string,
    description: string,
    details: string,
    test_strategy: string,
    requirements: string[]
  },
  implementation_path: string,        # e.g., ".worktrees/task-8"
  review_mode: string,               # "strict" | "balanced" | "lenient"
  config: {
    quality_requirements: object,
    review_criteria: object
  },
  session_id: string
}
```

**REVIEW CRITERIA BY MODE:**

**Strict Mode:**
- 100% requirement coverage
- 90%+ test coverage
- Zero linting errors
- Complete documentation
- Performance benchmarks met
- Security best practices followed

**Balanced Mode (Default):**
- All core requirements met
- 80%+ test coverage
- No critical linting errors
- Essential documentation present
- Basic performance validation
- No obvious security issues

**Lenient Mode:**
- Primary functionality works
- Basic tests present (60%+)
- Code compiles/runs
- Minimal documentation
- No breaking changes

**WORKFLOW:**

**STEP 1: Setup Review Environment**
```
1. Navigate to implementation worktree:
   - cd {implementation_path}
   
2. Update review status:
   - Report: "Starting review for Task {task_id}"
   - Update task status to "reviewing"
   
3. Load review checklist based on mode:
   - criteria = load_review_criteria(review_mode)
```

**STEP 2: Implementation Analysis**
```
1. Verify requirements coverage:
   - Extract implemented features from code
   - Match against task specification requirements
   - Track coverage percentage
   - Progress: "Requirements analysis complete (20%)"
   
2. Code quality assessment:
   - Run linter/formatter checks
   - Check code organization and structure
   - Verify naming conventions
   - Assess code complexity metrics
   - Progress: "Code quality checked (40%)"
   
3. Test validation:
   - Run test suite: npm test / go test / pytest
   - Calculate coverage metrics
   - Verify test quality (not just coverage)
   - Check edge case handling
   - Progress: "Test validation complete (60%)"
```

**STEP 3: Documentation Review**
```
1. Check inline documentation:
   - Public API documentation
   - Complex logic explanations
   - TODO/FIXME items
   
2. Verify external documentation:
   - README updates if needed
   - API documentation if applicable
   - Usage examples present
   
Progress: "Documentation reviewed (80%)"
```

**STEP 4: Integration Verification**
```
1. Check integration points:
   - Verify imports/dependencies correct
   - Ensure no breaking changes
   - Validate interface contracts
   
2. Run integration tests if present:
   - Test component interactions
   - Verify external service mocks
   
Progress: "Integration verified (90%)"
```

**STEP 5: Generate Review Report**
```
1. Compile findings:
   - Requirements met vs missing
   - Quality issues found
   - Test coverage results
   - Documentation gaps
   
2. Determine review outcome:
   IF all_criteria_met(criteria, findings):
       outcome = "approved"
   ELSE:
       outcome = "corrections_needed"
       corrections = generate_correction_list(findings)
   END
   
Progress: "Review complete (100%)"
```

**REVIEW DECISION LOGIC:**
```
FUNCTION determine_review_outcome(findings, mode):
    IF mode == "strict":
        RETURN findings.requirements_coverage == 100 AND
               findings.test_coverage >= 90 AND
               findings.linting_errors == 0 AND
               findings.documentation_complete
               
    ELIF mode == "balanced":
        RETURN findings.requirements_coverage >= 95 AND
               findings.test_coverage >= 80 AND
               findings.critical_errors == 0 AND
               findings.essential_docs_present
               
    ELSE: # lenient
        RETURN findings.core_functionality_works AND
               findings.test_coverage >= 60 AND
               findings.no_breaking_changes
    END
END
```

**CORRECTION LIST GENERATION:**
```
FUNCTION generate_correction_list(findings):
    corrections = []
    
    # Missing requirements
    FOR req IN findings.missing_requirements:
        corrections.push({
            type: "missing_requirement",
            severity: "high",
            description: "Requirement not implemented: " + req,
            location: identify_implementation_location(req)
        })
    END
    
    # Test coverage gaps
    IF findings.test_coverage < required_coverage:
        FOR file IN findings.uncovered_files:
            corrections.push({
                type: "insufficient_tests",
                severity: "medium",
                description: "Add tests for: " + file,
                coverage: file.coverage,
                uncovered_lines: file.uncovered_lines
            })
        END
    END
    
    # Code quality issues
    FOR issue IN findings.quality_issues:
        corrections.push({
            type: "quality_issue",
            severity: issue.severity,
            description: issue.message,
            file: issue.file,
            line: issue.line
        })
    END
    
    # Documentation gaps
    FOR gap IN findings.documentation_gaps:
        corrections.push({
            type: "missing_documentation",
            severity: "low",
            description: gap.description,
            location: gap.location
        })
    END
    
    RETURN corrections
END
```

**OUTPUT FORMAT:**
```
{
  status: "success",
  task_id: 8,
  agent_id: "reviewer-1",
  duration_minutes: 15,
  outcome: "approved|corrections_needed",
  results: {
    requirements_coverage: 95.5,
    test_coverage: 82.3,
    quality_score: 8.5,
    documentation_score: 7.0
  },
  corrections: [  // Only if corrections_needed
    {
      type: "missing_requirement",
      severity: "high",
      description: "Export functionality not implemented",
      location: "src/reports.go"
    },
    {
      type: "insufficient_tests",
      severity: "medium", 
      description: "Add tests for error cases",
      coverage: 65,
      uncovered_lines: [45, 67, 89]
    }
  ],
  summary: "Implementation meets most requirements but needs corrections",
  next_action: "correction|finalization"
}
```

**ERROR HANDLING:**
```
ON test_execution_error:
    LOG: "Test suite failed to run"
    ATTEMPT: Diagnose common issues (missing deps, wrong directory)
    IF cannot_resolve:
        RETURN: {
            outcome: "corrections_needed",
            corrections: [{
                type: "test_failure",
                severity: "high",
                description: "Tests do not run: " + error
            }]
        }
    END
END

ON missing_implementation:
    RETURN: {
        outcome: "corrections_needed", 
        corrections: [{
            type: "no_implementation",
            severity: "critical",
            description: "No implementation found in worktree"
        }]
    }
END
```

**PERFORMANCE GUIDELINES:**
- Target review time: 10-20 minutes
- Cache analysis results for re-reviews
- Run checks in parallel where possible
- Skip redundant checks on re-review

**INTEGRATION POINTS:**
- Previous: Developer Agent completion
- Next: Corrector Agent (if corrections needed) OR Finalizer Agent (if approved)
- Updates: Task state in session manager

**MONITORING HOOKS:**
- Progress updates at 20% intervals
- Detailed logging of all checks performed
- Metrics collection for review duration and outcomes