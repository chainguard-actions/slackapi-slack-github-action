<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v3.0.1** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes remote content directly to bash without first downloading to a file. Two curl invocations pipe the install script to `bash -s`: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows a compromised or MITM'd remote server to execute arbitrary code on the runner.

Locations:

- `cli/action.yml:57`
- `cli/action.yml:59`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step in cli/action.yml pipes a remotely fetched PowerShell script directly to `iex` (Invoke-Expression), which is the PowerShell equivalent of `curl | bash`: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This allows a compromised or MITM'd remote server to execute arbitrary code on the runner.

Locations:

- `cli/action.yml:71`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the env var `SLACK_COMMAND` (sourced from `inputs.command`) is concatenated into the shell variable `args` and then expanded unquoted in `output=$(slack $args 2>&1 | tee /dev/stderr)`. An unquoted `$args` allows the shell to parse metacharacters (`;`, `|`, `&`, `$(...)`, whitespace, glob chars) from the caller-controlled `inputs.command` value, enabling command injection. The variable should be expanded as `"$args"` or the command should be built as an array.

Locations:

- `cli/action.yml:80`
- `cli/action.yml:87`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed three security findings in hardened/action/cli/action.yml:

1. Bash Install unsafe-shell: Replaced both `curl | bash -s` pipe patterns with download-then-execute: `curl -fsSL ... -o "$INSTALL_SCRIPT"` followed by `bash "$INSTALL_SCRIPT" [-v "$SLACK_CLI_VERSION"]`. Dropped the `--` from the versioned invocation (it was the shell's option terminator for the pipe form, not an argument to the install script).

2. PowerShell Install unsafe-shell: Replaced `irm ... | iex` in the else branch with `Invoke-WebRequest ... -OutFile $installer` followed by `& $installer`. Both version/no-version branches now download to a temp file before executing.

3. Bash Run script-injection: Replaced the unquoted string expansion `slack $args` with a bash array approach. `$SLACK_COMMAND` is tokenized into an array using xargs (quote-aware), additional flags are appended as separate array elements, and the command is invoked as `slack "${args[@]}"` to prevent shell metacharacter injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the PowerShell script injection vulnerability in the 'Pwsh Run Slack CLI command' step of cli/action.yml. The original code interpolated `$env:SLACK_COMMAND` into a PowerShell double-quoted string (`$cliArgs = "$env:SLACK_COMMAND --skip-update"`), which allowed PowerShell metacharacters (e.g., `$(...)`, backticks, semicolons) in `inputs.command` to be evaluated as code. The fix replaces this with a safe array-based approach: the command string is split using `-split '\s+'` (a pure data operation), each token is added as a literal string to a `List[string]`, and the list is splatted to the `slack` command using `@cliArgs`. The `$env:SLACK_TOKEN` value is also added as a separate literal argument rather than being interpolated into a string. No untrusted input is ever evaluated as PowerShell code.

