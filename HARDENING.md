<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **slackapi--slack-github-action/v3.0.3** was hardened automatically. 2 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote shell script directly to bash without downloading it to a file first: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` (line 64) and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s` (line 66). Similarly, the 'Pwsh Install Slack CLI' step uses the PowerShell equivalent `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex` (line 83), which downloads and immediately executes a remote script. These patterns allow a compromised or malicious remote server to execute arbitrary code on the runner.

Locations:

- `cli/action.yml:64`
- `cli/action.yml:66`
- `cli/action.yml:83`

### script-injection (severity: high)

Rule (b) violation — unquoted shell variable expansion of untrusted data. In the 'Bash Run Slack CLI command' step, `inputs.command` is placed into the `SLACK_COMMAND` env var and then assigned to `args` via `args="$SLACK_COMMAND --skip-update"`. The `args` variable is subsequently expanded unquoted in `output=$(slack $args 2>&1 | tee /dev/stderr)` (line 105). Because `$args` is not double-quoted, the shell will word-split and glob-expand its contents, allowing an attacker-controlled `inputs.command` value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) to inject arbitrary shell commands.

Locations:

- `cli/action.yml:96`
- `cli/action.yml:105`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed three unsafe-shell instances and one script-injection instance in cli/action.yml:
1. Bash Install Slack CLI: Replaced `curl ... | bash` with download-to-tempfile then execute pattern using `mktemp`, `curl -fsSL -o`, and `bash "$installer"`.
2. Pwsh Install Slack CLI: Eliminated the `irm ... | iex` pattern by always downloading to a temp file first with `Invoke-WebRequest -OutFile`, then executing with `& $installer`.
3. Bash Run Slack CLI command: Replaced the unquoted string-based `$args` variable with a bash array (`read -ra args <<< "$SLACK_COMMAND"` + `args+=(...)`) and invoked with `slack "${args[@]}"` to prevent word-splitting and glob-expansion of attacker-controlled input.

