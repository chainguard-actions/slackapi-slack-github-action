<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.1

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.1** was hardened automatically. 6 finding(s) were identified and resolved across 4 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

Step 'Bash Run Slack CLI command' writes $output, the output of a command built from inputs.command and inputs.token, to $GITHUB_OUTPUT between a fixed heredoc delimiter: echo "response<<SLACKCLIEOF" / echo "$output" / echo "SLACKCLIEOF". If the output contains a line that reads 'SLACKCLIEOF', the block ends early and every line after it is added as a new output entry. The delimiter should be generated from a secure random source such as /dev/urandom.

Locations:

- `action.yml:112`

### github-env-injection (severity: high)

Step 'Pwsh Run Slack CLI command' writes $output, the CLI output from a command built from inputs.command and inputs.token, to $env:GITHUB_OUTPUT between a fixed heredoc delimiter: "response<<SLACKCLIEOF" / $output / "SLACKCLIEOF". A line equal to the delimiter ends the block early, so the value can add its own output entries. The delimiter should be generated with [guid]::NewGuid().

Locations:

- `action.yml:144`

### unsafe-shell (severity: high)

Step 'Bash Install Slack CLI' downloads a remote script and pipes it straight into a shell: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"`.

Locations:

- `action.yml:64`

### unsafe-shell (severity: high)

Step 'Bash Install Slack CLI' downloads a remote script and pipes it straight into a shell: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`.

Locations:

- `action.yml:66`

### unsafe-shell (severity: high)

Step 'Pwsh Install Slack CLI' downloads a remote script and runs it directly with PowerShell (the same pattern as curl | sh): `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. The script should be saved to a file first and then run, as the other branch of this step already does.

Locations:

- `action.yml:83`

### script-injection (severity: high)

Sub-rule (b): step 'Bash Run Slack CLI command' expands the shell variable `args` without quotes. `args` is built from SLACK_COMMAND (inputs.command) and SLACK_TOKEN (inputs.token). Offending line: `output=$(slack $args 2>&1 | tee /dev/stderr)`. Because it is unquoted, the shell splits the value into words and expands glob characters, so input values can inject extra arguments. Build the arguments as a bash array and expand it as "${args[@]}".

Locations:

- `action.yml:105`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, unsafe-shell, script-injection

**Notes:**

unsafe-shell: Both bash installs now download install.sh to a mktemp file and run it with 'bash "$installer"' ('-v VERSION' only when a version is given). The '-s --' left over from the pipe form was dropped. The pwsh install always saves the installer with Invoke-WebRequest and runs it, so the 'irm | iex' branch is gone. script-injection: The bash args are now an array. SLACK_COMMAND is split on whitespace with 'read -r -d "" -a ... || true', inside a check that it is not empty. This keeps upstream's unquoted $VAR behaviour: quotes stay as literal text and it does not use xargs. Then --skip-update, --verbose (debug only) and --token "$SLACK_TOKEN" are appended, and the command runs as "${args[@]}". The token is now passed as one argument, where upstream's unquoted $args split it on any whitespace. github-env-injection: The fixed SLACKCLIEOF delimiters are replaced with random ones (od from /dev/urandom in bash, [guid]::NewGuid() in pwsh). The multi-line output is written unchanged.

### Iteration 2

**Fixes applied:** github-env-injection, unsafe-shell

**Notes:**

Pwsh install step: the 'irm ... | iex' line is gone. The installer is now always saved to a temp file with Invoke-WebRequest -OutFile, then run with '& $installer' (adding -v when a version is given), then deleted. Pwsh run step: the fixed SLACKCLIEOF delimiter is replaced by one made fresh on each run from [guid]::NewGuid(), the same way the bash step already does it. The multi-line response value is written unchanged.

### Iteration 3

**Fixes applied:** unsafe-shell, github-env-injection

**Notes:**

I changed the 'Pwsh Install Slack CLI' step so it no longer pipes 'irm ... | iex'. It now always saves install-windows.ps1 to $env:TEMP with Invoke-WebRequest -OutFile and runs that file. It passes '-v $env:SLACK_CLI_VERSION' when a version is set and no arguments otherwise, then deletes the file. I removed the newline from the 'Bash Add Slack CLI to PATH' value with printf '%s' | tr -d '\n\r' before writing it to GITHUB_PATH. In the 'Pwsh Add Slack CLI to PATH' step, -replace "[`r`n]" now removes newlines from the USERPROFILE-based path before Add-Content writes it. I also dropped the old zizmor ignore comments. Nothing has been run or tested.

One thing to check: when no version is set, the installer now runs as a .ps1 file instead of through iex, so PowerShell's script execution policy now applies to that path. The version branch already ran the script this way, so it should work wherever that branch did.

### Iteration 4

**Fixes applied:** unsafe-shell

**Notes:**

In the 'Pwsh Install Slack CLI' step, replaced `irm ... | iex` with Invoke-WebRequest -OutFile to a temp file, then executed via `& $installer` (with `-v` when a version is given) and removed the file afterwards.

