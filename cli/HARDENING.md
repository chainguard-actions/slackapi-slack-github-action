<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.3** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes remote content directly to bash without first downloading to a file. Two occurrences: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. If the remote server is compromised or the URL is intercepted, arbitrary code executes on the runner.

Locations:

- `action.yml:63`
- `action.yml:65`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step pipes remote content directly to PowerShell's Invoke-Expression via `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This is the PowerShell equivalent of `curl | bash` and executes whatever the remote server returns without any integrity check.

Locations:

- `action.yml:82`

### script-injection (severity: high)

Rule (b) violation in 'Bash Run Slack CLI command': the env var `$SLACK_COMMAND` (sourced from `inputs.command`, an attacker-controlled value) is assigned to `args` and then expanded unquoted in `output=$(slack $args 2>&1 | tee /dev/stderr)`. An unquoted shell variable expansion allows the shell to parse metacharacters (`;`, `|`, `&`, `$(...)`, etc.) out of the value, enabling command injection. The variable should be expanded as `"$args"` or the command should be restructured to avoid word-splitting.

Locations:

- `action.yml:100`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed three security findings in hardened/action/action.yml:

1. Bash unsafe-shell (lines 63/65): Replaced `curl ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl ... | bash -s` with download-then-execute pattern: `curl -fsSL ... -o "$INSTALL_SCRIPT"` followed by `bash "$INSTALL_SCRIPT" [-v "$SLACK_CLI_VERSION"]`. Dropped the `--` (it was the shell's own option terminator for `bash -s`, not a script argument).

2. PowerShell unsafe-shell (line 82): Eliminated `irm ... | iex` in the else branch. Both version-specified and unspecified cases now download to a temp file with `Invoke-WebRequest -OutFile $installer` and execute with `& $installer [-v $env:SLACK_CLI_VERSION]`, with cleanup via `Remove-Item`.

3. Script-injection (line 100): Replaced unquoted string expansion `slack $args` with a bash array. `$SLACK_COMMAND` is tokenized quote-awarely via `xargs printf '%s\0'` into an array, additional flags appended as separate elements, and invoked as `slack "${args[@]}"` to prevent shell metacharacter injection while preserving correct argument boundaries.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings in action.yml:
1. 'Pwsh Install Slack CLI' step (line 72): Quoted `$env:SLACK_CLI_VERSION` in the installer call: `& $installer -v "$env:SLACK_CLI_VERSION"` to prevent argument splitting/injection.
2. 'Pwsh Run Slack CLI command' step (lines 107, 111): Replaced string concatenation + `.Split(' ')` with a `[System.Collections.Generic.List[string]]` array. SLACK_COMMAND parts are split into individual elements, and SLACK_TOKEN is added as two separate elements (`--token` and `"$env:SLACK_TOKEN"`). The array is splatted to `slack` with `@cliArgs`, preventing injection via embedded spaces in inputs.

