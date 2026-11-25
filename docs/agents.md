---
title: AI agents
description: >-
    A NextITS skill for AI coding agents (Claude Code and others), and the documentation in a format that agents can read.
---

# AI agents

If you use an AI agent such as Claude Code to set up your analyses, give it the NextITS skill.
A skill is a short instruction file that the agent reads when a task needs it.
The NextITS skill tells the agent how the pipeline is organised,
which questions to ask you before a run (barcodes, primers, platform),
and the mistakes that most often lead to a failed or wrong run.
For parameters and defaults it points the agent to this documentation, so it stays up to date with the pipeline.

[Download SKILL.md :lucide-download:](https://raw.githubusercontent.com/Next-ITS/nextits.github.io/main/docs/agents/nextits/SKILL.md){ .md-button .md-button--primary }

## Install the skill

=== "Claude Code"

    For all your projects:

    ```bash
    mkdir -p ~/.claude/skills/nextits
    curl -fsSL -o ~/.claude/skills/nextits/SKILL.md \
      https://raw.githubusercontent.com/Next-ITS/nextits.github.io/main/docs/agents/nextits/SKILL.md
    ```

    For one project only, use `.claude/skills/nextits/` in the project folder instead.
    Claude Code picks up the skill in the next session; check with `/skills`.

=== "Other agents"

    The skill follows the [Agent Skills](https://agentskills.io/) format, which many coding agents support.
    Save `SKILL.md` as `nextits/SKILL.md` in the skills folder named in your agent's documentation.

=== "Chat assistants"

    If your assistant does not support skills, attach `SKILL.md` to the conversation
    or paste its text at the start of the chat.

To update the skill, download it again.

## Documentation for agents

The documentation is also published as plain Markdown, which agents read more reliably than web pages:

- [`llms.txt`](https://next-its.github.io/llms.txt): an index of all pages with links to their Markdown versions
- [`llms-full.txt`](https://next-its.github.io/llms-full.txt): all pages in one file
- The **Copy as Markdown** button at the top of each page copies that page for pasting into a chat.

## Example requests

With the skill installed, you can describe the task in your own words, for example:

- *Set up NextITS Step-1 for the three PacBio runs in `Input/`. Barcodes are dual asymmetric.*
- *Write a Slurm script that runs Step-2 on `Step1_Results/` with UNOISE and VSEARCH at 98%.*
- *Step-1 of Run02 failed. Find out why.*
- *Draft the methods section from the NextITS output in `Step2_Results/`.*

!!! warning "Check before you run"
    Read the commands the agent proposes before running them, especially jobs that use a lot of cluster time
    and commands that delete files. Results in `--outdir` are links into the `work/` directory by default,
    so deleting `work/` breaks them (see [Where results are stored](usage.md#where-results-are-stored)).

## The skill

??? abstract "SKILL.md"

    ````markdown
    --8<-- "docs/agents/nextits/SKILL.md"
    ````
