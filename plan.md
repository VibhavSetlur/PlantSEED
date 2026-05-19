# Plan: Generalize Target Filename in Curation Issue Form & Workflow

## Overview

The current GitHub issue form + workflow pair appends curation data to a **hardcoded** TSV file (`Curation/seaver/test.tsv`). We need to generalize this so the user can specify **any** target file — either an **existing** file (to append new rows) or a **new** path (to create a new file).

---

## Current State

### File: `.github/ISSUE_TEMPLATE/test_workflow.yml`

Issue form with 4 fields:
| Field ID      | Type     | Label                  |
|---------------|----------|------------------------|
| `entity`      | input    | Entity Name / Pathway  |
| `action`      | dropdown | Action (ADD / UPDATE)  |
| `data_type`   | dropdown | Data Type (features / publications) |
| `value`       | input    | Value                  |

All rows are appended to the **hardcoded path** `Curation/seaver/test.tsv`.

### File: `.github/workflows/test_process_form.yaml`

Workflow triggered on `issues: types: [opened]` with label `curation`:

1. **Parse** issue body using regex on markdown headers -> extract `target_file`, `entity`, `action`, `data_type`, `value`
2. **Append** parsed values as a tab-separated row to the user-specified path
3. **Create a new branch** (`curation/issue-<N>`) and push the change
4. **Create a pull request** from that branch against the default branch
5. **Comment & close** the issue

### Existing TSV files in the repo

Curation TSVs follow the pattern:
```
Scripts/PlantSEED_v3/Curation/<username>/<pathway>/<Updates>.tsv
```

Examples: `pelle283/Glucosinolates/`, `ricon001/Homomethionine/`, `samseaver/Photosynthesis/`, etc.

---

## Changes Required

### 1. Modify Issue Template — Add "Target File" field

Add a **new field** to the form that lets the user specify where data should be appended.

**Approach: Single text `input` field**

GitHub issue forms do **not** support dynamic dropdowns (cannot list repo files at runtime). A text input is the only portable mechanism that supports both cases:

- **Existing file**: user types an existing path → row is appended
- **New file**: user types a new path → file + parent directories are created

**Field spec:**

```yaml
  - type: input
    id: target_file
    attributes:
      label: Target Curation File
      description: |
        Relative path to the target `.tsv` file in the repository.
        Use an **existing** file path to append new data, or a **new** path to create a new file.
        Examples:
          - `Scripts/PlantSEED_v3/Curation/seaver/test.tsv`
          - `Scripts/PlantSEED_v3/Curation/pelle283/Glucosinolates/NewStep.tsv`
    validations:
      required: true
```

**Placement**: Insert it **first**, before the existing `entity` field, so the user chooses their target before filling in data.

**Consideration — data_type field**: Keep it. Even though the path implies the curation context, the `data_type` column (`features` vs `publications`) is a distinct semantic attribute of the row being added.

### 2. Modify Workflow — Use Dynamic Target Path + Branch-per-Issue

The workflow needs to:

1. **Checkout** the repository's default branch (e.g., `main`)
2. **Parse** the new `target_file` field from the issue body
3. **Use** the parsed path instead of the hardcoded `Curation/seaver/test.tsv`
4. **Create a new branch** (`curation/issue-<N>`), commit the change, and push
5. **Open a pull request** against the default branch for review
6. **Comment & close** the issue

#### Parsing

The existing `get_field()` regex function matches markdown headers. The new field header will be `"Target Curation File"`, so:

```python
target_file = get_field("Target Curation File")
```

No changes to the regex logic needed — it generalizes to any header.

#### Path handling

Replace:

```python
target_dir = "Curation/seaver/"
target_file = f"{target_dir}/test.tsv"
```

With:

```python
target_file = get_field("Target Curation File")
target_dir = os.path.dirname(target_file)
```

`os.makedirs(target_dir, exist_ok=True)` already handles creating parent directories for new paths — no change needed.

#### Git add

Replace:

```bash
git add Curation/seaver/test.tsv
```

With a dynamic value. Options:

**Option A: Pass via env var from Python step**

