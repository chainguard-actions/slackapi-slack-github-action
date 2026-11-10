<!-- markdownlint-disable -->

# Hardening Report: slackapi--slack-github-action--cli/v3.0.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **slackapi--slack-github-action--cli/v3.0.4** was hardened automatically. 9 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### github-env-injection (severity: high)

The 'Bash Run Slack CLI command' step writes $output to $GITHUB_OUTPUT between a fixed heredoc delimiter: echo "response<<SLACKCLIEOF" / echo "$output" / echo "SLACKCLIEOF". $output is the output of the CLI command, which is built from inputs.command and inputs.token. If that output contains a line that says exactly SLACKCLIEOF, the block ends early and every line after it becomes a new output entry. Fix: generate the delimiter from /dev/urandom (or openssl rand) on every run.

Locations:

- `action.yml:112`

### github-env-injection (severity: high)

The 'Pwsh Run Slack CLI command' step writes $output to $env:GITHUB_OUTPUT between a fixed heredoc delimiter: "response<<SLACKCLIEOF" / $output / "SLACKCLIEOF". The CLI output is derived from inputs.command, so a line equal to the delimiter can end the block and inject extra outputs. Fix: use a random delimiter such as [guid]::NewGuid().

Locations:

- `action.yml:144`

### github-env-injection (severity: high)

The 'Bash Add Slack CLI to PATH' step writes the inherited env var $HOME to $GITHUB_PATH with no sanitising step: echo "$HOME/.slack/bin" >> "$GITHUB_PATH". In a composite action, a calling workflow can set HOME, and the check treats any inherited env var as untrusted. Fix: pass the value through printf '%s' | tr -d '\n\r' before writing it.

Locations:

- `action.yml:49`

### github-env-injection (severity: high)

The 'Pwsh Add Slack CLI to PATH' step writes the inherited env var $env:USERPROFILE to $env:GITHUB_PATH without removing newlines: Add-Content -Path $env:GITHUB_PATH -Value "$env:USERPROFILE\AppData\Local\slack-cli\bin". A calling workflow can control this value.

Locations:

- `action.yml:54`

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote script straight into bash: curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s -- -v "$SLACK_CLI_VERSION".

Locations:

- `action.yml:64`

### unsafe-shell (severity: high)

The 'Bash Install Slack CLI' step pipes a remote script straight into bash: curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash -s.

Locations:

- `action.yml:66`

### unsafe-shell (severity: high)

The 'Pwsh Install Slack CLI' step downloads a remote script and runs it straight through Invoke-Expression: irm https://downloads.slack-edge.com/slack-cli/install-windows.ps1 | iex. Download it to a file first, as the version-pinned branch above it already does.

Locations:

- `action.yml:83`

### script-injection (severity: high)

Rule (b): the 'Bash Run Slack CLI command' step expands workflow-controlled data unquoted. $args is built from $SLACK_COMMAND (inputs.command) and $SLACK_TOKEN (inputs.token), then expanded unquoted in: output=$(slack $args 2>&1 | tee /dev/stderr). This is not the array-assignment exception: there is no array and no shellcheck directive. Fix: build a bash array (e.g. read -ra for the command, then append the token as one quoted element) and expand it as "${arr[@]}".

Locations:

- `action.yml:105`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/cli/action.yml b/cli/action.yml
index 5fd8249..49acd74 100644
--- a/cli/action.yml
+++ b/cli/action.yml
@@ -60,11 +60,14 @@ runs:
         runner.os != 'Windows'
       shell: bash
       run: |
+        installer=$(mktemp)
+        curl -fsSL -o "$installer" https://downloads.slack-edge.com/slack-cli/install.sh
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
 
@@ -75,13 +78,14 @@ runs:
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
 
@@ -93,24 +97,28 @@ runs:
         SLACK_COMMAND: ${{ inputs.command }}
         SLACK_TOKEN: ${{ inputs.token }}
       run: |
-        args="$SLACK_COMMAND --skip-update"
+        # Build args as an array to prevent word-splitting and shell injection
+        read -ra args <<< "$SLACK_COMMAND"
+        args+=("--skip-update")
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
 
-        echo "ok=$([ $exit_code -eq 0 ] && echo 'true' || echo 'false')" >> "$GITHUB_OUTPUT"
-        echo "time=$(date +%s)" >> "$GITHUB_OUTPUT"
+        ok_val=$([ $exit_code -eq 0 ] && echo 'true' || echo 'false')
+        time_val=$(date +%s)
+        printf 'ok=%s\n' "$ok_val" >> "$GITHUB_OUTPUT"
+        printf 'time=%s\n' "$time_val" >> "$GITHUB_OUTPUT"
 
         echo "response<<SLACKCLIEOF" >> "$GITHUB_OUTPUT"
-        echo "$output" >> "$GITHUB_OUTPUT"
+        printf '%s\n' "$output" >> "$GITHUB_OUTPUT"
         echo "SLACKCLIEOF" >> "$GITHUB_OUTPUT"
 
         if [ $exit_code -ne 0 ]; then

```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** github-env-injection, unsafe-shell, script-injection, prior-fix

**Notes:**

Bash/pwsh response outputs now use a random heredoc delimiter (from /dev/urandom in bash, a GUID in pwsh), and the multi-line value is kept as is. Newlines are stripped from HOME and USERPROFILE before they are written to GITHUB_PATH. Both installers are downloaded to a temporary file before they run, and the pipe-only '-s --' was dropped. In the bash run step, the command input becomes an array through an unquoted assignment, which splits and globs the same way upstream's unquoted $args did. The token is added as its own quoted argument, and the array is expanded as "${args[@]}". The prior harden's install changes are re-applied. Its 'read -ra' split is deliberately replaced by the unquoted-array form so globbing stays the same as upstream.

