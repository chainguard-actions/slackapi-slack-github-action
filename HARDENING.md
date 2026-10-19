<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v4.0.0** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote install script directly to bash without first downloading and inspecting it: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` (line 64) and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s` (line 66). The 'Pwsh Install Slack CLI' step does the same in PowerShell: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex` (line 83). Remote content is executed immediately without any integrity verification.

Locations:

- `cli/action.yml:64`
- `cli/action.yml:66`
- `cli/action.yml:83`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env var $SLACK_COMMAND (sourced from inputs.command) is expanded unquoted: `args="$SLACK_COMMAND --skip-update"` (line 96). The resulting $args variable is then also passed unquoted to the shell: `output=$(slack $args 2>&1 | tee /dev/stderr)` (line 105). An attacker-controlled inputs.command value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can break out of the intended argument context and execute arbitrary shell commands.

Locations:

- `cli/action.yml:96`
- `cli/action.yml:105`

### github-env-injection (severity: high)

In the 'Bash Run Slack CLI command' step, the variable $output — which contains the stdout of running `slack $args` where $args is derived from the untrusted inputs.command — is written to $GITHUB_OUTPUT without sanitization: `echo "$output" >> "$GITHUB_OUTPUT"` (line 113). Because $output can contain newlines, an attacker-controlled command output could inject additional key=value pairs into GITHUB_OUTPUT (e.g. by embedding a newline followed by `some_key=malicious_value`). The same issue exists in the 'Pwsh Run Slack CLI command' step where `$output >> $env:GITHUB_OUTPUT` (line 142) writes unsanitized command output to GITHUB_OUTPUT. Neither write is preceded by the required `printf '%s' ... | tr -d '\n\r'` sanitization step.

Locations:

- `cli/action.yml:113`
- `cli/action.yml:142`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all three findings in hardened/action/cli/action.yml:

1. unsafe-shell (bash, lines 64/66): Download install.sh to a mktemp file first, then execute 'bash "$INSTALL_SCRIPT" [-v "$SLACK_CLI_VERSION"]'. Dropped '--' (it was the shell's option terminator in the pipe form, not the script's argument).

2. unsafe-shell (pwsh, line 83): Unified both branches to always download install-windows.ps1 to a temp file first, then execute with '& $installer [-v $env:SLACK_CLI_VERSION]'. Removed 'irm | iex' pipe.

3. script-injection (lines 96/105): Replaced unquoted string concatenation with a bash array. $SLACK_COMMAND is split via 'read -r -d "" -a args <<< "$SLACK_COMMAND" || true' (whitespace splitting, matching upstream's unquoted env var expansion). Additional flags appended as separate array elements. Command invoked as 'slack "${args[@]}"'.

4. github-env-injection (bash, line 113): Replaced fixed 'SLACKCLIEOF' heredoc delimiter with a random one from /dev/urandom. Multi-line $output preserved as-is between random delimiters.

5. github-env-injection (pwsh, line 142): Replaced fixed 'SLACKCLIEOF' with a random delimiter via [guid]::NewGuid().ToString('N').

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in cli/action.yml 'Pwsh Run Slack CLI command' step. Replaced string interpolation pattern ($cliArgs = "$env:SLACK_COMMAND --skip-update" followed by $cliArgs.Split(' ')) with a PowerShell array built element-by-element. SLACK_COMMAND is now tokenized with '-split '\s+' | Where-Object { $_ }' (whitespace split, matching the bash step's read -a behavior), and SLACK_TOKEN is added as a separate '--token' + value pair to the array. The array is passed to slack via PowerShell splatting (@cliArgs), preserving argument boundaries and preventing injection through spaces or special characters in attacker-controlled inputs.

