<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **slackapi--slack-github-action/v3.0.4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes remote content directly to bash without first saving it to a file: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` (line 65) and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s` (line 67). Additionally, the 'Pwsh Install Slack CLI' step uses PowerShell's equivalent pattern `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex` (line 84), which downloads and immediately executes remote content. These patterns prevent inspection of the downloaded content before execution.

Locations:

- `cli/action.yml:65`
- `cli/action.yml:67`
- `cli/action.yml:84`

### script-injection (severity: high)

Rule (b) violation: In the 'Bash Run Slack CLI command' step, the env vars SLACK_COMMAND (sourced from inputs.command) and SLACK_TOKEN (sourced from inputs.token) are used unquoted in shell variable assignments and command execution. Line 97: `args="$SLACK_COMMAND --skip-update"` — $SLACK_COMMAND is unquoted, allowing shell metacharacter injection. Line 102: `args="$args --token $SLACK_TOKEN"` — $SLACK_TOKEN is unquoted. Line 106: `output=$(slack $args 2>&1 | tee /dev/stderr)` — $args is unquoted, allowing word-splitting and glob expansion of attacker-controlled content. These env vars are set from inputs.command and inputs.token which are caller-controlled.

Locations:

- `cli/action.yml:97`
- `cli/action.yml:102`
- `cli/action.yml:106`

### github-env-injection (severity: high)

In the 'Bash Run Slack CLI command' step, the variable $output (containing the raw stdout/stderr of the CLI command, which is derived from the caller-controlled inputs.command) is written directly to $GITHUB_OUTPUT without sanitization: `echo "$output" >> "$GITHUB_OUTPUT"` (line 114). If the CLI output contains newlines, this can inject additional key=value pairs or heredoc delimiters into GITHUB_OUTPUT, potentially allowing output manipulation. The required sanitization step (`printf '%s' "$output" | tr -d '\n\r'`) is absent.

Locations:

- `cli/action.yml:114`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed three security issues in hardened/action/cli/action.yml:
1. unsafe-shell: Replaced both `curl | bash` patterns with download-then-execute: `curl -fsSL ... -o "$installer"` followed by `bash "$installer"`. Also replaced the PowerShell `irm | iex` pattern with `Invoke-WebRequest -OutFile $installer` followed by `& $installer`.
2. script-injection: Replaced unquoted string concatenation for args with a bash array. Used `read -ra cmd_parts <<< "$SLACK_COMMAND"` to safely split the command, built `args` as a proper bash array with quoted elements, and invoked `slack "${args[@]}"` to prevent word-splitting and glob expansion.
3. github-env-injection: Added `safe_output=$(printf '%s' "$output" | tr -d '\r')` to sanitize the CLI output before writing to GITHUB_OUTPUT. The existing heredoc pattern (<<SLACKCLIEOF) already handles newline safety for the delimiter, and the additional sanitization prevents carriage-return injection.