In the Python step, write the path to a file (`target_path.txt`) that a subsequent step reads.

```yaml
      - name: Parse Issue and Append to TSV
        env:
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: |
          python3 -c '
          import os, re, sys
          ...
          # Write target file path for subsequent steps
          with open(os.environ["GITHUB_ENV"], "a") as env_file:
              env_file.write(f"TARGET_FILE={target_file}\n")
          '
      - name: Commit and Push Changes
        run: |
          ...
          git add "$TARGET_FILE"
```

**Option B: Use `$GITHUB_ENV` directly**

Same as above but cleaner: use GitHub Actions native env file mechanism.

**Recommendation**: Option A using `GITHUB_ENV` is the standard GitHub Actions pattern.

### 3. Validation

Add these checks in the Python parsing step:

| Check | Action |
|-------|--------|
| `target_file` is empty | `sys.exit(1)` with error message |
| `target_file` doesn't end in `.tsv` | `sys.exit(1)` — enforce convention |
| `target_file` contains path traversal (`..`) | `sys.exit(1)` — security |
| File exists & `action == "ADD"` | Warn but proceed (append mode) |
| File doesn't exist & `action == "UPDATE"` | Warn but proceed (will create) |

### 4. Rename Files (Optional)

The current filenames (`test_workflow.yml`, `test_process_form.yaml`) suggest they're experimental. Consider renaming:

| Current | Proposed |
|---------|----------|
| `.github/ISSUE_TEMPLATE/test_workflow.yml` | `.github/ISSUE_TEMPLATE/curation_form.yml` |
| `.github/workflows/test_process_form.yaml` | `.github/workflows/process_curation_form.yaml` |

If renamed, update the `name` field in both files and the workflow `on:` trigger (none — no cross-reference between issue templates and workflows; GitHub auto-discovers both by location).

---

## Full Updated File Previews

### `.github/ISSUE_TEMPLATE/test_workflow.yml` (updated)

```yaml
name: Curation Update
description: Add or update features and publications
title: "[Curation] "
labels: ["curation"]
body:
  - type: markdown
    attributes:
      value: |
        Use this form to automatically append a new row to a `.tsv` file.
  - type: input
    id: target_file
    attributes:
      label: Target Curation File
      description: |
        Relative path to the target `.tsv` file in the repository.
        Use an **existing** path to append data, or a **new** path to create a new file.
        Examples:
          - `Scripts/PlantSEED_v3/Curation/seaver/test.tsv`
          - `Scripts/PlantSEED_v3/Curation/pelle283/Glucosinolates/NewStep.tsv`
    validations:
      required: true
  - type: input
    id: entity
    attributes:
      label: Entity Name / Pathway
      description: e.g., Glucosinolate gamma-glutamyl hydrolase (EC 3.4.19.16)
    validations:
      required: true
  - type: dropdown
    id: action
    attributes:
      label: Action
      options:
        - ADD
        - UPDATE
    validations:
      required: true
  - type: dropdown
    id: data_type
    attributes:
      label: Data Type
      options:
        - features
        - publications
    validations:
      required: true
  - type: input
    id: value
    attributes:
      label: Value
      description: e.g., Athaliana_TAIR10||AT4G30530 or 10.1038/nchembio.185
    validations:
      required: true
```

### `.github/workflows/test_process_form.yaml` (updated)

