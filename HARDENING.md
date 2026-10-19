<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v3.0.3** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes a remote install script directly to bash without first downloading and verifying it: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows a compromised or MITM'd remote server to execute arbitrary code on the runner. The script should be downloaded to a temporary file, its integrity verified (e.g. via checksum), and then executed separately.

Locations:

- `cli/action.yml:68`
- `cli/action.yml:70`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step in cli/action.yml uses PowerShell's `irm ... | iex` idiom (`irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`), which is the PowerShell equivalent of `curl | bash` — it downloads and immediately executes a remote script without any integrity verification. A compromised or MITM'd remote server could execute arbitrary code on the runner.

Locations:

- `cli/action.yml:85`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell

**Notes:**

Fixed both unsafe-shell findings in cli/action.yml:
1. Bash Install Slack CLI (lines 68/70): Replaced `curl ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl ... | bash -s` with: download script to a mktemp file, then execute with `bash "$INSTALL_SCRIPT" -v "$SLACK_CLI_VERSION"` or `bash "$INSTALL_SCRIPT"`. Dropped the `--` (it was the shell's stdin option terminator, not a script argument). Temp file is removed after execution.
2. Pwsh Install Slack CLI (line 85): Replaced `irm ... | iex` with `Invoke-WebRequest` to download to a temp file, then `& $installer` (with or without `-v` flag). The download is now done once before the conditional, and the temp file is cleaned up with `Remove-Item` afterward.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed the 'Bash Run Slack CLI command' step in cli/action.yml:
1. SLACK_COMMAND (a whitespace-separated list input expanded as env var) is now tokenized into a bash array using `read -r -d '' -a args <<< "$SLACK_COMMAND" || true`, matching upstream's whitespace-splitting behavior without xargs quote-awareness.
2. Additional flags (--skip-update, --verbose, --token) are appended to the array with `args+=(...)` keeping each token separate.
3. SLACK_TOKEN is now double-quoted as `"$SLACK_TOKEN"` when passed as the --token value.
4. The command is invoked as `slack "${args[@]}"` to preserve argument boundaries.
5. Also fixed the fixed heredoc delimiter SLACKCLIEOF by replacing it with a random delimiter from /dev/urandom to prevent heredoc injection.

