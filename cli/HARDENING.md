<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v4.0.0** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote install script directly to bash without downloading it first: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows a compromised or MITM'd remote server to execute arbitrary code on the runner.

Locations:

- `action.yml:64`
- `action.yml:66`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step uses the PowerShell equivalent of curl|bash: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This pipes a remotely fetched script directly into PowerShell's Invoke-Expression, allowing arbitrary code execution if the remote content is compromised or intercepted.

Locations:

- `action.yml:83`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env var `$SLACK_COMMAND` (sourced from `inputs.command`) is used unquoted when building the args string: `args="$SLACK_COMMAND --skip-update"`, and then `$args` is passed unquoted to the shell: `output=$(slack $args 2>&1 | tee /dev/stderr)`. An attacker-controlled `inputs.command` value containing shell metacharacters (`;`, `|`, `$(...)`, etc.) can break out of the intended command and execute arbitrary shell code.

Locations:

- `action.yml:96`
- `action.yml:105`

### github-env-injection (severity: high)

In the 'Bash Run Slack CLI command' step, the variable `$output` (CLI output derived from user-controlled `$SLACK_COMMAND` / `inputs.command`) is written to `$GITHUB_OUTPUT` via a heredoc without sanitization (`printf '%s' ... | tr -d '\n\r'`): `echo "$output" >> "$GITHUB_OUTPUT"`. If the CLI output contains newlines with content matching `key=value` patterns, or contains the heredoc delimiter `SLACKCLIEOF`, an attacker can inject arbitrary entries into GITHUB_OUTPUT, potentially overwriting other step outputs.

Locations:

- `action.yml:112`
- `action.yml:113`

### github-env-injection (severity: high)

In the 'Pwsh Run Slack CLI command' step, the variable `$output` (CLI output derived from user-controlled `$env:SLACK_COMMAND` / `inputs.command`) is written to `$env:GITHUB_OUTPUT` without sanitization: `$output >> $env:GITHUB_OUTPUT`. Newlines in the output can inject additional key=value pairs or break the heredoc delimiter, allowing an attacker to manipulate subsequent step outputs.

Locations:

- `action.yml:138`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all 5 findings in action.yml:
1. Bash Install Slack CLI: Replaced `curl ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl ... | bash -s` with download-to-tempfile then execute pattern (mktemp + curl -o + bash script). Dropped '--' per instructions since it was the shell's option terminator, not the script's.
2. Pwsh Install Slack CLI: Replaced `irm ... | iex` else-branch with Invoke-WebRequest download to temp file then `& $installer` execution.
3. Bash Run Slack CLI command (script-injection): Tokenized $SLACK_COMMAND via xargs into a bash array using the NUL-delimited read loop pattern, then used `slack "${cmd_args[@]}"` for safe execution with proper argument boundaries.
4. Bash Run Slack CLI command (github-env-injection): Replaced fixed heredoc delimiter `SLACKCLIEOF` with a random `SLACKCLIEOF_$(openssl rand -hex 16)` to prevent attacker-controlled output from injecting the delimiter.
5. Pwsh Run Slack CLI command (github-env-injection): Replaced fixed heredoc delimiter with a random GUID-based `SLACKCLIEOF_$([System.Guid]::NewGuid().ToString("N"))` delimiter. Also improved PowerShell arg handling to use an array with splatting (@cliArgs) instead of string splitting.

