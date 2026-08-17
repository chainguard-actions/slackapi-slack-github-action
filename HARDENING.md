<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v4.0.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes remote content directly to bash without first downloading to a file. Two occurrences: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` (line 64) and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s` (line 66). The script should be downloaded to a temporary file, verified, and then executed separately.

Locations:

- `cli/action.yml:64`
- `cli/action.yml:66`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step pipes remote content directly to PowerShell's Invoke-Expression (iex): `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This is equivalent to `curl | bash` and executes remotely-fetched code without any integrity check. The script should be downloaded to a file first and then executed separately.

Locations:

- `cli/action.yml:83`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env vars `$SLACK_COMMAND` (from `inputs.command`) and `$SLACK_TOKEN` (from `inputs.token`) are expanded unquoted in the shell, allowing an attacker-controlled input to inject shell metacharacters. Offending lines: `args="$SLACK_COMMAND --skip-update"` (line 96, unquoted `$SLACK_COMMAND`), `args="$args --token $SLACK_TOKEN"` (line 101, unquoted `$SLACK_TOKEN`), and `output=$(slack $args 2>&1 | tee /dev/stderr)` (line 105, unquoted `$args`). All of these should use double-quoted expansions such as `"$SLACK_COMMAND"` and `"$args"`.

Locations:

- `cli/action.yml:96`
- `cli/action.yml:101`
- `cli/action.yml:105`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed three security findings in hardened/action/cli/action.yml:

1. unsafe-shell (bash): Replaced both `curl | bash -s -- -v ...` and `curl | bash -s` patterns with download-then-execute: script is saved to a mktemp file, then executed with `bash "$INSTALL_SCRIPT" [-v "$SLACK_CLI_VERSION"]`. The `--` was dropped as it was the shell's option terminator for the pipe form, not the script's argument.

2. unsafe-shell (pwsh): Replaced `irm ... | iex` in the else-branch with `Invoke-WebRequest -Uri ... -OutFile $installer` followed by `& $installer`, consistent with the already-fixed versioned branch.

3. script-injection: Replaced string-concatenation args building with a bash array. SLACK_COMMAND (a list input) is tokenized via xargs into an array to preserve quoted sub-arguments. SLACK_TOKEN is appended as `("--token" "$SLACK_TOKEN")` keeping it properly double-quoted. The slack invocation uses `"${args[@]}"` for safe expansion.

