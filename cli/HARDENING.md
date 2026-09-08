<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v4.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote install script directly to bash without first downloading it to a file. Two occurrences: (1) `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and (2) `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows a compromised or MITM'd remote server to execute arbitrary code on the runner.

Locations:

- `action.yml:64`
- `action.yml:66`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step uses `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`, which is the PowerShell equivalent of `curl | bash`. It fetches a remote script and immediately executes it via Invoke-Expression without saving to disk first, allowing arbitrary code execution if the remote content is compromised or intercepted.

Locations:

- `action.yml:83`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env vars `$SLACK_COMMAND` (from `inputs.command`) and `$SLACK_TOKEN` (from `inputs.token`) are expanded unquoted inside shell string concatenation. Specifically: `args="$SLACK_COMMAND --skip-update"` (line 96) and `args="$args --token $SLACK_TOKEN"` (line 101). An attacker-controlled input containing shell metacharacters (`;`, `|`, `$(...)`, etc.) can break out of the string and inject arbitrary shell commands.

Locations:

- `action.yml:96`
- `action.yml:101`

### script-injection (severity: high)

Sub-rule (b): In the 'Pwsh Run Slack CLI command' step, `$env:SLACK_COMMAND` (from `inputs.command`) and `$env:SLACK_TOKEN` (from `inputs.token`) are interpolated unquoted into PowerShell string concatenation: `$cliArgs = "$env:SLACK_COMMAND --skip-update"` (line 123) and `$cliArgs = "$cliArgs --token $env:SLACK_TOKEN"` (line 127). Attacker-controlled values can inject additional CLI arguments or PowerShell metacharacters.

Locations:

- `action.yml:123`
- `action.yml:127`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed all four findings in action.yml:

1. Bash Install Slack CLI (unsafe-shell): Replaced both `curl | bash` pipe patterns with download-then-execute: `curl -fsSL ... -o "$INSTALL_SCRIPT"` followed by `bash "$INSTALL_SCRIPT" [-v "$SLACK_CLI_VERSION"]`. Dropped the `--` per the rules (it was the shell's option terminator, not the script's).

2. Pwsh Install Slack CLI (unsafe-shell): Replaced `irm ... | iex` with `Invoke-WebRequest -Uri ... -OutFile $installer` followed by `& $installer [-v $env:SLACK_CLI_VERSION]`. Both branches now download first then execute.

3. Bash Run Slack CLI command (script-injection): Replaced string concatenation of `$SLACK_COMMAND` and `$SLACK_TOKEN` with a bash array. `$SLACK_COMMAND` is tokenized via xargs (quote-aware) into `cmd_args`; `$SLACK_TOKEN` is appended as `--token "$SLACK_TOKEN"`. Command invoked as `slack "${cmd_args[@]}"`.

4. Pwsh Run Slack CLI command (script-injection): Replaced PowerShell string concatenation with a `[System.Collections.Generic.List[string]]` array. `$env:SLACK_COMMAND` is split on whitespace into individual tokens; `$env:SLACK_TOKEN` is added as two separate elements. Command invoked with splatting `slack @cliArgs`.

