<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.3

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.3** was hardened automatically. 8 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Step 'Bash Run Slack CLI command' writes $output to $GITHUB_OUTPUT between a FIXED heredoc delimiter: `echo "response<<SLACKCLIEOF" >> "$GITHUB_OUTPUT"; echo "$output" >> "$GITHUB_OUTPUT"; echo "SLACKCLIEOF" >> "$GITHUB_OUTPUT"`. $output is the CLI output of a command built from inputs.command/inputs.token. If that output contains a line 'SLACKCLIEOF', the block ends early and an attacker can inject extra outputs. The delimiter must be generated from /dev/urandom, openssl rand or uuidgen on every run.

Locations:

- `action.yml:112`

### github-env-injection (severity: high)

Step 'Pwsh Run Slack CLI command' writes $output to $env:GITHUB_OUTPUT between a FIXED heredoc delimiter: `"response<<SLACKCLIEOF" >> $env:GITHUB_OUTPUT; $output >> $env:GITHUB_OUTPUT; "SLACKCLIEOF" >> $env:GITHUB_OUTPUT`. $output comes from a command run with inputs.command/inputs.token. The delimiter should be random, e.g. built with [guid]::NewGuid().

Locations:

- `action.yml:144`

### github-env-injection (severity: high)

Step 'Bash Add Slack CLI to PATH' writes an inherited process env var to $GITHUB_PATH without sanitization: `echo "$HOME/.slack/bin" >> "$GITHUB_PATH"`. A composite action inherits HOME from its caller, and the script reads it without setting it. The `printf '%s' ... | tr -d '\n\r'` step is required before the write.

Locations:

- `action.yml:49`

### github-env-injection (severity: high)

Step 'Pwsh Add Slack CLI to PATH' writes an inherited process env var to $GITHUB_PATH without stripping newlines: `Add-Content -Path $env:GITHUB_PATH -Value "$env:USERPROFILE\AppData\Local\slack-cli\bin"`. USERPROFILE is inherited from the caller's environment, so CR/LF should be removed before the write.

Locations:

- `action.yml:54`

### script-injection (severity: high)

Sub-rule (b): step 'Bash Run Slack CLI command' builds `args` from the env vars SLACK_COMMAND (inputs.command) and SLACK_TOKEN (inputs.token), then expands it unquoted: `output=$(slack $args 2>&1 | tee /dev/stderr)`. The shell word-splits and glob-expands the input data. The arguments should be built as a bash array and expanded as "${arr[@]}".

Locations:

- `action.yml:105`

### unsafe-shell (severity: high)

Remote script piped straight to a shell: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"`.

Locations:

- `action.yml:64`

### unsafe-shell (severity: high)

Remote script piped straight to a shell: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`.

Locations:

- `action.yml:66`

### unsafe-shell (severity: high)

Remote PowerShell script piped straight to Invoke-Expression: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`.

Locations:

- `action.yml:83`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, unsafe-shell

**Notes:**

Random heredoc delimiters (urandom in bash, GUID in pwsh) for response output; PATH writes stripped of CR/LF in both bash and pwsh; slack args built as a bash array (whitespace split of SLACK_COMMAND via read -r -d '' -a, matching upstream's unquoted $args split, token kept as separate quoted argument); installers downloaded to a file then executed (dropped '-s --'), pwsh irm|iex replaced by download-then-invoke.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

I fixed all three findings in action.yml. I did not run the action after the edits.

- **Bash step:** the fixed SLACKCLIEOF delimiter is replaced with a random one, made each run with `od -An -N16 -tx1 /dev/urandom`. The response is written unchanged, so multi-line output keeps its lines.
- **Pwsh step:** the fixed delimiter is replaced with `"EOF_" + [guid]::NewGuid().ToString('N')`.
- **Pwsh Add Slack CLI to PATH:** CR/LF characters are now removed from `$env:USERPROFILE` before it is written to `$GITHUB_PATH`. I also removed the `zizmor: ignore[github-env]` comment that told the scanner to skip that line, since the line no longer needs the exception.

### Iteration 3

**Fixes applied:** github-env-injection

**Notes:**

Fixed the broken lines in the 'Bash Run Slack CLI command' step. EOF is now assigned from /dev/urandom on its own line. The { response<<$EOF ... } >> $GITHUB_OUTPUT block is on its own line and still holds the full multi-line output. The leftover fixed SLACKCLIEOF echo is removed. The time=$(date +%s) output line is restored.

