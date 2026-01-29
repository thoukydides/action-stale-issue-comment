# `action-stale-issue-summary`

This action uses Google Gemini to turn a <kbd>stale</kbd> label into a structured triage ping that tells the reporter exactly what is blocking an issue:
- **Fetch Issue and Comments**: Retrieves the issue body and all comments, excluding bot comments.
- **Truncate Content**: Intelligently truncates logs, code blocks, and long text to fit the AI model's context limit.
- **Generate Summary**: Uses Google AI Studio to generate a structured summary with next steps.
- **Post and Minimise**: Posts the summary as a comment and minimises any previous summaries to reduce noise.

> [!CAUTION]
> This action is provided for my own use and published in case it is useful to others. If you rely on it, fork and maintain your own copy. No support or stability guarantees are offered.

## Prerequisites

Before using this workflow, ensure:
- The workflow has `issues: write` and `contents: read` permissions (either via the default `GITHUB_TOKEN` or a fine-grained token).
- You have created a [Gemini API key](https://ai.google.dev/gemini-api/docs/api-key) and placed it in a repository secret (e.g. `GEMINI_API_KEY`).
- You understand the [rate limits](https://ai.google.dev/gemini-api/docs/rate-limits) for your chosen model and usage tier.

> [!TIP]
> Google AI Studio Gemini rate limits are per-project. Create multiple projects, each with its own API key, to increase quotas.

## Inputs

Various inputs are defined in the action to configure its operation:

| Name | Description | Default
| --- | --- | ---
| `gemini_api_key`: The Google AI Studio Gemini API key | *required*
| `issue_number` | The GitHub issue to summarise | *required*
| `dry_run` | Disables actions that modify the issue (adding the comment and minimising previous comments) for testing | `false`

## Usage

Create a workflow triggered when the <kbd>stale</kbd> label is applied to an issue.

```yaml
name: AI Stale Issue Summary
permissions:
  issues: write
  contents: read

on:
  issues:
    types: [labeled]
  workflow_dispatch:
    inputs:
      issue_number:
        description: 'Issue number'
        required: true
        type: number
      dry_run:
        description: 'Dry run (do not modify issue)'
        type: boolean
        default: true

jobs:
  stale-summary:
    runs-on: ubuntu-latest

    steps:
    - name: AI stale issue summary
      if: github.event_name == 'workflow_dispatch' || github.event.label.name == 'stale'
      uses: thoukydides/action-stale-issue-comment@v1
      with:
        gemini_api_key: ${{ secrets.GEMINI_API_KEY }}
        # Use the event issue number for label triggers, or the manual input for workflow_dispatch
        issue_number: ${{ github.event.issue.number || fromJson(inputs.issue_number) }}
        dry_run: ${{ inputs.dry_run }}
```

> [!TIP]
> Use `workflow_dispatch` to manually trigger the workflow for specific issues (e.g. for testing or one-off summaries).

### `actions/stale` companion workflow

This workflow is responsible for adding the <kbd>stale</kbd> label; the above workflow reacts to that label by generating and posting the AI summary.

```yaml
name: Close Stale Issues
permissions:
  issues: write
  pull-requests: write

on:
  schedule:
  # Runs at 01:30 UTC daily
  - cron: '30 1 * * *'

jobs:
  stale:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/stale@v10
      with:
        repo-token: ${{ secrets.GITHUB_TOKEN }}
        stale-issue-message: ''
        close-issue-message: '💤 This issue was closed because it has been stalled for 7 days with no activity.'
        stale-issue-label: 'stale'
        days-before-issue-stale: 14
        days-before-issue-close: 7
        remove-issue-stale-when-updated: false
```

> [!IMPORTANT]
> The `remove-issue-stale-when-updated: false` option must be set to prevent the summary comment itself from being treated as activity and immediately clearing the <kbd>stale</kbd> label. However, this also means that **any** comment (including from maintainers) will not remove the label either. Users must manually remove the label to indicate that they are re-engaging; otherwise the issue will still be auto-closed after the grace period.

## ISC License (ISC)

<details>
<summary>Copyright © 2026 Alexander Thoukydides</summary>

> Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted, provided that the above copyright notice and this permission notice appear in all copies.
>
> THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
</details>