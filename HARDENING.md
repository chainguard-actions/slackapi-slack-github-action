<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v3.0.1** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step in cli/action.yml pipes a remote install script directly to bash without first downloading it to a file. This pattern (`curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`) executes whatever the remote server returns without any integrity verification, enabling supply-chain attacks if the remote URL is compromised.

Locations:

- `cli/action.yml:64`
- `cli/action.yml:66`

### script-injection (severity: high)

Rule (b) violation: In the 'Bash Run Slack CLI command' step, the env var `$SLACK_COMMAND` (sourced from `inputs.command`, an attacker-controlled value) is concatenated into `$args` and then `$args` is expanded unquoted in `output=$(slack $args 2>&1 | tee /dev/stderr)` (line 105). Unquoted expansion of `$args` allows shell word-splitting and glob expansion on the user-controlled command string, enabling injection of additional shell arguments or metacharacters. The variable should be expanded as `"$args"` or the command should be passed as an array.

Locations:

- `cli/action.yml:96`
- `cli/action.yml:105`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed two findings in hardened/action/cli/action.yml:

1. unsafe-shell: Replaced `curl -fsSL ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL ... | bash -s` with a download-then-execute pattern: download to a mktemp file, execute with `bash "$INSTALL_SCRIPT" -v "$SLACK_CLI_VERSION"` or `bash "$INSTALL_SCRIPT"`, then remove the temp file. The `--` was dropped (it was the shell's option terminator in the pipe form, not an argument to the install script).

2. script-injection: Replaced string concatenation `args="$SLACK_COMMAND --skip-update"` and unquoted `slack $args` with a bash array. `SLACK_COMMAND` is tokenized via `read -r -d '' -a args <<< "$SLACK_COMMAND" || true` (whitespace-split, matching upstream's unquoted env var expansion). Flags are appended as array elements (`args+=(--skip-update)`, etc.) and the command is invoked with `slack "${args[@]}"` to prevent word-splitting/glob injection.

### Iteration 2

**Fixes applied:** unsafe-shell, github-env-injection, script-injection

**Notes:**

Fixed all four findings in hardened/action/cli/action.yml:
1. unsafe-shell: The 'else' branch of 'Pwsh Install Slack CLI' used `irm ... | iex`. Fixed by always downloading the installer to a temp file via Invoke-WebRequest and executing it with `& $installer` (with or without the version flag).
2. github-env-injection (Bash, line 107): The fixed heredoc delimiter 'SLACKCLIEOF' was replaced with a random one: `EOF="EOF_$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')`.
3. github-env-injection (Pwsh, line 138): The fixed heredoc delimiter 'SLACKCLIEOF' was replaced with a random GUID-based one: `$eof = "EOF_" + [guid]::NewGuid().ToString('N')`.
4. script-injection (line 122): Replaced `$cliArgs = "$env:SLACK_COMMAND --skip-update"` + `.Split(' ')` with proper PowerShell array tokenization using `-split '\s+'` and `@cliArgs` splatting, with `--token` and its value added as separate array elements.

