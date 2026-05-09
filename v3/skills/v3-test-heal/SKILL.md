# Skill: /v3-test-heal

**Purpose**: Self-healing loop that runs the generated test against a real cluster, analyzes failures from logs, consults documentation and repository code to fix issues, and records lessons learned in GRAVEYARD.md

## Input
- **Required**: Path to the generated test file (e.g., `tests/virt/cluster/test_vnc_screenshot.py`)
- **Optional**: `--max-iterations N` - Maximum heal attempts (default: 3)

## Output
- **Modified File**: Test file edited in-place with fixes applied
- **GRAVEYARD.md**: Updated (or created) at repository root with documented mistakes and lessons
- **Exit Code**: 0 if test passes, 1 if max iterations reached with failures
- **Log**: Summary of failures encountered and fixes applied

## How Tests Are Run

Tests are executed against a live OpenShift cluster by running the following command:

```bash
../runtstqe <test_file_path>
```

This is a wrapper script that sets up the cluster environment and runs pytest. Its contents are:

```bash
#!/bin/bash
# Run tests on cluster
# KUBECONFIG and Artifactory credentials are expected to be set externally

uv run pytest "$1" \
  --skip-artifactory-check \
  --tc=conformance_tests:True
```

It expects `KUBECONFIG` and Artifactory credentials to be set in the environment before invocation, then runs `uv run pytest` with `--skip-artifactory-check` and `--tc=conformance_tests:True` flags.

## Implementation

### Self-Healing Loop

```
1. RUN test via: ../runtstqe <test_file>
2. CAPTURE full output (stdout + stderr) as test_run.log
3. IF all tests pass (exit code 0):
     UPDATE GRAVEYARD.md (if any fixes were applied in previous iterations)
     RETURN success
4. ELSE:
     a. PARSE test output for failures:
        - Collect traceback for each failed test
        - Identify error type (ImportError, AttributeError, AssertionError, TimeoutError, etc.)
        - Extract the failing line and context
     b. FOR each failure:
        - ANALYZE root cause from traceback
        - CONSULT documentation and repository code:
          * Read relevant source files referenced in traceback
          * Search docs/ for correct API usage
          * Search utilities/ and libs/ for correct method signatures
          * Search existing tests for working examples of similar patterns
          * Check upstream documentation if needed (KubeVirt, CDI, HCO)
        - DETERMINE fix strategy
        - APPLY fix via Edit tool
        - DOCUMENT the mistake in memory for GRAVEYARD.md
     c. INCREMENT iteration counter
     d. IF iteration >= max_iterations:
          UPDATE GRAVEYARD.md with all documented mistakes
          RETURN failure with remaining errors
     e. GOTO step 1 (re-run test)
```

### Failure Categories & Fix Strategies

#### 1. **ImportError / ModuleNotFoundError**
```
Error: ModuleNotFoundError: No module named 'utilities.foo'
Fix Strategy:
  1. SEARCH repository for the correct module path
  2. UPDATE import statement
  3. If module doesn't exist, find alternative or implement locally
```

#### 2. **AttributeError**
```
Error: 'VirtualMachine' object has no attribute 'restart'
Fix Strategy:
  1. READ the actual class definition to find correct method name
  2. SEARCH existing tests for how this operation is performed
  3. UPDATE code to use correct attribute/method
```

#### 3. **TypeError (wrong arguments)**
```
Error: __init__() got an unexpected keyword argument 'cpu_cores'
Fix Strategy:
  1. READ the actual constructor/method signature
  2. FIND correct parameter name
  3. UPDATE function call
```

#### 4. **AssertionError (wrong test logic)**
```
Error: AssertionError: assert 'Stopped' == 'Running'
Fix Strategy:
  1. ANALYZE whether the assertion expectation is wrong or the setup is wrong
  2. CHECK if a wait/poll is needed before assertion
  3. CHECK if the precondition setup is correct
  4. FIX the root cause (not just the assertion)
```

#### 5. **TimeoutError / WaitTimeout**
```
Error: TimeoutExpiredError: waited 300s for VM to be Running
Fix Strategy:
  1. CHECK if timeout value is sufficient
  2. CHECK if the wait condition is correct
  3. CHECK if the resource was created/configured correctly
  4. INCREASE timeout or fix the underlying issue
```

#### 6. **Fixture Errors**
```
Error: fixture 'vm_for_test' not found
Fix Strategy:
  1. CHECK conftest.py for fixture definition
  2. CHECK if fixture name matches exactly
  3. CHECK fixture scope and placement
  4. ADD or FIX fixture definition
```

#### 7. **Resource/API Errors**
```
Error: NotFoundError: virtualmachines.kubevirt.io "test-vm" not found
Fix Strategy:
  1. CHECK if resource was created in correct namespace
  2. CHECK if resource name matches
  3. CHECK if API group/version is correct
  4. CONSULT KubeVirt docs for correct resource specification
```

## Algorithm

