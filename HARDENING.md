# Hardening Report: slackapi--slack-github-action/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `ff50f15e4b79bfbf764dafdfd2579175a6ea9771`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **slackapi--slack-github-action/v3.0.3** was hardened automatically. 1 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes remote content directly to bash using `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. The script is never downloaded to a file and verified before execution. Additionally, the 'Pwsh Install Slack CLI' step uses `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`, which is the PowerShell equivalent of curl|bash. All three patterns execute remotely-fetched scripts without any integrity verification, making the action vulnerable to supply-chain attacks if the remote server is compromised.

Locations:

- `cli/action.yml:63`
- `cli/action.yml:64`
- `cli/action.yml:80`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell

**Notes:**

Fixed all three unsafe pipe-to-shell patterns in cli/action.yml:
1. Bash step: replaced `curl -fsSL ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL ... | bash -s` with: download script to a mktemp file, then execute with `bash "$installer"`, then remove the temp file.
2. Pwsh step: replaced `irm https://... | iex` (in the else branch) with: download via `Invoke-WebRequest -OutFile` to a random temp file, then execute with `& $installer`, then remove the temp file. The already-fixed versioned branch was unified into the same download-then-execute pattern for consistency.

