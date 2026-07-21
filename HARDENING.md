<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v3.0.5** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes remote content directly to bash without first saving it to a file: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This means arbitrary code from the remote server is executed immediately without any integrity check. The 'Pwsh Install Slack CLI' step has the same pattern using PowerShell's `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex` (Invoke-RestMethod piped to Invoke-Expression), which is the PowerShell equivalent of curl|bash.

Locations:

- `cli/action.yml:62`
- `cli/action.yml:64`
- `cli/action.yml:77`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env var `SLACK_COMMAND` (sourced from `inputs.command`) is expanded unquoted when building the `args` string: `args="$SLACK_COMMAND --skip-update"`. The resulting `$args` variable is then also passed unquoted to the `slack` command: `output=$(slack $args 2>&1 | tee /dev/stderr)`. An attacker-controlled `inputs.command` value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can break out of the intended argument context and inject arbitrary shell commands. The same pattern exists in the 'Pwsh Run Slack CLI command' step where `$cliArgs = "$env:SLACK_COMMAND --skip-update"` is later split and passed to `slack $cliArgs.Split(' ')`.

Locations:

- `cli/action.yml:88`
- `cli/action.yml:96`
- `cli/action.yml:107`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed all four vulnerable patterns in hardened/action/cli/action.yml:

1. unsafe-shell (Bash, lines 62-64): Replaced `curl -fsSL ... | bash -s` with downloading the install script to a mktemp file first, then executing it with `bash "$installer"`. The temp file is cleaned up afterwards.

2. unsafe-shell (PowerShell, line 77): Removed the `irm ... | iex` branch. Now always downloads to a temp file with `Invoke-WebRequest -OutFile $installer`, then executes with `& $installer` (with or without the version flag). The temp file is cleaned up afterwards.

3. script-injection (Bash, lines 88/96): Replaced unquoted string `args="$SLACK_COMMAND --skip-update"` and `slack $args` with a bash array: `read -ra cmd_args <<< "$SLACK_COMMAND"`, `args=("${cmd_args[@]}" "--skip-update")`, and `slack "${args[@]}"`. Each element is properly quoted, preventing shell metacharacter injection.

4. script-injection (PowerShell, lines 107): Replaced `$cliArgs = "$env:SLACK_COMMAND --skip-update"` and `$cliArgs.Split(' ')` with a PowerShell array built via `$env:SLACK_COMMAND -split '\s+'`, appending flags individually, and invoking with `slack @cliArgs` (splatting). This keeps each argument as a separate token without relying on string splitting.

