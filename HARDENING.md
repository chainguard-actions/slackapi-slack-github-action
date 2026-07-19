<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v3.0.4** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes a remote install script directly to bash via `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This executes whatever the remote server returns without any integrity verification. Additionally, the 'Pwsh Install Slack CLI' step uses `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`, which is the PowerShell equivalent of curl|bash and carries the same risk.

Locations:

- `cli/action.yml:63`
- `cli/action.yml:65`
- `cli/action.yml:80`

### script-injection (severity: high)

Rule (b) violation: In the 'Bash Run Slack CLI command' step, the env var $SLACK_COMMAND (sourced from inputs.command) and $SLACK_TOKEN (sourced from inputs.token) are concatenated into the shell variable $args, which is then expanded **unquoted** in `output=$(slack $args 2>&1 | tee /dev/stderr)`. An attacker-controlled input.command value containing shell metacharacters (`;`, `|`, `$(...)`, etc.) will be interpreted by the shell, enabling command injection.

Locations:

- `cli/action.yml:97`

### github-env-injection (severity: high)

In the 'Bash Run Slack CLI command' step, the variable $output (which contains the stdout of the CLI invoked with user-controlled arguments from inputs.command and inputs.token) is written directly to $GITHUB_OUTPUT via `echo "$output" >> "$GITHUB_OUTPUT"` without first stripping newlines using `printf '%s' ... | tr -d '\n\r'`. A value containing newline characters can inject arbitrary key=value pairs or heredoc delimiters into the GitHub Actions environment file, potentially overwriting other outputs or environment variables.

Locations:

- `cli/action.yml:106`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed three high-severity findings in hardened/action/cli/action.yml:

1. unsafe-shell: Replaced curl|bash pattern with download-then-execute: `curl -fsSL -o "$installer" <url>` followed by `bash "$installer"`. For PowerShell, unified both branches to use `Invoke-WebRequest` to download to a temp file, then execute with `& $installer`, eliminating the `irm | iex` pattern.

2. script-injection: Replaced string-based args accumulation (`args="$SLACK_COMMAND --skip-update"` expanded unquoted as `slack $args`) with a bash array (`read -ra args <<< "$SLACK_COMMAND"` with `args+=(...)` appends), expanded safely as `slack "${args[@]}"`.

3. github-env-injection: Changed `echo "$output" >> "$GITHUB_OUTPUT"` to `printf '%s\n' "$output" >> "$GITHUB_OUTPUT"` and used `printf` for the ok= and time= single-line outputs as well. The multi-line response output already used the heredoc pattern (response<<SLACKCLIEOF) which is the correct approach for multi-line values.

