# ai-toolbox

A collection of AI-assisted tools, skills, and commands for automating common development workflows. While not exclusive to any single AI tool, the repo currently contains skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Getting Started

1. Clone this repo (or copy the relevant files into your own setup).
2. Reference these skills in your Claude Code configuration

## Claude Code Skills

Skills are defined in `.claude/skills/` and can be invoked via slash commands or natural language.

### record-demo

Generates animated GIF recordings of Terraform example workflows using [VHS](https://github.com/charmbracelet/vhs) (a terminal recording tool).

**What it does:** Reads the `## Usage` section of an example's README, generates a `.tape` file with the corresponding Terraform commands, runs VHS to produce a GIF, and embeds it in the README.

**Prerequisites:**
- [VHS](https://github.com/charmbracelet/vhs) (`brew install vhs`)
- [Terraform](https://developer.hashicorp.com/terraform/install)
- ffmpeg (installed automatically with VHS via Homebrew)
- RobotoMono Nerd Font (`brew install font-roboto-mono-nerd-font`)

**When to use:** You want an animated GIF demo of a Terraform example's commands and output.

**How to invoke:** `/record-demo`, or ask Claude to "record a demo", "create a gif", or "generate a tape file".

---

### capture-output

Runs the Terraform commands from an example's `## Usage` section and writes the real terminal output into the `## Expected Output` section of the README.

**What it does:** Generates an executable bash script from the usage code blocks, runs it, filters and cleans the output (replacing IDs, timestamps, etc. with placeholders), and injects the result back into the README.

**Prerequisites:**
- [Terraform](https://developer.hashicorp.com/terraform/install)

**When to use:** You want to populate or refresh the expected output in an example README so it reflects actual Terraform behavior.

**How to invoke:** `/capture-output`, or ask Claude to "capture output", "update expected output", or "run usage commands".

#### Example

Given a README like this:

~~~markdown
# My Example

Some description of what this example does.

## Usage

```shell
terraform init
terraform apply
```

```shell
terraform apply -replace=random_pet.this
```

## Expected Output
~~~

After running `/capture-output`, the skill populates the `## Expected Output` section with real, filtered output:

~~~markdown
## Expected Output

**Default apply:**

```
Plan: 1 to add, 0 to change, 0 to destroy.
random_pet.this: Creating...
random_pet.this: Creation complete after 0s [id=<pet-name>]
```

**Replace apply:**

```
Plan: 0 to add, 0 to change, 1 to destroy.
random_pet.this: Creating...
random_pet.this: Creation complete after 0s [id=<pet-name>]
```
~~~

Dynamic values like pet names, timestamps, and UUIDs are replaced with stable placeholders (`<pet-name>`, `<timestamp>`, `<id>`).

## Repo Structure

```
.claude/
  skills/
    record-demo/
      SKILL.md        # Claude-facing instructions for the record-demo skill
    capture-output/
      SKILL.md        # Claude-facing instructions for the capture-output skill
      evals/
        evals.json    # Test cases for the capture-output skill
```
