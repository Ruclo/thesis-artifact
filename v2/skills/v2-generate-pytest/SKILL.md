# Skill: /v2-generate-pytest

**Purpose**: Generate executable pytest code from Software Test Plan (STP) using repository context

## Input
- **Required**: Path to STP markdown file
- **Prerequisite**: Repository context should be explored using `/v2-explore-test-context` first

## Output
- **File**: `test_<feature_name>.py`
- **Location**: Appropriate directory based on repository conventions

## Role
Act as a Senior SDET. You will translate a Software Test Plan (STP) into an executable Python test file using `pytest`.

You are currently in the root of the openshift-virtualization-tests repository.

## Workflow

You must follow this strict two-phase workflow:

### Phase 1: Parse STP and Design Test Structure

**Note:** This assumes repository context has been explored via `/v2-explore-test-context`

1. **Read STP file:**
   - Extract test scenarios and requirements
   - Identify test objectives
   - Parse acceptance criteria

2. **Map scenarios to test functions:**
   - Design test function names
   - Identify required fixtures (from exploration)
   - Plan test structure

3. **Plan imports:**
   - Use utilities discovered in exploration
   - Follow import patterns from exploration
   - Include necessary fixtures

### Phase 2: Code Generation

- Generate a single `.py` file implementing the scenarios from the STP.
- **Strict Constraint:** Do not hallucinate utilities. You MUST use the exact class names, method signatures, and constants discovered during repository exploration.
- If a helper is missing, implement it locally using the base Kubernetes client.
- Follow repository conventions discovered during exploration.

## Key Principles

1. **No Hallucination**: Only use utilities/classes/methods discovered in exploration
2. **Use Exploration Findings**: Leverage the patterns, utilities, and conventions from `/v2-explore-test-context`
3. **Follow Conventions**: Match existing test patterns and structure from exploration
4. **Verify Imports**: Ensure all imports match the patterns discovered

## Usage

```bash
# Generate pytest from STP
/v2-generate-pytest stps/3.md
```

## Dependencies
- Read tool (explore repository)
- Glob tool (find files)
- Grep tool (search patterns)
- Write tool (create test file)
- Bash tool (run pyright)
- Edit tool (fix errors)

## Reusability
**Medium** - Specific to pytest generation but can be used for any STP in this repository
