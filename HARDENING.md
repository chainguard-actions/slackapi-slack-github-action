<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action/v3.0.3** was hardened automatically. 2 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remotely fetched shell script directly to bash without first downloading and verifying it: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` (and the else branch `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`). If the remote URL is compromised or subject to a MITM attack, arbitrary code executes on the runner immediately.

Locations:

- `cli/action.yml:75`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step uses the PowerShell equivalent of curl|bash: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This fetches a remote PowerShell script and immediately executes it via Invoke-Expression, with no download-then-verify step.

Locations:

- `cli/action.yml:93`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell

**Notes:**

Fixed two unsafe-shell findings in hardened/action/cli/action.yml:
1. Bash Install Slack CLI step: Replaced `curl ... | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl ... | bash -s` with a download-then-execute pattern: curl downloads install.sh to a mktemp file, then bash executes it directly. Dropped `-s` and `--` from the invocation (they were shell stdin-reading flags, not script arguments).
2. Pwsh Install Slack CLI step: Replaced the `else` branch `irm ... | iex` with a download-then-execute pattern using `Invoke-WebRequest -OutFile` followed by `& $installer`. Both branches now share the same download step, with conditional invocation based on whether SLACK_CLI_VERSION is set. Temp file is cleaned up with Remove-Item afterward.

### Iteration 2

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed cli/action.yml 'Bash Run Slack CLI command' step: (1) script-injection: replaced unquoted string-based args variable with a bash array; SLACK_COMMAND is tokenized via xargs (quote-aware, handles embedded quotes) into array elements, SLACK_TOKEN is appended as separate quoted array elements '--token' and "$SLACK_TOKEN", and the command is invoked as `slack "${args[@]}"` keeping argument boundaries intact. (2) github-env-injection: replaced the static 'SLACKCLIEOF' heredoc delimiter with a randomly generated one `SLACKCLIEOF_$(openssl rand -hex 16)` so attacker-controlled output cannot contain the exact delimiter and inject additional key=value pairs into GITHUB_OUTPUT; also switched from echo to printf for writing output values.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in the 'Pwsh Run Slack CLI command' step of cli/action.yml. The original code built a string by interpolating $env:SLACK_COMMAND and $env:SLACK_TOKEN, then split on spaces with .Split(' '), which allowed injection via spaces or PowerShell metacharacters in the input. The fix uses PowerShell's own AST parser ([System.Management.Automation.Language.Parser]::ParseInput) to safely tokenize SLACK_COMMAND into an array of arguments, then appends --skip-update, --verbose (conditionally), and --token + SLACK_TOKEN (conditionally) as separate array elements. The final invocation uses PowerShell splatting (@cliArgs) so each element is passed as a distinct argument to slack, preventing argument injection.

