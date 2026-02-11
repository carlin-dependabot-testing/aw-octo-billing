---
name: Security Bundle Issue Creator
description: Find all open Dependabot and code scanning fix PRs and create bundle issues grouped by ecosystem and directory, assigned to Copilot

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

### Step 1: List Open PRs

Call `github___list_pull_requests` with:
- `owner`: "carlin-dependabot-testing"
- `repo`: "aw-octo-billing"
- `state`: "open"
- `per_page`: 100
- `page`: 1

Then call again with `page: 2` to get the next batch.

The response may be too large to return inline. If you receive a `payloadPath` instead, you may use `head`, `grep`, or `python3` to extract PR numbers and titles from that file. This is the ONLY situation where running a command is acceptable.

From the results, identify two types of PRs:

1. **Dependabot PRs**: `user.login` is `"dependabot[bot]"`
2. **Code scanning fix PRs**: Title contains `[code-scanning-fix]`

Ignore all other PRs.

### Step 2: Group PRs by Ecosystem and Directory

**For Dependabot PRs**, extract the ecosystem and directory from the title:
- Pattern: "Bump {package} from {old} to {new} in {directory}"
- If no "in {directory}" is present, the directory is "/"
- The ecosystem is determined by the package manager (npm, pip, docker, github-actions, etc.)

**For code scanning fix PRs**, group them separately:
- Use "code-scanning" as the ecosystem
- Extract the directory from the file path mentioned in the PR body or title
- If the directory is unclear, use "/"

Group all PRs by ecosystem + directory. Keep track of each PR's number and title.

### Step 3: Check for Existing Bundle Issues

Call `github___search_issues` with:
- `q`: "repo:carlin-dependabot-testing/aw-octo-billing label:agentic-campaign is:open"

If a bundle issue already exists that covers a group (match by ecosystem and directory in the title), skip that group.

### Step 4: Create Bundle Issues

For each NEW group (not already covered by an existing issue), call `create_issue` with:

- **title**: `{ecosystem} updates for {directory}` (the "[Security Bundle] " prefix is added automatically)
- **body**: A markdown body listing the PRs in that group

Example for a Dependabot group:

```json
{
  "title": "npm updates for /src",
  "body": "## Security Bundle Issue\n\nThis issue tracks all open security-related PRs for the **npm** ecosystem in the **/src** directory.\n\n### Open PRs\n\n- #12 - Bump express from 4.16.0 to 5.2.1 in /src\n- #15 - Bump lodash from 4.17.4 to 4.17.23 in /src\n\n### Instructions\n\n@copilot Please review these dependency updates and:\n1. Check for any breaking changes\n2. Verify compatibility between updates\n3. Recommend which PRs can be merged together\n4. Highlight any security updates that should be prioritized\n\n*Ecosystem: npm | Directory: /src*"
}
```

Example for a code scanning group:

```json
{
  "title": "code-scanning fixes for /src",
  "body": "## Security Bundle Issue\n\nThis issue tracks all open code scanning fix PRs in the **/src** directory.\n\n### Open PRs\n\n- #20 - [code-scanning-fix] Fix js/xss: Sanitize user input in handler\n- #21 - [code-scanning-fix] Fix js/sql-injection: Use parameterized queries\n\n### Instructions\n\n@copilot Please review these security fixes and:\n1. Verify each fix properly addresses the vulnerability\n2. Check for any regressions or side effects\n3. Recommend merge order based on severity\n4. Confirm no new vulnerabilities are introduced\n\n*Ecosystem: code-scanning | Directory: /src*"
}
```

Create one issue per group. Maximum 20 issues.

### Step 5: Create a Project Status Update (LAST)

After ALL issues are created, call `create_project_status_update` exactly once. This MUST be your final tool call:

```json
{
  "project": "https://github.com/orgs/carlin-dependabot-testing/projects/1",
  "status": "ON_TRACK",
  "body": "## Security Bundle Summary\n\nCreated X bundle issues covering Y PRs across Z directories.\n\nBreakdown:\n- Dependabot PRs: N\n- Code scanning fix PRs: N\n\nAll issues assigned to Copilot for review."
}
```

### Step 6: If No PRs Found

If there are no open Dependabot or code scanning fix PRs, call `noop` with message: "No open Dependabot or code scanning fix PRs found."