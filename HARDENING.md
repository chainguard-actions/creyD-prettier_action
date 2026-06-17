<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **creyD--prettier_action/v4.6** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): The run: block in action.yml directly interpolates `${{ github.action_path }}` inside the shell command string. Any ${{ ... }} expression interpolated directly into a run: block is a script-injection risk because YAML template substitution occurs before the shell ever sees the value. The offending line is: `${{ github.action_path }}/entrypoint.sh`

Locations:

- `action.yml:62`

### script-injection (severity: high)

Rule (b): entrypoint.sh expands user-controlled input env vars without double-quoting in shell commands, allowing shell metacharacter injection. Three unquoted expansions: (1) `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — INPUT_PRETTIER_VERSION is set from inputs.prettier_version; (2) `npm install --silent $INPUT_PRETTIER_PLUGINS` — INPUT_PRETTIER_PLUGINS is set from inputs.prettier_plugins; (3) `npx prettier $INPUT_PRETTIER_OPTIONS` — INPUT_PRETTIER_OPTIONS is set from inputs.prettier_options. All three allow an attacker to inject arbitrary shell commands via the respective input.

Locations:

- `entrypoint.sh:43`
- `entrypoint.sh:53`
- `entrypoint.sh:60`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:
1. action.yml: Replaced `${{ github.action_path }}/entrypoint.sh` in the run: block with `"$GITHUB_ACTION_PATH/entrypoint.sh"`, eliminating YAML template substitution in the shell command by using the equivalent shell environment variable.
2. entrypoint.sh line 43: Quoted `"prettier@$INPUT_PRETTIER_VERSION"` to prevent shell metacharacter injection.
3. entrypoint.sh line 53: Replaced unquoted `$INPUT_PRETTIER_PLUGINS` with a safe array expansion using `read -ra PRETTIER_PLUGINS_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"` and `"${PRETTIER_PLUGINS_ARRAY[@]}"`, preserving multi-plugin support while preventing injection.
4. entrypoint.sh line 60: Replaced unquoted `$INPUT_PRETTIER_OPTIONS` with a safe array expansion using `read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"` and `"${PRETTIER_OPTIONS_ARRAY[@]}"`, preserving multi-option support while preventing injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all 9 script injection vulnerabilities in entrypoint.sh:
1. Line 53: Replaced unquoted `for plugin in $INPUT_PRETTIER_PLUGINS` with `read -ra PRETTIER_PLUGINS_CHECK_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"` and `for plugin in "${PRETTIER_PLUGINS_CHECK_ARRAY[@]}"`.
2. Line 78: Replaced `if $INPUT_CLEAN_NODE_FOLDER` with `if [ "$INPUT_CLEAN_NODE_FOLDER" = "true" ]`.
3. Lines 94/96: Replaced `[ $INPUT_ONLY_CHANGED = true ] || [ $INPUT_ONLY_CHANGED_PR = true ]` and `if $INPUT_ONLY_CHANGED` with properly quoted string comparisons.
4. Line 112: Replaced `if $INPUT_DRY` with `if [ "$INPUT_DRY" = "true" ]`.
5. Lines 115/127: Replaced both `if $INPUT_NO_COMMIT` with `if [ "$INPUT_NO_COMMIT" = "true" ]`.
6. Line 132: Replaced `if $INPUT_SAME_COMMIT` with `if [ "$INPUT_SAME_COMMIT" = "true" ]`.
7. Line 143: Replaced `git push origin ${INPUT_PUSH_OPTIONS:-}` with a safe array-based approach using `read -ra PUSH_OPTIONS_ARRAY` and `git push origin "${PUSH_OPTIONS_ARRAY[@]}"`.

