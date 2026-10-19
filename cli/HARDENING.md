<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.3** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remotely fetched shell script directly to bash without first downloading it to a file for inspection. Both the versioned and unversioned install paths use `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. If the remote server is compromised or the connection is intercepted, arbitrary code executes on the runner.

Locations:

- `action.yml:61`
- `action.yml:63`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step uses `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex` (Invoke-RestMethod piped to Invoke-Expression), which is the PowerShell equivalent of `curl | bash`. The remote script is executed immediately without being saved to disk first, giving a compromised or intercepted response full code execution on the runner.

Locations:

- `action.yml:74`

### script-injection (severity: high)

Rule (b) violation: In the 'Bash Run Slack CLI command' step, the env var `$SLACK_COMMAND` (sourced from `inputs.command`, a required caller-controlled input) is concatenated into `$args` and then expanded unquoted in the command substitution `output=$(slack $args 2>&1 | tee /dev/stderr)`. An unquoted `$args` allows the shell to perform word-splitting and glob expansion on the value, enabling a caller to inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) into the executed command.

Locations:

- `action.yml:91`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection

**Notes:**

Fixed three findings in hardened/action/action.yml:

1. Bash Install Slack CLI (unsafe-shell, lines 61/63): Replaced `curl ... | bash -s` pipe pattern with downloading the script to a mktemp file first, then executing it with bash. Dropped the '--' from the versioned invocation (it was the shell's option terminator in the pipe form, not an installer argument).

2. Pwsh Install Slack CLI (unsafe-shell, line 74): Replaced the `irm ... | iex` else branch with downloading the script via Invoke-WebRequest to a temp file, then executing with `& $installer`. Now both versioned and unversioned paths download first.

3. Bash Run Slack CLI command (script-injection, line 91): Replaced string concatenation + unquoted `$args` expansion with a bash array. SLACK_COMMAND is tokenized via `read -r -d '' -a args <<< "$SLACK_COMMAND" || true` (whitespace splitting, matching upstream's unquoted env var behavior), additional flags appended as separate array elements, and invoked as `slack "${args[@]}"`.

