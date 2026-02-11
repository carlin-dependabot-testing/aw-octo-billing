---
name: Add Security Bundle Issues to Project
description: Automatically adds newly created agentic-campaign issues to the project board

on:
  issues:
    types: [opened, labeled]

permissions:
  contents: read
  issues: read
  pull-requests: read

tools:
  github:
    toolsets:
      - default
      - projects

safe-outputs:
  update-project:
    project: https://github.com/orgs/carlin-dependabot-testing/projects/1
    max: 1
    github-token: ${{ secrets.GH_AW_PROJECT_GITHUB_TOKEN }}

timeout-minutes: 5

network:
  allowed:
    - defaults
    - github
---

# Add Security Bundle Issue to Project Board

You are an assistant that adds newly created issues to a GitHub Project board.

## Instructions

1. Look at the issue that triggered this workflow. Use the GitHub MCP tools to fetch the issue details for this repository: `carlin-dependabot-testing/aw-octo-billing`.
2. If it has the `agentic-campaign` label, call `update_project` to add it to the project board with Status set to "Review Required".
3. If it does NOT have the `agentic-campaign` label, call `noop` with a message explaining the issue was skipped.

The project URL is: `https://github.com/orgs/carlin-dependabot-testing/projects/1`