<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **creyD--prettier_action/v4.2** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A GitHub Actions expression `${{ github.action_path }}` is directly interpolated inside the `run:` shell command string in action.yml. The YAML template substitution occurs before the shell processes the string, meaning any attacker-controlled value in `github.action_path` would be executed as shell code. The offending line is: `${{ github.action_path }}/entrypoint.sh`

Locations:

- `action.yml:57`

### script-injection (severity: high)

Sub-rule (b): Multiple unquoted shell variable expansions of user-controlled inputs in entrypoint.sh. The env vars INPUT_WORKING_DIRECTORY, INPUT_PRETTIER_VERSION, INPUT_PRETTIER_OPTIONS, and INPUT_PRETTIER_PLUGINS are all sourced from `inputs.*` (via the env: block in action.yml) but are expanded unquoted in shell commands, allowing shell metacharacter injection (`;`, `|`, `&`, `$(...)`, etc.):
- `case $INPUT_WORKING_DIRECTORY in` (line 37) — unquoted case pattern
- `cd $INPUT_WORKING_DIRECTORY` (line 41) — unquoted cd argument
- `case $INPUT_PRETTIER_VERSION in` (line 43) — unquoted case pattern
- `npm install --silent prettier@$INPUT_PRETTIER_VERSION` (line 48) — unquoted in npm argument
- `npm install --silent $INPUT_PRETTIER_PLUGINS` (line 55) — unquoted npm argument
- `prettier $INPUT_PRETTIER_OPTIONS` (line 62) — unquoted prettier argument

Locations:

- `entrypoint.sh:37`
- `entrypoint.sh:41`
- `entrypoint.sh:43`
- `entrypoint.sh:48`
- `entrypoint.sh:55`
- `entrypoint.sh:62`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:

1. action.yml (line 57): Replaced `${{ github.action_path }}/entrypoint.sh` with `"$GITHUB_ACTION_PATH/entrypoint.sh"`. The GITHUB_ACTION_PATH env var holds the same value but avoids direct YAML template interpolation into the shell command string. Also quoted the variable in the `cd` command.

2. entrypoint.sh (multiple lines): Quoted all unquoted variable expansions:
   - `case $INPUT_WORKING_DIRECTORY` → `case "$INPUT_WORKING_DIRECTORY"`
   - `cd $INPUT_WORKING_DIRECTORY` → `cd "$INPUT_WORKING_DIRECTORY"`
   - `case $INPUT_PRETTIER_VERSION` → `case "$INPUT_PRETTIER_VERSION"`
   - `npm install --silent prettier@$INPUT_PRETTIER_VERSION` → `npm install --silent "prettier@$INPUT_PRETTIER_VERSION"`
   - `prettier $INPUT_PRETTIER_OPTIONS` → split into array with `read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"` then `prettier "${PRETTIER_OPTIONS_ARRAY[@]}"` to safely handle word-splitting while preventing shell metacharacter injection
   - The `npm install --silent $INPUT_PRETTIER_PLUGINS` line was left with intentional word-splitting since plugins are a space-separated list and each is validated by regex before installation.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all 6 script injection vulnerabilities in entrypoint.sh: (1) for loop over $INPUT_PRETTIER_PLUGINS now uses read -ra array split and quoted array expansion; (2) npm install now uses quoted array "${PRETTIER_PLUGINS_ARRAY[@]}"; (3) `if $INPUT_DRY` replaced with `if [ "$INPUT_DRY" = "true" ]`; (4) `if $INPUT_ONLY_CHANGED` replaced with `if [ "$INPUT_ONLY_CHANGED" = "true" ]`; (5) `if $INPUT_SAME_COMMIT` replaced with `if [ "$INPUT_SAME_COMMIT" = "true" ]`; (6) `git push origin ${INPUT_PUSH_OPTIONS:-}` replaced with conditional read -ra array split and quoted expansion to prevent word-splitting and flag injection.

