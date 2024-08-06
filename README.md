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
name: Dynamic Fork Maintenance Schedule Workflow

on:
  schedule:
    - cron: '0 * * * *' # Runs every hour

jobs:
  schedule_job:
    runs-on: ubuntu-latest
    env:
      SCHEDULE_CONFIG: ${{ secrets.SCHEDULE_CONFIG }} # Secret storing the schedule JSON

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Read schedule configuration
        id: read_config
        run: |
          echo "${{ env.SCHEDULE_CONFIG }}" > schedule.json

      - name: Parse and use schedule data
        id: parse_schedule
        run: |
          SCHEDULE_FILE=schedule.json
          SCHEDULE=$(cat $SCHEDULE_FILE)
          echo "Schedule: $SCHEDULE"

          # Loop through each event in the schedule
          for row in $(jq -c '.events[]' $SCHEDULE_FILE); do
            NAME=$(echo $row | jq -r '.name')
            LAST_RUN=$(echo $row | jq -r '.last_run')
            BRANCHING_DATE=$(echo $row | jq -r '.branching_date')

            # Get the current date
            CURRENT_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

            if [[ "$CURRENT_DATE" > "$BRANCHING_DATE" && "$LAST_RUN" < "$BRANCHING_DATE" ]]; then
              echo "Event $NAME needs to run. Last run: $LAST_RUN, Branching date: $BRANCHING_DATE, Current date: $CURRENT_DATE"
              echo "::set-output name=event::$NAME"
              SCHEDULE_JSON=$(echo $row | jq -c '{upstream_main_branch, upstream_release_branch, downstream_testing_branch, downstream_develop_branch, commits}')
              echo "::set-output name=schedule_json::$SCHEDULE_JSON"
              break
            fi
          done

      - name: Fork Maintenance System
        if: steps.parse_schedule.outputs.event != ''
        uses: Cemberk/Fork-Maintenance-System@main 
        with:
          github_token: ${{ secrets.CRED_TOKEN }}
          upstream_repo: "https://github.com/huggingface/transformers"
          schedule_json: ${{ steps.parse_schedule.outputs.schedule_json }}
          pr_branch_prefix: "scheduled-merge"
          unit_test_command: "your-unit-test-command"
          performance_test_command: "your-performance-test-command"

      - name: Remove completed event from schedule
        if: success() && steps.parse_schedule.outputs.event != ''
        run: |
          SCHEDULE_FILE=schedule.json
          EVENT_NAME=${{ steps.parse_schedule.outputs.event }}
          jq 'del(.events[] | select(.name == "'$EVENT_NAME'"))' $SCHEDULE_FILE > tmp.$$.json && mv tmp.$$.json $SCHEDULE_FILE

      - name: Update secret with new schedule configuration
        if: success() && steps.parse_schedule.outputs.event != ''
        run: |
          SCHEDULE_JSON=$(cat schedule.json)
          gh secret set SCHEDULE_CONFIG --body "$SCHEDULE_JSON"
        env:
          GITHUB_TOKEN: ${{ secrets.CRED_TOKEN }}
```


### Dynamic Schedule Input

```json
{
  "events": [
    {
      "name": "weekly_backup",
      "upstream_main_branch": "main",
      "upstream_release_branch": "release-v1.0",
      "downstream_testing_branch": "v6.3testing",
      "downstream_develop_branch": "develop",
      "commits": [
        "abcd1234",
        "efgh5678"
      ],
      "branching_date": "2024-08-10T00:00:00Z",
      "last_run": "2024-08-03T00:00:00Z"
    },
    {
      "name": "monthly_report",
      "upstream_main_branch": "main",
      "upstream_release_branch": "release-v2.0",
      "downstream_testing_branch": "v6.4testing",
      "downstream_develop_branch": "develop",
      "commits": [
        "ijkl9012",
        "mnop3456"
      ],
      "branching_date": "2024-08-15T00:00:00Z",
      "last_run": "2024-07-15T00:00:00Z"
    }
  ]
}
```

