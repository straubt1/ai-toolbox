---
name: capture-output
description: >
  Use when the user asks to "capture output", "update expected output", "run usage commands",
  "capture demo output", or wants to run the terraform commands from an example's ## Usage
  section and write the real terminal output into the ## Expected Output section of the README.
  Make sure to use this skill whenever the user mentions updating expected output, capturing
  command output, or running usage examples for any directory.
---

# Capture Output Skill

Run the terraform commands from an example's `## Usage` section, capture real terminal output,
and inject it into the `## Expected Output` section of the README.

The artifact pattern mirrors `record-demo`: generate an editable bash script, run it to produce
a raw output file, then read and format that output into the README. This lets the user tweak
the script and re-run cheaply without reinvoking the full skill.

## Input

The user provides a path to a directory (e.g., `path/to/my-example`).
The directory contains a `main.tf` and a `README.md` with a `## Usage` section containing
one or more shell code blocks.

---

## Process

### Step 1: Analyze the README

Read `<example>/README.md`. Locate the `## Usage` section.
Identify each shell code block in order — each becomes one section in the capture script.
Note any comment lines inside code blocks (e.g., `# Invoke standalone`) — these become section labels.

### Step 2: Generate the capture script

Write `assets/<example-name>-capture.sh` at the repo root.

**Script structure:**

```bash
#!/bin/bash
set -e
cd <path-to-directory>

# Silent setup: init and clean any existing state
terraform init -input=false > /dev/null 2>&1
terraform destroy -auto-approve > /dev/null 2>&1

echo "=== SECTION 1: Default apply ==="
terraform apply -no-color -auto-approve 2>&1
echo ""

echo "=== SECTION 2: Invoke standalone ==="
terraform apply -invoke=action.local_command.foo -no-color -auto-approve 2>&1
echo ""
```

**Script generation rules:**

- `terraform init` → always silent (`> /dev/null 2>&1`), in the setup block at the top
- `terraform destroy -auto-approve` → always silent (`> /dev/null 2>&1`), in the setup block
- `cd <subdir>` → emit as-is (for examples with subdirectories like `02-old-way-vs-new-way`)
- `terraform apply` (and `-replace`, `-var` variants) → append `-no-color -auto-approve` before running, capture output
- `terraform apply -invoke=...` → append `-no-color -auto-approve`, capture output
- `terraform plan -invoke=...` → append `-no-color`, capture output
- Each code block from the README becomes one `=== SECTION N: <label> ===` block
- Use the code block's leading comment as the label; fall back to `Section N` if none present
- State persists across sections (section 2 can reference resources created in section 1)

**For examples with subdirectories** (e.g., `02-old-way-vs-new-way`):
emit separate `cd old-way` / `cd ../new-way` navigation with their own init+destroy blocks.

Make the script executable: `chmod +x assets/<example-name>-capture.sh`

### Step 3: Run the script

```bash
bash assets/<example-name>-capture.sh > assets/<example-name>-output.txt 2>&1
```

Check for errors. If it fails, diagnose the problem (wrong directory, state conflict, missing provider),
fix `assets/<example-name>-capture.sh`, and re-run. Don't retry blindly — read the output file first.

### Step 4: Read and filter the output

Read `assets/<example-name>-output.txt`. Split on `=== SECTION N: <label> ===` markers.

For each section, **keep** these line patterns:

- `Plan: X to add, Y to change, Z to destroy. Actions: N to invoke.`
- `Action started: ...`
- `Action action.* (...):` — the action's log-prefix header line
- All lines between `Action started:` and `Action complete:` (the action's own stdout)
- `Action complete: ...`
- `<resource>.this: Creating...`
- `<resource>.this: Creation complete after Xs [id=...]`

**Discard:**

- `Terraform used the selected providers to generate the following execution plan...`
- `Terraform will perform the following actions:` and the diff block (`+`, `-`, `~` lines)
- `Apply complete! Resources: N added...`
- Provider lock / plugin init messages
- Blank lines at the very start or end of a section's output

### Step 5: Apply placeholder substitutions

Replace dynamic values so the Expected Output stays stable across runs:

| Dynamic value | Example | Replacement |
|---|---|---|
| random_pet name inside `[id=X]` | `[id=probable-foal]` | `[id=<pet-name>]` |
| Timestamps / full dates | `Thu Mar 13 12:00:00 CDT 2025` | `<timestamp>` |
| UUIDs | `550e8400-e29b-41d4-a716-446655440000` | `<id>` |
| Elapsed seconds | `0s`, `1s` | keep as-is |

### Step 6: Format and inject into README

**Single section** (one code block in ## Usage):

```markdown
## Expected Output

On `terraform apply`, you should see:

```
<filtered output>
```
```

**Multiple sections** (multiple code blocks — use the section label as sub-heading):

```markdown
## Expected Output

**Default apply:**

```
<filtered output from section 1>
```

**Invoke standalone:**

```
<filtered output from section 2>
```
```

Replace the entire `## Expected Output` section in the README
(from the `## Expected Output` heading through to the next `##` heading, or EOF).
If no `## Expected Output` section exists, insert it immediately after `## Usage`.

### Step 7: Update Taskfile.yml

Add a `capture:<example-name>` task to `Taskfile.yml` at the repo root (if not already present):

```yaml
  capture:03-before-after:
    desc: Run capture script and update Expected Output for 03-before-after
    cmds:
      - bash assets/03-before-after-capture.sh > assets/03-before-after-output.txt 2>&1
```

The user can tweak the script and re-run via `task capture:<example-name>`, then ask Claude to
re-read `assets/<example-name>-output.txt` and update the README — no full skill reinvocation needed.

### Step 8: Verify

- Confirm `assets/<example-name>-capture.sh` exists and is readable
- Confirm `assets/<example-name>-output.txt` exists and is non-empty
- Confirm `## Expected Output` in the README contains real terraform output
- Report the output file size
