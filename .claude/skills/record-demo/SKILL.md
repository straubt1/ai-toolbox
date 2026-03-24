---
name: record-demo
description: >
  Use when the user asks to "record a demo", "create a gif", "generate a tape file",
  "record-demo", or wants to create a VHS terminal recording for a Terraform example.
  Make sure to use this skill whenever the user mentions VHS, tape files, terminal recordings,
  or demo GIFs for any example in the examples/ directory.
---

# Record Demo Skill

Generate a VHS terminal recording (GIF) for a Terraform example directory.

## Input

The user provides a path to a directory (e.g., `examples/01-hello-action`).
This directory will either contain a `main.tf` file directly, or sub-directories with their own `main.tf` files (e.g., `examples/02-old-way-vs-new-way/old-way/main.tf` and `examples/02-old-way-vs-new-way/new-way/main.tf`).
There will also be a `README.md` file in the example directory that has a "## Usage" section.


## Process

### Step 1: Analyze the Readme

In the example's `README.md`, find the "## Usage" section. This section will contain the Terraform commands that should be run to demonstrate the example, and may also contain additional instructions (e.g., "To force the resource to be recreated, run `terraform apply -replace=random_pet.this`") that modify the default flow.

Within the "## Usage" section, identify the key commands that should be included in the demo.
Each grouping of commands will be encased in a code block, example:

```shell  
terraform init    
terraform apply
```

Each of these code blocks will be a separate section that results in a *.tape file. If there are multiple code blocks, the demo should run through them sequentially.

### Step 2: Generate the `.tape` Files

Write the tape file to `assets/<example-name>-<section-n>.tape` at the repo root. The example name is the directory name (e.g., `01-hello-action-01`).

**Always start with these common settings:**

```tape
Output assets/<example-name>.gif
Require terraform

Set Shell bash
Set Theme "Catppuccin Mocha"
Set FontSize 16
Set FontFamily "Roboto Mono"
Set Width 1200
Set Height 800
Set Padding 20
Set Framerate 30
Set PlaybackSpeed 0.75
Set TypingSpeed 75ms
Set WaitTimeout 30s
```

- `Framerate 30` — captures fewer frames so fast-scrolling output is more readable per frame
- `PlaybackSpeed 0.75` — slows the GIF to 75% speed, giving viewers more time to read long output

**Tape structure guidelines:**

- Navigate to the example directory with `Type "cd examples/<example-name>"` + `Enter`
  - `Hide` this and `clear` after.
- The following commands should be placed inside a `Hide`/`Show` block:
  - `terraform init`
  - `cd`
  - `terraform destroy -auto-approve`
- The last step in any `Hide` block should be to call `clear` to empty the terminal before showing the next command
- After any `terraform` command, use a `Sleep 2s` to allow the command to execute and the output to be visible before the next command is typed
- End with a final `Sleep 3s` for the last frame
- Use `Type` + `Enter` for each command
- **ALWAYS** after any `terraform apply` command (including `-invoke`, `-replace`, or plain `apply`), add `Wait /Enter a value/` then `Type "yes"` then `Enter`. Every `terraform apply` variant prompts for confirmation.

### Step 3: Run VHS

Execute from the repo root each generated tape file with:

```bash
vhs assets/<example-name>-<section-n>.tape
```

Check the output for errors. If VHS fails, diagnose and fix the tape file.

### Step 4: Create or Update Example README

If the example already has a `README.md`, add the GIF embed directly after the code block in the "## Usage" section.

**GIF markdown format:**

```markdown
![demo](../../assets/<example-name>.gif)
```

### Step 4b: Update Taskfile

If not present, create a task in the root `Taskfile.yml` to be able to regenerate the demo with `task record-<example-name>`. The task should run the VHS command to regenerate the GIF.

```yaml
  record:01-hello-action:
    desc: Generate the demo GIF for examples/01-hello-action
    cmds:
      - vhs assets/01-hello-action-01.tape
      - vhs assets/01-hello-action-02.tape
```

### Step 5: Verify

- Confirm `assets/<example-name>.gif` exists
- Confirm `assets/<example-name>.tape` exists
- Confirm the example's `README.md` contains the GIF reference
- Report the GIF file size to the user
