# Skill: /v3-generate-pytest

**Purpose**: Generate executable pytest code from Software Test Description (STD) using repository context

## Input
- **Required**: Path to STD markdown file
- **Prerequisite**: Repository context should be explored using `/v3-explore-test-context` first
- **Optional**: `--output-dir` - target directory (default: infer from repository conventions)

## Output
- **File**: `test_<feature_name>.py`
- **Location**: Determined by repository conventions (e.g., `tests/<domain>/<feature>/`)
- **Format**: Executable pytest file with proper structure

## Role
Act as a Senior SDET. You will translate a Software Test Description (STD) into an executable Python test file using `pytest`.

You are currently in the root of the openshift-virtualization-tests repository.

## Workflow

You must follow this strict two-phase workflow:

### Phase 1: Parse STD and Design Test Structure

**Note:** This assumes repository context has been explored via `/v3-explore-test-context`

1. **Read STD file:**
   - Extract test scenarios from the STD
   - Parse preconditions
   - Identify test data requirements
   - Extract acceptance criteria

2. **Map scenarios to test functions:**
   - test_scenario_1()
   - test_scenario_2()
   - ...

3. **Identify required fixtures:**
   - Based on scenario requirements
   - Match from discovered fixtures in exploration phase

4. **Plan imports:**
   - Use discovered common import patterns
   - Add scenario-specific imports
   - Include utilities found in exploration

### Phase 2: Generate Code

Generate a single `.py` file implementing the scenarios from the STD.

- **Strict Constraint:** Do not hallucinate utilities. You MUST use the exact class names, method signatures, and constants discovered during repository exploration.
- If a helper is missing, implement it locally using the base Kubernetes client.
- Follow repository conventions discovered during exploration
- Use fixtures and patterns discovered in exploration
- **GRAVEYARD Awareness**: Apply all "DO NOT" rules from GRAVEYARD.md findings reported during exploration. Actively avoid patterns that have caused failures in previous runs.

## Code Generation Template

```python
# -*- coding: utf-8 -*-

"""
{feature_name} Tests

{brief_description}

Based on: {std_file}
"""

import logging

import pytest

# Imports from repository exploration
{discovered_imports}

LOGGER = logging.getLogger(__name__)


{fixture_definitions_if_needed}


@pytest.mark.polarion("{polarion_id}")
@pytest.mark.{tier}
def test_{scenario_name}({fixtures}):
    """
    {scenario_description}

    Steps:
    {test_steps}

    Expected:
    {expected_outcome}
    """
    LOGGER.info(f"Starting test: {scenario_name}")

    # Preconditions
    {precondition_code}

    # Test execution
    {test_code}

    # Validation
    {assertion_code}

    # Cleanup (if needed)
    {cleanup_code}
```

## Strict Constraints

### 1. No Hallucination
- **MUST** only use utilities/classes/methods that exist in the repository
- If a helper is missing → implement locally using base Kubernetes client
- Never assume a method exists without verification

### 2. Repository Fidelity
- **MUST** follow discovered naming conventions
- **MUST** use discovered fixtures
- **MUST** match discovered import patterns
- **MUST** place file according to repository structure

### 3. STD Completeness
- **MUST** implement ALL scenarios from STD
- **MUST** cover ALL acceptance criteria
- **MUST** include ALL preconditions from STD
- **MUST** add test data from STD requirements

### 4. GRAVEYARD Compliance
- **MUST** check all patterns against GRAVEYARD lessons from exploration
- **MUST NOT** repeat any mistake documented in GRAVEYARD.md
- If a GRAVEYARD entry conflicts with an STD pattern, follow the GRAVEYARD lesson

## Key Principles

1. **No Hallucination**: Only use utilities/classes/methods discovered in exploration
2. **Use Exploration Findings**: Leverage the patterns, utilities, and conventions from `/v3-explore-test-context`
3. **Follow Conventions**: Match existing test patterns and structure from exploration
4. **STD Fidelity**: Implement exactly what's in the STD
5. **Verify Imports**: Ensure all imports match the patterns discovered
6. **Avoid Past Mistakes**: Apply GRAVEYARD lessons proactively

## Usage

```bash
# Standard usage
/v3-generate-pytest std_vm_reset.md

# Custom output directory
/v3-generate-pytest std_feature.md --output-dir tests/custom/

# Output: tests/virt/lifecycle/test_vm_reset.py
```

## Error Handling

If STD is malformed:
```
ERROR: Could not parse STD file
HINT: Ensure STD follows expected structure
```

If utility is missing from repository:
```
WARNING: Utility 'FooHelper' not found in repository
ACTION: Implementing locally using base Kubernetes client
```

## Dependencies
- Read tool (explore repository and read STD)
- Glob tool (find files)
- Grep tool (search patterns)
- Write tool (create test file)

## Reusability
**Medium** - Can be used for generating pytest tests from any STD in this repository
