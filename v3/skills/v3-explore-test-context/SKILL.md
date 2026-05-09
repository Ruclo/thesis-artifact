# Skill: /v3-explore-test-context

**Purpose**: Discover repository-specific testing patterns, utilities, fixtures, conventions, and learn from past mistakes documented in GRAVEYARD.md

## Input
- **Required**: Repository root (implicit - current working directory)

## Output
- **No file artifacts** - findings are reported in conversation and used for subsequent code generation
- **Format**: Verbal summary of discovered patterns, utilities, fixtures, conventions, and past mistakes

## Role
You are exploring the test repository to understand its structure, patterns, and available utilities before generating test code. You also review the GRAVEYARD.md to learn from mistakes made in previous test generation attempts.

## Exploration Phases

### Phase 1: Scan Documentation
Read `docs/` to understand testing conventions:
- Testing guidelines and best practices
- Pytest marker usage
- Fixture conventions
- Coding standards

**Report findings:**
- Key conventions documented
- Marker patterns used
- Special testing requirements

### Phase 2: Map Utilities
Analyze `libs/` and `utilities/` to discover available helpers:
- Helper classes and their methods
- Method signatures and parameters
- Constants (TIMEOUT_*, DEFAULT_*, etc.)
- Common patterns and utilities

**Report findings:**
- Available utility classes (e.g., VirtualMachineForTests, DataVolumeForTests)
- Commonly used methods (e.g., wait_for_vm_running, create_snapshot)
- Important constants (e.g., TIMEOUT_5MIN, DEFAULT_STORAGE_CLASS)

### Phase 3: Check Precedents
Look at existing tests in `tests/` to see valid patterns:
- Import patterns and structure
- Fixture usage examples
- Test class organization
- Assertion patterns
- Common test patterns

**Report findings:**
- Typical import statements
- Common fixtures used (e.g., namespace, admin_client, storage_class_matrix)
- Test file structure conventions
- Naming conventions (test_<feature>_<scenario>)

### Phase 4: Review GRAVEYARD.md (Past Mistakes)

**This phase is critical for avoiding repeated mistakes.**

Read `GRAVEYARD.md` at the repository root (if it exists) to learn from past test generation failures:
- Common mistakes made during code generation
- Patterns that caused test failures at runtime
- Incorrect API usage or wrong method signatures
- Fixture misuse or missing fixtures
- Import errors that weren't caught by pyright
- Runtime errors (e.g., wrong resource names, incorrect wait conditions, bad assertions)
- Lessons learned and how to avoid each mistake

**Report findings:**
- List of documented mistakes and their categories
- Specific patterns to AVOID
- Correct alternatives for each mistake
- Any recurring themes (e.g., "always use X instead of Y")

**If GRAVEYARD.md does not exist:** Report that no past mistakes are documented yet. This is expected for the first run.

## Exploration Algorithm

```
1. READ docs/testing_guidelines.md (if exists)
   - Extract conventions
   - Note marker patterns

2. GLOB utilities/*.py
   FOR each utility_file:
     - READ file
     - EXTRACT class names
     - EXTRACT method signatures
     - EXTRACT constants
     - SUMMARIZE findings

3. GLOB tests/**/*.py (sample 5-10 files)
   FOR each test_file:
     - READ file
     - EXTRACT import patterns
     - EXTRACT fixture usage
     - NOTE test structure
     - SUMMARIZE patterns

4. READ GRAVEYARD.md (if exists)
   - EXTRACT all documented mistakes
   - EXTRACT lessons learned
   - EXTRACT patterns to avoid
   - SUMMARIZE as "DO NOT" rules for code generation

5. REPORT comprehensive findings:
   - Available utilities summary
   - Common fixtures summary
   - Import patterns summary
   - Testing conventions summary
   - GRAVEYARD lessons (mistakes to avoid)
```

## Example Output

After exploration, provide a summary like:

```
Repository Exploration Complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Documentation Findings:
  - Tests use @pytest.mark.polarion() for test case IDs
  - Architecture markers: @pytest.mark.arm64, @pytest.mark.s390x
  - Tiering: @pytest.mark.tier1, @pytest.mark.tier2

Available Utilities:
  Classes:
    - VirtualMachineForTests (from utilities.virt)
    - DataVolumeForTests (from utilities.storage)

  Methods:
    - wait_for_vm_running(vm, timeout=300)
    - create_snapshot(vm, snapshot_name)
    - ssh_to_vm(vm, command)

  Constants:
    - TIMEOUT_5MIN = 300
    - TIMEOUT_10MIN = 600
    - DEFAULT_STORAGE_CLASS = "ocs-storagecluster-ceph-rbd"

Common Fixtures:
  - namespace (function scope) - provides isolated test namespace
  - admin_client (session scope) - provides cluster admin client
  - storage_class_matrix (parametrized) - tests across storage classes

Import Patterns:
  - from ocp_resources.virtual_machine import VirtualMachine
  - from utilities.virt import VirtualMachineForTests
  - from utilities.constants import TIMEOUT_5MIN

Test Structure Conventions:
  - File naming: test_<feature_name>.py
  - Test naming: test_<action>_<expected_outcome>
  - Class grouping: class Test<FeatureName>
  - Shared preconditions in class docstrings

GRAVEYARD Lessons (Mistakes to Avoid):
  - AVOID: Using `vm.restart()` directly - use `vm.clean_up()` then recreate
  - AVOID: Hardcoding namespace names - always use the namespace fixture
  - AVOID: Missing `yield` in fixtures that need cleanup
  - AVOID: Using `time.sleep()` instead of proper wait utilities
  - PATTERN: Always check resource exists before operating on it
```

## Key Principles

1. **Thorough but Focused**: Explore enough to understand patterns, don't read everything
2. **No Hallucination**: Only report what actually exists in the repository
3. **Pattern Recognition**: Look for common patterns across multiple files
4. **Actionable Findings**: Report findings that will be useful for code generation
5. **Learn from Mistakes**: GRAVEYARD.md findings are HIGH PRIORITY - they represent real failures

## Constraints

- **Must not hallucinate**: Only report actual findings from the repository
- **Must verify**: Check that classes/methods actually exist
- **No file output**: This skill does NOT create context.json or any other file
- **Conversational output**: Findings are reported in the conversation for immediate use
- **GRAVEYARD is mandatory**: Always check for and read GRAVEYARD.md - skip only if it doesn't exist

## Usage

```bash
# Explore repository context
/v3-explore-test-context

# The findings will be used by subsequent /v3-generate-pytest or /v3-generate-std calls
```

## Difference from V2.2

In v3, this skill:
- ✅ Reads GRAVEYARD.md to learn from past mistakes (NEW)
- ✅ Reports GRAVEYARD lessons as "DO NOT" rules for code generation (NEW)
- ✅ Reports findings in conversation
- ❌ Does NOT create context.json

## Reusability
**Medium** - Useful before generating tests in this repository, but findings are not persisted for reuse

## Dependencies
- Glob tool (file discovery)
- Grep tool (pattern searching)
- Read tool (file content)
