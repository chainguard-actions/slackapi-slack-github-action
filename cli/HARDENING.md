<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v4.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v4.0.0** was hardened automatically. 8 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Step 'Bash Run Slack CLI command' writes $output to $GITHUB_OUTPUT between a fixed heredoc delimiter: echo "response<<SLACKCLIEOF" / echo "$output" / echo "SLACKCLIEOF". $output is the output of a CLI command built from inputs.command and inputs.token. If that output contains a line 'SLACKCLIEOF', the block ends early and the lines after it become new output entries. Fix: use a random delimiter (e.g. from /dev/urandom).

Locations:

- `action.yml:112`

### github-env-injection (severity: high)

Step 'Pwsh Run Slack CLI command' writes $output to $env:GITHUB_OUTPUT between a fixed heredoc delimiter: "response<<SLACKCLIEOF" >> $env:GITHUB_OUTPUT; $output >> $env:GITHUB_OUTPUT; "SLACKCLIEOF". The value comes from a command run with untrusted input. Fix: use a delimiter built from [guid]::NewGuid().

Locations:

- `action.yml:144`

### github-env-injection (severity: high)

Step 'Bash Add Slack CLI to PATH' writes the inherited process env var $HOME to $GITHUB_PATH without the sanitization step: echo "$HOME/.slack/bin" >> "$GITHUB_PATH". HOME is not set inside this script, and a calling workflow can override it. Fix: apply printf '%s' ... | tr -d '\n\r' before the write.

Locations:

- `action.yml:49`

### github-env-injection (severity: high)

Step 'Pwsh Add Slack CLI to PATH' writes the inherited env var $env:USERPROFILE to $env:GITHUB_PATH without stripping newlines: Add-Content -Path $env:GITHUB_PATH -Value "$env:USERPROFILE\AppData\Local\slack-cli\bin". The calling workflow controls this value. Fix: remove CR/LF characters from it before writing.

Locations:

- `action.yml:54`

### unsafe-shell (severity: high)

Step 'Bash Install Slack CLI' pipes remote content straight into bash: curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION". Fix: download the script to a file first, then run it.

Locations:

- `action.yml:64`

### unsafe-shell (severity: high)

Step 'Bash Install Slack CLI' pipes remote content straight into bash: curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s. Fix: download the script to a file first, then run it.

Locations:

- `action.yml:66`

### unsafe-shell (severity: high)

Step 'Pwsh Install Slack CLI' pipes remote content straight into the PowerShell interpreter: irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex. Fix: download it to a file with Invoke-WebRequest -OutFile and run it, as the other branch of this step already does.

Locations:

- `action.yml:83`

### script-injection (severity: high)

Sub-rule (b): step 'Bash Run Slack CLI command' expands a variable built from untrusted inputs without quotes: output=$(slack $args 2>&1 | tee /dev/stderr). $args is made from $SLACK_COMMAND (inputs.command) and $SLACK_TOKEN (inputs.token). Because it is unquoted, the shell word-splits it and expands globs on it. Fix: build the arguments as a bash array and expand it quoted.

Locations:

- `action.yml:105`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, unsafe-shell, script-injection

**Notes:**

All 8 findings are fixed in action.yml. Nothing was run or tested.

- **Fixed heredoc delimiters (bash and pwsh):** the delimiter for the `response` output is now random on every run. Bash builds it from /dev/urandom (via od), PowerShell from `[guid]::NewGuid()`. The output keeps its line breaks.
- **PATH writes (bash and pwsh):** newline and carriage-return characters are now removed from `$HOME/.slack/bin` (bash, using `tr -d '\n\r'`) and from `$env:USERPROFILE\AppData\Local\slack-cli\bin` (pwsh, using `-replace`) before they are written to `GITHUB_PATH`.
- **Installers piped into a shell:** both scripts are now downloaded to a file and then run. The bash installer goes to a mktemp file run with `bash "$installer" -v "$SLACK_CLI_VERSION"`; I dropped `-s --` so the script still gets `-v` as its first argument. The Windows branch with no version now downloads with Invoke-WebRequest and runs the file, like the version branch already did.
- **Unquoted `slack $args`:** the command is now built as a bash array and passed as `"${args[@]}"`. The command and token are split on whitespace (`read -r -d '' -a ... || true`), the same way upstream's unquoted `$args` was. Quotes in the input stay as plain text, the token keeps its own argument after `--token`, and `--skip-update` and `--verbose` are added the same way as before. Unlike upstream, the expansion no longer expands globs.

I left the pwsh command step's `.Split(' ')` unchanged: it has no finding and already splits on spaces.

### Iteration 2

**Fixes applied:** unsafe-shell, github-env-injection

**Notes:**

unsafe-shell: the Bash install step now downloads install.sh to a temp file with mktemp + curl -o and checks the file isn't empty. It then runs `bash "$installer" -v "$SLACK_CLI_VERSION"` (or `bash "$installer"` when no version is given), with the pipe's '-s --' dropped. github-env-injection: (1) the $HOME/.slack/bin value is passed through printf | tr -d '\n\r' before it is written to $GITHUB_PATH; (2) removed the fixed SLACKCLIEOF heredoc line and the stray 'e'/'fi' text, so only the delimiter drawn from /dev/urandom is used. I also put back the 'exit $exit_code' block that the stray 'fi' had come from. Separately, the Pwsh install step had been broken by the last pass (an unclosed if block, a stray 'inputs.version }}' and no env block). I rebuilt it to download install-windows.ps1 to a file and run it, using an env SLACK_CLI_VERSION mapping.

