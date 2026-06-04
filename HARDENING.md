# Hardening Report: slackapi--slack-github-action/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **slackapi--slack-github-action/v3.0.3** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes a remotely fetched shell script directly to bash without first saving it to a file for inspection: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. If the remote URL is compromised or subject to a MITM attack, arbitrary code executes on the runner.

Locations:

- `cli/action.yml:57`
- `cli/action.yml:59`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step in cli/action.yml uses PowerShell's `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex` pattern, which downloads and immediately executes a remote PowerShell script via Invoke-Expression. This is the PowerShell equivalent of `curl | bash` and carries the same supply-chain risk.

Locations:

- `cli/action.yml:74`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell

**Notes:**

Fixed two unsafe-shell findings in cli/action.yml:

1. Bash Install Slack CLI: Replaced both `curl -fsSL ... | bash -s` variants (versioned and unversioned) with a safe pattern: download to a mktemp file, then execute with `bash "$installer"`. The temp file is cleaned up afterward.

2. Pwsh Install Slack CLI: Replaced `irm https://... | iex` (PowerShell curl|bash equivalent) with a safe pattern: download via `Invoke-WebRequest -OutFile $installer`, then execute with `& $installer`. The download is now unified for both versioned and unversioned cases, eliminating the dangerous inline execution pattern entirely.