```python
iteration = 0
max_iterations = args.max_iterations ?? 3
graveyard_entries = []

while iteration < max_iterations:
    # Run test against real cluster
    result = bash(f"../runtstqe {test_file} 2>&1 | tee test_run.log")

    if result.exit_code == 0:
        log(f"✓ All tests pass after {iteration} iterations")
        if graveyard_entries:
            update_graveyard(graveyard_entries)
        return SUCCESS

    # Parse failures from output
    failures = parse_pytest_failures(result.output)
    log(f"  Found {len(failures)} failure(s) in iteration {iteration}")

    for failure in failures:
        # Read context around failing code
        file_content = read_file(test_file,
                                  offset=failure.line - 10,
                                  limit=20)

        # Consult docs and repository for correct approach
        docs = search_documentation(failure)
        examples = find_similar_working_tests(failure)
        source = read_source_definition(failure)

        # Determine fix
        fix = analyze_and_generate_fix(failure, file_content, docs, examples, source)

        # Apply fix
        edit_file(test_file,
                  old_string=failure.context,
                  new_string=fix.new_code)

        # Record for GRAVEYARD
        graveyard_entries.append({
            "mistake": failure.description,
            "category": failure.category,
            "wrong_code": failure.context,
            "correct_code": fix.new_code,
            "lesson": fix.lesson_learned,
            "how_to_avoid": fix.avoidance_strategy
        })

        log(f"  Fix {iteration}.{failure.id}: {fix.description}")

    iteration++

# Max iterations reached - still update GRAVEYARD
if graveyard_entries:
    update_graveyard(graveyard_entries)

log(f"✗ Tests still failing after {max_iterations} iterations")
return FAILURE
```

## GRAVEYARD.md Format

The GRAVEYARD.md file is maintained at the repository root. When creating or updating it, you **MUST** follow this exact format. See `GRAVEYARD.md` in the repository root for existing entries.

### File Header (create only if file does not exist)

```markdown
# GRAVEYARD - Test Generation Lessons Learned

This file documents mistakes made during automated test generation and how to avoid them.
It is read during the exploration phase to prevent repeating past errors.

---
```

### Entry Format (append one per mistake)

```markdown
## Entry: [Short description of mistake]

**Category**: [ImportError | AttributeError | TypeError | AssertionError | TimeoutError | FixtureError | ResourceError | LogicError]
**Date**: [YYYY-MM-DD]
**Test**: [path/to/test_file.py::test_name]

### What Went Wrong
[Description of the mistake - what did the code do wrong?]

### Wrong Code
```python
[the incorrect code that was generated]
```

### Correct Code
```python
[the fixed code]
```

### Lesson Learned
[What should be done differently - the insight gained]

### How to Avoid
[Specific, actionable rule for future code generation]

---
```

### Format Rules

- Every section (`What Went Wrong`, `Wrong Code`, `Correct Code`, `Lesson Learned`, `How to Avoid`) is **mandatory** for each entry
- `Wrong Code` and `Correct Code` must contain actual code snippets inside python fenced code blocks
- `How to Avoid` must be a concrete, actionable rule — not a vague suggestion
- Each entry ends with `---` as a separator

### GRAVEYARD.md Update Rules

1. **Append only**: Never remove existing entries - they are historical records
2. **Deduplicate**: If the same mistake category and root cause already exists, do NOT add a duplicate. Instead, add a note to the existing entry if there's new information.
3. **Be specific**: Include actual code snippets, not vague descriptions
4. **Actionable lessons**: The "How to Avoid" section must contain a concrete, actionable rule
5. **Create if missing**: If GRAVEYARD.md doesn't exist, create it with the header

## Consulting Documentation

When analyzing failures, consult these sources in order:

1. **Repository code** (highest priority):
   - Read the actual class/method definition from the traceback
   - Search `utilities/` and `libs/` for correct patterns
   - Check existing tests for working examples

2. **Repository docs** (`docs/`):
   - Testing guidelines
   - Fixture documentation
   - API usage patterns

3. **Upstream documentation** (if repository code is insufficient):
   - KubeVirt API docs: https://kubevirt.io/api-reference/
   - CDI docs: https://github.com/kubevirt/containerized-data-importer
   - OpenShift docs: https://docs.redhat.com/en/documentation/openshift_container_platform/

## Constraints

### 1. Fix Root Causes
- Don't suppress errors (no `try/except: pass`)
- Don't weaken assertions to make tests pass
- Find and fix the actual problem

### 2. Preserve Test Intent
- Fixes must maintain the original test purpose
- If a test step is fundamentally wrong, fix the approach, not the expectation
- The STD docstring defines what the test SHOULD do

### 3. Consult Before Fixing
- Always read the relevant source code before applying a fix
- Don't guess method signatures - look them up
- Use existing working tests as reference

### 4. Document Everything
- Every fix becomes a GRAVEYARD entry
- Lessons must be actionable and specific
- Future runs will benefit from documented mistakes

### 5. Iteration Limits
- Default: 3 iterations maximum
- Each iteration should fix multiple failures
- If stuck in a loop (same errors recurring), stop and report

## Usage

```bash
# Standard usage - run and heal test
/v3-test-heal tests/virt/cluster/test_vnc_screenshot.py

# With custom iteration limit
/v3-test-heal tests/compute/test_cpu_migration.py --max-iterations 5

# The skill will:
# 1. Run: ../runtstqe tests/virt/cluster/test_vnc_screenshot.py
# 2. Parse failures
# 3. Fix code
# 4. Re-run until pass or max iterations
# 5. Update GRAVEYARD.md
```

## Reusability
**HIGH** - Can heal any test file in the repository against the live cluster

## Dependencies
- Bash tool (run tests via `../runtstqe`, capture output)
- Read tool (read test file, source code, docs)
- Edit tool (apply fixes)
- Write tool (create/update GRAVEYARD.md)
- Glob tool (find related files)
- Grep tool (search for patterns)