```yaml
name: Process Curation Form

on:
  issues:
    types: [opened]

jobs:
  append-to-tsv:
    if: contains(github.event.issue.labels.*.name, 'curation')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write

    steps:
      - name: Checkout Repository (default branch)
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.repository.default_branch }}

      - name: Parse Issue and Append to TSV
        env:
          ISSUE_BODY: ${{ github.event.issue.body }}
        run: |
          python3 -c '
          import os, re, sys

          body = os.environ["ISSUE_BODY"]

          def get_field(header):
              match = re.search(r"### " + header + r"\n+(.*?)(?:\n+(?:###|$)|\n*$)", body, re.DOTALL)
              return match.group(1).strip() if match else None

          # Parse all fields
          target_file = get_field("Target Curation File")
          entity      = get_field("Entity Name / Pathway")
          action      = get_field("Action")
          data_type   = get_field("Data Type")
          value       = get_field("Value")

          # --- Validation ---
          errors = []
          if not target_file:
              errors.append("Target Curation File is required.")
          if not target_file.endswith(".tsv"):
              errors.append("Target Curation File must end in .tsv")
          if ".." in target_file.split("/"):
              errors.append("Path traversal (..) is not allowed.")
          if not all([entity, action, data_type, value]):
              errors.append("All fields (Entity, Action, Data Type, Value) are required.")

          if errors:
              for e in errors:
                  print(f"Error: {e}")
              sys.exit(1)

          # --- Append to target file ---
          target_dir = os.path.dirname(target_file)
          os.makedirs(target_dir, exist_ok=True)

          with open(target_file, "a") as f:
              f.write(f"{entity}\t{action}\t{data_type}\t{value}\n")

          # --- Export target file path for next steps ---
          with open(os.environ["GITHUB_ENV"], "a") as env_file:
              env_file.write(f"TARGET_FILE={target_file}\n")
          '

      - name: Commit and Push to New Branch
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "41898282+github-actions[bot]@users.noreply.github.com"

          git checkout -b "$BRANCH_NAME"
          git add "$TARGET_FILE"
          git commit -m "curation: add data from issue #${{ github.event.issue.number }}"
          git push origin "$BRANCH_NAME"

      - name: Create Pull Request
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr create \
            --base "${{ github.event.repository.default_branch }}" \
            --head "$BRANCH_NAME" \
            --title "Curation: ${{ github.event.issue.title }}" \
            --body "Automated curation update from issue #${{ github.event.issue.number }}." \
            --label "curation"

      - name: Comment and Close Issue
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          BRANCH_NAME: curation/issue-${{ github.event.issue.number }}
        run: |
          gh issue comment ${{ github.event.issue.number }} --body "Curation data added on branch \`$BRANCH_NAME\`. Pull request created for review. Closing issue."
          gh issue close ${{ github.event.issue.number }} --reason "completed"
```

---

## Testing / Verification

Since this is a GitHub Actions workflow, testing requires either:

1. **Dry-run on the fork**: Push the branch and open a test issue on `VibhavSetlur/PlantSEED` (the fork). Verify the issue is parsed, the correct file is created/updated, and the commit appears.
2. **Local simulation**: Run the Python parsing logic locally with a sample issue body to verify regex extraction works for the new field.

Test cases to try:

| Test | Input | Expected |
|------|-------|----------|
| Existing file path | `Scripts/PlantSEED_v3/Curation/seaver/test.tsv` | Row appended to existing file |
| New file path | `Scripts/PlantSEED_v3/Curation/seaver/new_data.tsv` | New file created, directories created |
| Missing `.tsv` extension | `Curation/seaver/data` | Workflow fails with clear error |
| Path with `..` | `Curation/../../etc/passwd` | Workflow fails — security guard |

---

## Future Considerations (Beyond This Plan)

1. **Dropdown of existing files**: If GitHub adds support for dynamic form fields (e.g., via API-backed selectors), replace the text input with a dropdown of all existing curration TSVs.
2. **Per-user directories**: Could auto-detect the issue author's GitHub username and suggest/prefill paths under `Scripts/PlantSEED_v3/Curation/<username>/`.
3. **Multiple row support**: Allow the form to accept multiple values in a single submission (e.g., textarea with one value per line).
4. **Slack integration**: If the user who opens the issue comes from Slack, post a confirmation to the Slack thread.
5. **Auto-merge PR**: Optionally add auto-merge for curation PRs to skip manual review for low-risk updates.

---

## Implementation Order

1. Edit `.github/ISSUE_TEMPLATE/test_workflow.yml` — add the `target_file` field
2. Edit `.github/workflows/test_process_form.yaml` — parse `target_file`, make path dynamic, add validation, use `GITHUB_ENV` for handoff, switch to new-branch + PR flow
3. Push to the `github_flow_test_260518` branch on the fork (or merge to default branch)
4. Test by opening a test issue on the fork
5. Rename YAML files (optional — ask stakeholders)
