<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **slackapi--slack-github-action/v3.0.1** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes remote content directly to bash without downloading and verifying it first: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows a compromised or MITM'd remote server to execute arbitrary code on the runner.

Locations:

- `cli/action.yml:64`
- `cli/action.yml:66`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step pipes remote PowerShell content directly to Invoke-Expression (iex): `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This is the PowerShell equivalent of `curl | bash` and allows a compromised or MITM'd remote server to execute arbitrary code on the runner.

Locations:

- `cli/action.yml:80`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env var `$SLACK_COMMAND` (sourced from `inputs.command` via `env: SLACK_COMMAND: ${{ inputs.command }}`) is used unquoted in shell variable expansion: `args="$SLACK_COMMAND --skip-update"` and then `output=$(slack $args 2>&1 | tee /dev/stderr)`. An attacker-controlled `inputs.command` value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can break out of the intended argument context and inject arbitrary shell commands.

Locations:

- `cli/action.yml:92`
- `cli/action.yml:100`

### script-injection (severity: high)

Sub-rule (b): In the 'Pwsh Run Slack CLI command' step, `$env:SLACK_COMMAND` (sourced from `inputs.command`) is interpolated unquoted into a string: `$cliArgs = "$env:SLACK_COMMAND --skip-update"` and then split and passed to `slack $cliArgs.Split(' ')`. A malicious `inputs.command` value with embedded spaces or special characters can inject additional CLI arguments or alter command behavior.

Locations:

- `cli/action.yml:120`
- `cli/action.yml:128`

### github-env-injection (severity: high)

In the 'Bash Run Slack CLI command' step, the raw output of the Slack CLI (`$output`) is written directly to `$GITHUB_OUTPUT` without sanitization: `echo "$output" >> "$GITHUB_OUTPUT"`. If the CLI output contains newline characters followed by `key=value` pairs, an attacker could inject additional entries into GITHUB_OUTPUT, potentially overwriting outputs consumed by downstream steps. The required sanitization (`printf '%s' "$output" | tr -d '\n\r'`) is absent.

Locations:

- `cli/action.yml:107`

### github-env-injection (severity: high)

In the 'Pwsh Run Slack CLI command' step, the raw CLI output (`$output`) is written directly to `$env:GITHUB_OUTPUT` without sanitization: `$output >> $env:GITHUB_OUTPUT`. If the CLI output contains newline-delimited `key=value` content, an attacker could inject additional entries into GITHUB_OUTPUT. The required sanitization step is absent.

Locations:

- `cli/action.yml:131`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all 6 findings in cli/action.yml:
1. unsafe-shell (bash, lines 64/66): Replaced `curl | bash` with download-to-temp-file then execute: `curl -fsSL -o "$installer"` followed by `bash "$installer"`.
2. unsafe-shell (pwsh, line 80): Replaced `irm | iex` with `Invoke-WebRequest -OutFile $installer` followed by `& $installer`, unifying both version/no-version branches.
3. script-injection (bash, lines 92/100): Replaced unquoted string interpolation `args="$SLACK_COMMAND --skip-update"` with array-based approach: `read -ra args <<< "$SLACK_COMMAND"` then `args+=("--skip-update")`, called as `slack "${args[@]}"`.
4. script-injection (pwsh, lines 120/128): Replaced string interpolation `$cliArgs = "$env:SLACK_COMMAND --skip-update"` with array-based approach: `[string[]]$cliArgs = $env:SLACK_COMMAND -split '\s+'` then `$cliArgs += "--skip-update"`, called via splatting `slack @cliArgs`.
5. github-env-injection (bash, line 107): Added `safe_output=$(printf '%s' "$output" | tr -d '\r')` sanitization before writing to GITHUB_OUTPUT via heredoc.
6. github-env-injection (pwsh, line 131): Added `$safe_output = $output -replace "`r`n|`n|`r", "`n"` sanitization before writing to GITHUB_OUTPUT via heredoc.

