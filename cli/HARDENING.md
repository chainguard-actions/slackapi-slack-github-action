<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.1** was hardened automatically. 6 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes remote shell scripts directly to bash without first downloading to a file: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"` and `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. This allows a compromised or MITM'd remote server to execute arbitrary code. The script should be downloaded to a temporary file, verified, and then executed separately.

Locations:

- `action.yml:60`
- `action.yml:62`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step pipes a remote PowerShell script directly to Invoke-Expression (iex): `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. This is the PowerShell equivalent of `curl | bash` and allows a compromised or MITM'd remote server to execute arbitrary code. The script should be downloaded to a file first (as the step already does in the non-else branch), verified, and then executed.

Locations:

- `action.yml:73`

### script-injection (severity: high)

Sub-rule (b): In the 'Bash Run Slack CLI command' step, the variable `$args` is built from `$SLACK_COMMAND` (which holds `inputs.command`, a caller-controlled value) and then used **unquoted** in `output=$(slack $args 2>&1 | tee /dev/stderr)`. An unquoted shell variable expansion allows shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) embedded in `inputs.command` to be interpreted by the shell, enabling command injection. The variable should be quoted: `output=$(slack "$args" 2>&1 | tee /dev/stderr)` or, better, use an array to hold arguments.

Locations:

- `action.yml:91`

### script-injection (severity: high)

Sub-rule (b): In the 'Pwsh Run Slack CLI command' step, `$cliArgs` is built by string-concatenating `$env:SLACK_COMMAND` (from `inputs.command`) and then split with `.Split(' ')` before being passed to `slack`. Splitting on spaces is fragile and does not handle shell metacharacters or quoted arguments safely, allowing a caller to inject unexpected arguments or manipulate command execution via specially crafted `inputs.command` values.

Locations:

- `action.yml:120`

### github-env-injection (severity: high)

In the 'Bash Run Slack CLI command' step, `echo "$output" >> "$GITHUB_OUTPUT"` writes the CLI's stdout (derived from running `slack` with caller-controlled `inputs.command`) to `$GITHUB_OUTPUT` without sanitization. Although a heredoc delimiter (`<<SLACKCLIEOF`) is used, the `$output` value itself is not sanitized with `printf '%s' "$output" | tr -d '\n\r'` before writing. A malicious CLI response containing a line matching `SLACKCLIEOF` could break out of the heredoc and inject arbitrary key=value pairs into `$GITHUB_OUTPUT`, potentially poisoning downstream step outputs.

Locations:

- `action.yml:97`

### github-env-injection (severity: high)

In the 'Pwsh Run Slack CLI command' step, `$output >> $env:GITHUB_OUTPUT` writes the CLI's stdout (derived from caller-controlled `inputs.command`) to `$GITHUB_OUTPUT` without sanitization. A heredoc delimiter (`<<SLACKCLIEOF`) is used, but `$output` is not sanitized before writing. A malicious CLI response containing a line matching `SLACKCLIEOF` could break out of the heredoc and inject arbitrary key=value pairs into `$GITHUB_OUTPUT`.

Locations:

- `action.yml:126`

## Iteration Notes

### Iteration 1

**Fixes applied:** unsafe-shell, script-injection, github-env-injection

**Notes:**

Fixed all 6 findings in action.yml:
1. Bash Install: replaced 'curl | bash -s -- -v ...' and 'curl | bash -s' with download to mktemp file + 'bash "$INSTALL_SCRIPT" -v ...' / 'bash "$INSTALL_SCRIPT"' (dropped '--' per instructions).
2. Pwsh Install: replaced 'irm ... | iex' with Invoke-WebRequest to temp file + '& $installer', unified download outside if/else.
3. Bash Run: replaced unquoted string $args with bash array; split SLACK_COMMAND with 'read -r -d '' -a'; used 'slack "${args[@]}"'.
4. Pwsh Run: replaced .Split(' ') string with PowerShell array built via '-split \s+' and splatted with @cliArgs.
5. Bash GITHUB_OUTPUT: replaced fixed 'SLACKCLIEOF' heredoc delimiter with random 'EOF_$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')'.
6. Pwsh GITHUB_OUTPUT: replaced fixed 'SLACKCLIEOF' heredoc delimiter with random '[guid]::NewGuid().ToString("N")'-based delimiter.

