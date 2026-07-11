<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **slackapi--slack-github-action/v3.0.5** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote script directly to bash without first downloading it to a file: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. The script content is never verified before execution.

Locations:

- `cli/action.yml:63`
- `cli/action.yml:65`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step pipes a remote PowerShell script directly to iex (Invoke-Expression) without first downloading it to a file: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This is the PowerShell equivalent of curl | bash.

Locations:

- `cli/action.yml:79`

### script-injection (severity: high)

Rule (b): In the 'Bash Run Slack CLI command' step, the shell variable `$args` is built from `$SLACK_COMMAND` (sourced from `inputs.command`, an attacker-controlled value) and then expanded unquoted in `output=$(slack $args 2>&1 | tee /dev/stderr)`. An unquoted expansion allows the shell to parse metacharacters (`;`, `|`, `&`, `$(...)`, whitespace, glob chars) from the value, enabling command injection.

Locations:

- `cli/action.yml:97`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed three findings in hardened/action/cli/action.yml:
1. Bash Install Slack CLI (lines 63/65): Replaced `curl ... | bash` with download-then-execute pattern using `mktemp` to create a temp file, `curl -fsSL ... -o "$installer"` to download, and `bash "$installer"` to execute.
2. Pwsh Install Slack CLI (line 79): Replaced `irm ... | iex` with download-then-execute pattern using `Invoke-WebRequest -OutFile $installer` and `& $installer` for both the versioned and unversioned cases.
3. Bash Run Slack CLI command (line 97): Replaced string-based `$args` expansion (unquoted) with a bash array. Used `read -ra cmd_args <<< "$SLACK_COMMAND"` to split the command into array elements, built `args` as a proper bash array, and invoked `slack "${args[@]}"` with proper quoting to prevent shell metacharacter injection.

