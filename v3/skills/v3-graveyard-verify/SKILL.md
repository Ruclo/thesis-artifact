# Skill: /v3-graveyard-verify

**Purpose**: Post-generation verification that cross-references the generated test code against GRAVEYARD.md to catch known mistakes before runtime

## Input
- **Required**: Path to the generated test file

## Output
- **Modified File**: Test file edited in-place with fixes for any GRAVEYARD violations found
- **Report**: Summary of violations found and fixes applied (conversational output)

## Role
You are a **code reviewer specializing in catching known mistakes**. You have access to GRAVEYARD.md, which documents every mistake made during previous test generation runs. Your job is to read the generated code and flag any pattern that matches a documented mistake, then fix it before the code ever reaches pyright or runtime execution.

## Why This Phase Exists

The code generation phase (Phase 3) is instructed to apply GRAVEYARD lessons, but it operates from memory of the exploration phase and may miss specific patterns. This dedicated verification phase provides a **second pass** with direct, focused comparison between the generated code and every GRAVEYARD entry. It catches mistakes that slip through generation, reducing the number of pyright and runtime iterations needed.

## Workflow

### Step 1: Load GRAVEYARD.md

Read `GRAVEYARD.md` from the repository root. For each entry, extract:
- **Category**
- **Wrong Code** pattern
- **Correct Code** replacement
- **How to Avoid** rule

If GRAVEYARD.md does not exist, report that there are no past mistakes to verify against and exit successfully.

### Step 2: Read Generated Test File

Read the entire generated test file.

### Step 3: Cross-Reference Against GRAVEYARD Entries

For **each** GRAVEYARD entry, check whether the generated code contains the "Wrong Code" pattern or violates the "How to Avoid" rule:

```
FOR each graveyard_entry in GRAVEYARD.md:
    # Check for exact pattern match
    IF generated_code contains graveyard_entry.wrong_code:
        FLAG as violation

    # Check for semantic rule violation
    IF generated_code violates graveyard_entry.how_to_avoid:
        FLAG as violation
```

### Step 4: Apply Fixes

For each violation found:
1. Replace the wrong pattern with the correct pattern from GRAVEYARD
2. Verify the fix doesn't break surrounding code
3. Log the fix applied

### Step 5: Report Results

Report how many entries were checked, how many violations were found, and what fixes were applied.

## Constraints

### 1. Only Fix Known Patterns
- Only flag and fix patterns that are explicitly documented in GRAVEYARD.md
- Do NOT invent new rules or apply subjective code review
- This phase is about **known mistakes**, not general code quality

### 2. Preserve Test Intent
- Fixes must not change what the test is testing
- Only change HOW something is done, not WHAT is being tested

### 3. Minimal Changes
- Apply the smallest possible fix for each violation
- Don't refactor surrounding code

## Usage

```bash
/v3-graveyard-verify <test_file_path>
```

## Dependencies
- Read tool (read GRAVEYARD.md and test file)
- Edit tool (apply fixes)
- Grep tool (search for patterns in generated code)

## Reusability
**High** - Can verify any generated test file against GRAVEYARD.md
