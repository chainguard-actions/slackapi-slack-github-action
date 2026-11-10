<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.5** was hardened automatically. 7 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

The 'Bash Run Slack CLI command' step writes the CLI output to $GITHUB_OUTPUT between fixed heredoc delimiters: `echo "response<<SLACKCLIEOF"` / `echo "$output"` / `echo "SLACKCLIEOF"`. $output comes from running a command built from inputs.command and inputs.token. If the output contains a line that reads exactly SLACKCLIEOF, the block ends early and any lines after it are read as new output entries. Fix: build the delimiter from a secure random source on every run (e.g. /dev/urandom or openssl rand).

Locations:

- `action.yml:112`

### github-env-injection (severity: high)

The 'Pwsh Run Slack CLI command' step writes $output (CLI output from a command built from inputs.command) to $env:GITHUB_OUTPUT between the fixed delimiter SLACKCLIEOF: `"response<<SLACKCLIEOF" >> $env:GITHUB_OUTPUT`. Output containing that line can end the block early and add extra output entries. Fix: use a delimiter generated with [guid]::NewGuid() on every run.

Locations:

- `action.yml:144`

### script-injection (severity: high)

Rule (b): in the 'Bash Run Slack CLI command' step, `output=$(slack $args 2>&1 | tee /dev/stderr)` expands $args without quotes. $args is built from the workflow-controlled SLACK_COMMAND (inputs.command) and SLACK_TOKEN (inputs.token), so the shell splits these values on whitespace and expands glob characters in them. This is not the exempt array-assignment form with a shellcheck directive.

Locations:

- `action.yml:105`

### unsafe-shell (severity: high)

Remote script piped straight into a shell: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"`. Download the script to a file first, then run it separately.

Locations:

- `action.yml:64`

### unsafe-shell (severity: high)

Remote script piped straight into a shell: `curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s`. Download the script to a file first, then run it separately.

Locations:

- `action.yml:66`

### unsafe-shell (severity: high)

Remote script piped straight into the PowerShell interpreter: `irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex`. Download it with Invoke-WebRequest -OutFile and then run it, as the versioned branch of the same step already does.

Locations:

- `action.yml:83`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/cli/action.yml b/cli/action.yml
index 5fd8249..358b8e0 100644
--- a/cli/action.yml
+++ b/cli/action.yml
@@ -60,11 +60,15 @@ runs:
         runner.os != 'Windows'
       shell: bash
       run: |
+        installer=$(mktemp /tmp/install-slack-cli.XXXXXX.sh)
+        curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh -o "$installer"
+        chmod +x "$installer"
         if [ -n "$SLACK_CLI_VERSION" ]; then
-          curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION"
+          bash "$installer" -v "$SLACK_CLI_VERSION"
         else
-          curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s
+          bash "$installer"
         fi
+        rm -f "$installer"
       env:
         SLACK_CLI_VERSION: ${{ inputs.version }}
 
@@ -75,13 +79,14 @@ runs:
         runner.os == 'Windows'
       shell: pwsh
       run: |
+        $installer = Join-Path $env:TEMP "install-slack-cli.ps1"
+        Invoke-WebRequest -Uri "https://downloads.slack-edge.com/slack-cli/install-windows.ps1" -OutFile $installer
         if ($env:SLACK_CLI_VERSION) {
-          $installer = Join-Path $env:TEMP "install-slack-cli.ps1"
-          Invoke-WebRequest -Uri "https://downloads.slack-edge.com/slack-cli/install-windows.ps1" -OutFile $installer
           & $installer -v $env:SLACK_CLI_VERSION
         } else {
-          irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex
+          & $installer
         }
+        Remove-Item -Path $installer -Force
       env:
         SLACK_CLI_VERSION: ${{ inputs.version }}
 
@@ -93,16 +98,18 @@ runs:
         SLACK_COMMAND: ${{ inputs.command }}
         SLACK_TOKEN: ${{ inputs.token }}
       run: |
-        args="$SLACK_COMMAND --skip-update"
+        # Split the command string into an array to avoid unquoted expansion
+        read -ra cmd_args <<< "$SLACK_COMMAND"
+        args=("${cmd_args[@]}" "--skip-update")
         if [ "$RUNNER_DEBUG" = "1" ]; then
-          args="$args --verbose"
+          args+=("--verbose")
         fi
         if [ -n "$SLACK_TOKEN" ]; then
-          args="$args --token $SLACK_TOKEN"
+          args+=("--token" "$SLACK_TOKEN")
         fi
 
         set +e
-        output=$(slack $args 2>&1 | tee /dev/stderr)
+        output=$(slack "${args[@]}" 2>&1 | tee /dev/stderr)
         exit_code=$?
         set -e
 
@@ -125,16 +132,19 @@ runs:
         SLACK_COMMAND: ${{ inputs.command }}
         SLACK_TOKEN: ${{ inputs.token }}
       run: |
-        $cliArgs = "$env:SLACK_COMMAND --skip-update"
+        # Build args as an array to avoid unquoted string splitting
+        $cliArgs = $env:SLACK_COMMAND -split '\s+' | Where-Object { $_ -ne '' }
+        $cliArgs += '--skip-update'
         if ($env:RUNNER_DEBUG -eq "1") {
-          $cliArgs = "$cliArgs --verbose"
+          $cliArgs += '--verbose'
         }
         if ($env:SLACK_TOKEN) {
-          $cliArgs = "$cliArgs --token $env:SLACK_TOKEN"
+          $cliArgs += '--token'
+          $cliArgs += $env:SLACK_TOKEN
         }
 
         $outputFile = New-TemporaryFile
-        slack $cliArgs.Split(' ') 2>&1 | Tee-Object -FilePath $outputFile.FullName
+        slack @cliArgs 2>&1 | Tee-Object -FilePath $outputFile.FullName
         $exit_code = $LASTEXITCODE
         $output = Get-Content $outputFile.FullName -Raw
 

```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, script-injection, unsafe-shell, prior-fix

**Notes:**

Bash install step: the installer is downloaded to a temp file with mktemp + curl -o, then run with 'bash "$installer" -v "$SLACK_CLI_VERSION"' or with no arguments ('-s --' dropped), then deleted. Pwsh install step: the script is always downloaded with Invoke-WebRequest -OutFile and run with '& $installer', replacing 'irm | iex'. Bash run step: SLACK_COMMAND is split with the unquoted array form 'args=($SLACK_COMMAND)' plus a shellcheck disable comment, so splitting and globbing match upstream. The previous harden's 'read -ra' was not used because it does not glob. The token is added as a separate quoted argument, and the command runs as slack "${args[@]}". Pwsh run step: arguments are an array split on whitespace with -split '\s+' and passed with @cliArgs. The response output's heredoc delimiter is now random on each run: od from /dev/urandom in bash, [guid]::NewGuid() in pwsh. The output value is written unchanged.

### Iteration 2

**Fixes applied:** github-env-injection

**Notes:**

Bash step: the $HOME/.slack/bin value now goes through tr -d '\n\r' before it is appended to $GITHUB_PATH. Pwsh step: the USERPROFILE-based path now has [\r\n] removed with -replace before Add-Content. Removed the zizmor ignore comments.

