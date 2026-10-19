<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v4.0.0** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote install script directly to bash without first downloading it to a file: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows the remote server to serve arbitrary code that is immediately executed by the shell.

Locations:

- `action.yml:60`
- `action.yml:62`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step pipes a remote PowerShell script directly to Invoke-Expression (iex): `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This is the PowerShell equivalent of `curl | bash` — remote content is fetched and immediately executed without any integrity check.

Locations:

- `action.yml:74`

### script-injection (severity: high)

Rule (b) violation in 'Bash Run Slack CLI command': the env var `SLACK_COMMAND` holds `${{ inputs.command }}` (attacker-controlled) and is assigned unquoted to `args`, then expanded unquoted in `output=$(slack $args 2>&1 | tee /dev/stderr)`. An unquoted shell variable expansion of untrusted data allows shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) in the input to be interpreted by the shell, enabling command injection.

Locations:

- `action.yml:91`

### github-env-injection (severity: high)

In 'Bash Run Slack CLI command', the variable `$output` — which contains the stdout of the Slack CLI invoked with user-controlled `$SLACK_COMMAND` — is written to `$GITHUB_OUTPUT` without sanitization: `echo "$output" >> "$GITHUB_OUTPUT"`. If the CLI output contains newline characters, an attacker can inject additional key=value pairs into the output file, potentially overwriting subsequent step outputs or injecting environment variables.

Locations:

- `action.yml:97`

### github-env-injection (severity: high)

In 'Pwsh Run Slack CLI command', the variable `$output` — which contains the stdout of the Slack CLI invoked with user-controlled `$env:SLACK_COMMAND` — is written to `$env:GITHUB_OUTPUT` without sanitization: `$output >> $env:GITHUB_OUTPUT`. If the CLI output contains newline characters, an attacker can inject additional key=value pairs into the output file.

Locations:

- `action.yml:122`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all 5 findings in hardened/action/action.yml:
1. Bash Install Slack CLI: Replaced `curl | bash -s -- -v ...` and `curl | bash -s` with download-to-tempfile then execute pattern (mktemp + curl -o + bash). Dropped the '--' shell option terminator per instructions.
2. Pwsh Install Slack CLI: Replaced `irm ... | iex` with Invoke-WebRequest download to temp file then `& $installer` (with or without -v flag).
3. Bash Run Slack CLI command: Replaced unquoted string-based args assembly with a bash array using `read -r -d '' -a args <<< "$SLACK_COMMAND"` for whitespace tokenization, then `slack "${args[@]}"` to prevent shell metacharacter injection.
4. Bash Run Slack CLI command github-env-injection: Replaced fixed heredoc delimiter 'SLACKCLIEOF' with a random one generated from /dev/urandom.
5. Pwsh Run Slack CLI command github-env-injection: Replaced fixed heredoc delimiter 'SLACKCLIEOF' with a random one generated from [guid]::NewGuid().ToString('N').

