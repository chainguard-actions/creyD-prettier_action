<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.2

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **creyD--prettier_action/v4.2** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): The `run:` block in action.yml directly interpolates `${{ github.action_path }}` inside the shell command string. Any `${{ ... }}` expression directly in a `run:` script is a script-injection risk because YAML template substitution occurs before the shell ever sees the value. Offending line: `${{ github.action_path }}/entrypoint.sh`

Locations:

- `action.yml:59`

### script-injection (severity: high)

Sub-rule (b): entrypoint.sh expands multiple workflow-controlled `$INPUT_*` environment variables (sourced from `inputs.*` via the action's `env:` block) without double-quoting, allowing shell metacharacter injection. Unquoted expansions include: `case $INPUT_WORKING_DIRECTORY in` (line 36), `cd $INPUT_WORKING_DIRECTORY` (line 39), `case $INPUT_PRETTIER_VERSION in` (line 42), `npm install --silent prettier@$INPUT_PRETTIER_VERSION` (line 46), `for plugin in $INPUT_PRETTIER_PLUGINS` (line 50), `npm install --silent $INPUT_PRETTIER_PLUGINS` (line 56), `prettier $INPUT_PRETTIER_OPTIONS` (line 61), `if $INPUT_DRY` (line 70), `if $INPUT_ONLY_CHANGED` (line 80), `if $INPUT_SAME_COMMIT` (line 91), and `git push origin ${INPUT_PUSH_OPTIONS:-}` (line 103). An attacker-controlled input value containing shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) can execute arbitrary commands.

Locations:

- `entrypoint.sh:36`
- `entrypoint.sh:39`
- `entrypoint.sh:42`
- `entrypoint.sh:46`
- `entrypoint.sh:50`
- `entrypoint.sh:56`
- `entrypoint.sh:61`
- `entrypoint.sh:70`
- `entrypoint.sh:80`
- `entrypoint.sh:91`
- `entrypoint.sh:103`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:

1. action.yml (line 59): Replaced `${{ github.action_path }}/entrypoint.sh` with `"$GITHUB_ACTION_PATH/entrypoint.sh"` — uses the pre-set GitHub Actions environment variable instead of YAML template interpolation in the run block. Also quoted the `cd "$GITHUB_ACTION_PATH"` call.

2. entrypoint.sh (11 locations): Fixed all unquoted $INPUT_* expansions:
   - Quoted `case "$INPUT_WORKING_DIRECTORY"` and `cd "$INPUT_WORKING_DIRECTORY"`
   - Quoted `case "$INPUT_PRETTIER_VERSION"` and `"prettier@$INPUT_PRETTIER_VERSION"`
   - Plugin loop: collected validated plugins into an array, then used `"${validated_plugins[@]}"` for npm install
   - Prettier options: used `read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"` and `"${PRETTIER_OPTIONS_ARRAY[@]}"`
   - Boolean inputs (INPUT_DRY, INPUT_ONLY_CHANGED, INPUT_SAME_COMMIT): changed from `if $VAR` (executes value as command) to `if [ "$VAR" = "true" ]` (safe string comparison)
   - Push options: used `read -ra push_opts <<< "$INPUT_PUSH_OPTIONS"` and `"${push_opts[@]}"`

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted shell variable expansion in entrypoint.sh at line 49. Replaced `for plugin in $INPUT_PRETTIER_PLUGINS` (vulnerable to word-splitting and glob expansion on attacker-controlled input) with `read -ra plugins_array <<< "$INPUT_PRETTIER_PLUGINS"` followed by `for plugin in "${plugins_array[@]}"`. This safely splits the input into an array and iterates over elements with proper double-quoting, preventing shell metacharacter injection. The approach is consistent with how INPUT_PRETTIER_OPTIONS is already handled later in the same script.

