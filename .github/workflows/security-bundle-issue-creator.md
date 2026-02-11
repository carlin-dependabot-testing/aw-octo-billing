---
name: Security Bundle Issue Creator
description: Find all open Dependabot and Code Scanning PRs and create bundle issues for each top level directory combination, assign the Issues to to Copilot

on:
  schedule: weekly on monday
  workflow_dispatch:

timeout-minutes: 30

permissions:
  contents: read
  issues: read
  pull-requests: read
  security-events: read

network:
  allowed:
    - defaults
    - github

tools:
  github:
    toolsets:
      - default
      - dependabot
      - code_security
      - code_scanning
      - projects

safe-outputs:
  create-issue:
    expires: 7d
    title-prefix: "[Security Bundle] "
    labels: [agentic-campaign]
    assignees: [copilot]
    max: 20
    group: false
  create-project-status-update:
    project: "https://github.com/orgs/carlin-dependabot-testing/projects/1"
    max: 1
    github-token: ${{ secrets.GH_AW_PROJECT_GITHUB_TOKEN }}
---

# Security Bundle Issue Creator

## CRITICAL RULES

1. Do NOT write files to the repository. Only call MCP tools.
2. When generating markdown for issue bodies, use standard 3-backtick code fences, NOT 6-backtick fences.
3. `create_project_status_update` MUST be the very last tool call you make.

## Step-by-Step Instructions

Complete ALL steps in order.

### Step 1: List Open Dependabot and Code Scanning PRs

Call `github___list_pull_requests` with:
- `owner`: "carlin-dependabot-testing"
- `repo`: "aw-octo-billing"
- `state`: "open"
- `per_page`: 100
- `page`: 1

Then call again with `page: 2` to get the next batch.

The response may be too large to return inline. If you receive a `payloadPath` instead, you may use `head`, `grep`, or `python3` to extract PR numbers and titles from that file. This is the ONLY situation where running a command is acceptable.

From the results, identify PRs authored by Dependabot by checking for `user.login` = `"dependabot[bot]"` and identify PRs related to code scanning alerts by checking for `labels` that include `"code-scanning-alert"` or PR titles that include `code-scanning-fix`.

### Step 2: Group PRs by Ecosystem and Directory

For each PR, extract the ecosystem and directory from the PR title. Dependdabot titles follow this pattern:
- "Bump {package} from {old} to {new} in {directory}"
- If no "in {directory}" is present, the directory is "/"

For each PR, extract the ecosystem and directory from the PR title. Code Scanning PR titles follow this pattern:
`code-scanning-fix` prefix.

Group the PRs by ecosystem + directory. Keep track of each PR's number and title.

### Step 3: Check for Existing Bundle Issues

Call `github___search_issues` with:
- `q`: "repo:carlin-dependabot-testing/aw-octo-billing label:agentic-campaign is:open"

If a bundle issue already exists for a group, skip that group.

### Step 4: Create Bundle Issues

For each NEW group (not already covered by an existing issue), call `create_issue` with:

- **title**: `{ecosystem} updates for {directory}` (the "[Security Bundle]" prefix is added automatically)
- **body**: A markdown body listing the PRs in that group

Example:

```json
{
  "title": "npm updates for /test-deps/project1",
  "body": "## Security Bundle Issue\n\nThis issue tracks all open Security PRs for the **npm** ecosystem in the **/test-deps/project1** directory.\n\n### Open PRs\n\n- #123 - Bump express from 4.16.0 to 5.2.1 in /test-deps/project1\n- #456 - Bump lodash from 4.17.4 to 4.17.23 in /test-deps/project1\n\n### Instructions\n\n@copilot Please review these dependency updates and:\n1. Check for any breaking changes\n2. Verify compatibility between updates\n3. Recommend which PRs can be merged together\n4. Highlight any security updates that should be prioritized\n\n*Runtime: npm | Manifest: /test-deps/project1*"
}
```

Create one issue per group. Maximum 20 issues.

### Step 5: Create a Project Status Update (LAST)

After ALL issues are created, call `create_project_status_update` exactly once. This MUST be your final tool call:

```json
{
  "project": "https://github.com/orgs/carlin-dependabot-testing/projects/1",
  "status": "ON_TRACK",
  "body": "## Seucrity Bundle Summary\n\nCreated X bundle issues covering Y Security PRs across Z directories.\n\nAll issues added to project board with status 'Review Required'."
}
```

### Step 6: If No PRs Found

If there are no open Security PRs, call `noop` with message: "No open Security PRs found."