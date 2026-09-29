<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v4.0.0** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes a remote script directly to bash without first saving it to a file: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows the remote server to serve arbitrary code that is immediately executed. The 'Pwsh Install Slack CLI' step similarly uses `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`, which is the PowerShell equivalent of curl | bash.

Locations:

- `cli/action.yml:59`
- `cli/action.yml:60`
- `cli/action.yml:75`

### script-injection (severity: high)

Rule (b) violation in the 'Bash Run Slack CLI command' step: env vars `$SLACK_COMMAND` (from `inputs.command`) and `$SLACK_TOKEN` (from `inputs.token`) are expanded unquoted inside the run script. Specifically: `args="$SLACK_COMMAND --skip-update"` and `args="$args --token $SLACK_TOKEN"` leave the values unquoted, and `output=$(slack $args 2>&1 | tee /dev/stderr)` passes `$args` unquoted, allowing shell metacharacter injection (`;`, `|`, `&`, `$(...)`, etc.) from attacker-controlled inputs.

Locations:

- `cli/action.yml:83`
- `cli/action.yml:87`
- `cli/action.yml:90`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed cli/action.yml:
1. unsafe-shell (bash, lines 59-60): Replaced `curl ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl ... | bash -s` with downloading the install script to a mktemp temp file, then executing `bash "$INSTALL_SCRIPT" -v "$SLACK_CLI_VERSION"` or `bash "$INSTALL_SCRIPT"`. The `--` was dropped (it was the shell's option terminator for stdin mode, not needed when running a file). Temp file is removed afterward.
2. unsafe-shell (PowerShell, line 75): Replaced `irm ... | iex` with always downloading via `Invoke-WebRequest` to a temp file, then executing with `& $installer` (with or without `-v $env:SLACK_CLI_VERSION`). Temp file is removed afterward.
3. script-injection (lines 83, 87, 90): Replaced unquoted string concatenation (`args="$SLACK_COMMAND --skip-update"`, etc.) with a bash array. SLACK_COMMAND is tokenized quote-awarely via xargs into the array (handles commands like `deploy --app myapp` or `sh -c "exit 0"`). SLACK_TOKEN is added as properly quoted separate tokens `--token "$SLACK_TOKEN"`. Command invoked as `slack "${args[@]}"` preserving all argument boundaries.

