# fork-maintenance-system
action automations to help support fork maintenance

# Roadmap
- [X] merge conflict resolution button function
- [X] cadence workflow variables and execution for create and update
- [ ] triage testing issues with upstream 

# Fork Maintenance GitHub Action

This repository contains a custom GitHub Action for maintaining forked repositories. It includes features to check for misspelled code, resolve merge conflicts from upstream PRs, and automatically create PRs based on a schedule.

## Features

- **Misspell Check:** Automatically checks for misspellings in markdown files and suggests fixes.
- **Merge Conflict Resolution:** Automatically attempts to merge upstream changes and uses reviewdog to suggest fixes for merge conflicts.
- **Scheduled PRs:** Automatically creates PRs based on the cadence of internal and release branching.

## Usage

### Fork Maintenance Workflow

To use this action in your forked repository, create a workflow file (`.github/workflows/use-fork-maintenance.yml`) with the following content:

```yaml
name: Fork Maintenance

on:
  pull_request:
    branches:
      - main

jobs:
  fork-maintenance:
    runs-on: ubuntu-latest
    steps:
      - name: Fork Maintenance System
        uses: Cemberk/Fork-Maintenance-System@main 
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          upstream_repo: "https://github.com/original/repo.git" # Replace with actual upstream repo URL
          upstream_branch: "v4.40-release"
          internal_branch: "main"
          pr_branch_prefix: "scheduled-merge"
```

### Dynamic Schedule Input

```json
{
  "events": [
    {
      "name": "weekly_backup",
      "cron": "0 0 * * 1",
      "last_run": "2023-06-12T00:00:00Z"
    },
    {
      "name": "monthly_report",
      "cron": "0 0 1 * *",
      "last_run": "2023-06-01T00:00:00Z"
    }
  ]
}
```

