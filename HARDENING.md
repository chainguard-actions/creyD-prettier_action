<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **creyD--prettier_action/v4.5** was hardened automatically. 2 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A ${{ }} expression is directly interpolated inside a run: shell command string in action.yml. The line `${{ github.action_path }}/entrypoint.sh` embeds the github.action_path context value directly into the shell command before the shell ever sees it. Even though github.action_path is not attacker-controlled, any ${{ ... }} expression directly inside a run: block is a script-injection risk per the check rules, as YAML template substitution occurs before shell quoting.

Locations:

- `action.yml:70`

### script-injection (severity: high)

Sub-rule (b): Multiple unquoted shell variable expansions of env vars holding workflow-controllable (untrusted) input values in entrypoint.sh:
1. Line 43: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — $INPUT_PRETTIER_VERSION (from inputs.prettier_version) is unquoted, allowing shell metacharacter injection.
2. Line 55: `npm install --silent $INPUT_PRETTIER_PLUGINS` — $INPUT_PRETTIER_PLUGINS (from inputs.prettier_plugins) is unquoted, allowing word-splitting and metacharacter injection.
3. Line 62: `prettier $INPUT_PRETTIER_OPTIONS` — $INPUT_PRETTIER_OPTIONS (from inputs.prettier_options) is unquoted, allowing shell metacharacter injection. All three variables must be double-quoted (e.g., "$INPUT_PRETTIER_VERSION") to prevent command injection.

Locations:

- `entrypoint.sh:43`
- `entrypoint.sh:55`
- `entrypoint.sh:62`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:
1. action.yml line 70: Replaced `${{ github.action_path }}/entrypoint.sh` with `"$GITHUB_ACTION_PATH/entrypoint.sh"` — uses the equivalent GitHub Actions environment variable instead of a ${{ }} template expression in the run: block.
2. entrypoint.sh line 43: Quoted `prettier@$INPUT_PRETTIER_VERSION` as `"prettier@$INPUT_PRETTIER_VERSION"` to prevent shell metacharacter injection.
3. entrypoint.sh line 55: Used `read -ra` array splitting to safely pass `$INPUT_PRETTIER_PLUGINS` as separate arguments to npm install, preventing word-splitting and metacharacter injection.
4. entrypoint.sh line 62: Used `read -ra` array splitting to safely pass `$INPUT_PRETTIER_OPTIONS` as separate arguments to prettier, preventing word-splitting and metacharacter injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all 7 script injection vulnerabilities in entrypoint.sh:
1. Line 49: Replaced unquoted `for plugin in $INPUT_PRETTIER_PLUGINS` with `read -ra _PLUGINS_CHECK_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"` and `for plugin in "${_PLUGINS_CHECK_ARRAY[@]}"`
2. Line 65: Replaced `if $INPUT_CLEAN_NODE_FOLDER` with `if [ "$INPUT_CLEAN_NODE_FOLDER" = "true" ]`
3. Line 79: Added double-quotes around `$INPUT_ONLY_CHANGED` and `$INPUT_ONLY_CHANGED_PR` in `[ ]` tests
4. Line 80: Replaced `if $INPUT_ONLY_CHANGED` with `if [ "$INPUT_ONLY_CHANGED" = "true" ]`
5. Line 100: Replaced `if $INPUT_DRY` with `if [ "$INPUT_DRY" = "true" ]`
6. Lines 104 & 113: Replaced both `if $INPUT_NO_COMMIT` with `if [ "$INPUT_NO_COMMIT" = "true" ]`
7. Line 113: Replaced `if $INPUT_SAME_COMMIT` with `if [ "$INPUT_SAME_COMMIT" = "true" ]`
8. Line 131: Replaced `git push origin ${INPUT_PUSH_OPTIONS:-}` with a bash array pattern: `push_args=(); [ -n "$INPUT_PUSH_OPTIONS" ] && push_args+=($INPUT_PUSH_OPTIONS); git push origin "${push_args[@]+"${push_args[@]}"}"` to safely handle optional push options

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed unquoted expansion of $INPUT_PUSH_OPTIONS in entrypoint.sh line 140. Replaced `push_args+=($INPUT_PUSH_OPTIONS)` with `read -ra push_args <<< "$INPUT_PUSH_OPTIONS"` to prevent word-splitting and glob expansion on attacker-controlled input while still correctly splitting multi-word push options into separate array elements. This matches the pattern already used elsewhere in the script for $INPUT_PRETTIER_OPTIONS and $INPUT_PRETTIER_PLUGINS.

