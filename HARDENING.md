<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **creyD--prettier_action/v4.4** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): A ${{ ... }} expression is interpolated directly inside a `run:` shell command string in action.yml. The expression `${{ github.action_path }}/entrypoint.sh` is expanded by the Actions template engine before the shell ever sees it, allowing any value injected into that context to be executed as a shell command. Even though `github.action_path` is not directly attacker-controlled, any `${{ ... }}` inside a `run:` block is a script-injection violation per the check rules.

Locations:

- `action.yml:67`

### script-injection (severity: high)

Sub-rule (b): Multiple unquoted shell variable expansions of user-controlled INPUT_* env vars in entrypoint.sh. These variables are sourced directly from action inputs (inputs.*) and are set in the calling workflow's env: block, making them workflow-controllable. Unquoted expansions allow shell metacharacters (`;`, `|`, `&`, `$(...)`, whitespace, glob chars) in the values to be interpreted by the shell, enabling command injection. Offending lines include:
- Line 44: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` (unquoted version string)
- Line 47: `for plugin in $INPUT_PRETTIER_PLUGINS` (unquoted, enables word-splitting of attacker-controlled value)
- Line 54: `npm install --silent $INPUT_PRETTIER_PLUGINS` (unquoted plugins list)
- Line 59: `prettier $INPUT_PRETTIER_OPTIONS` (unquoted options, allows arbitrary flag/argument injection)
- Line 63: `if $INPUT_CLEAN_NODE_FOLDER` (unquoted boolean used as command)
- Line 77: `if [ $INPUT_ONLY_CHANGED = true ]` (unquoted in test expression)
- Line 80: `if $INPUT_ONLY_CHANGED` (unquoted boolean used as command)
- Line 96: `if $INPUT_DRY` (unquoted boolean used as command)
- Line 99: `if $INPUT_NO_COMMIT` (unquoted boolean used as command)
- Line 106: `if $INPUT_NO_COMMIT` (unquoted boolean used as command)
- Line 110: `if $INPUT_SAME_COMMIT` (unquoted boolean used as command)
- Line 118: `git push origin ${INPUT_PUSH_OPTIONS:-}` (unquoted push options)

Locations:

- `entrypoint.sh:44`
- `entrypoint.sh:47`
- `entrypoint.sh:54`
- `entrypoint.sh:59`
- `entrypoint.sh:63`
- `entrypoint.sh:77`
- `entrypoint.sh:80`
- `entrypoint.sh:96`
- `entrypoint.sh:99`
- `entrypoint.sh:106`
- `entrypoint.sh:110`
- `entrypoint.sh:118`

### github-env-injection (severity: high)

The `run:` block in entrypoint.sh writes a value derived from an untrusted input (`$INPUT_PRETTIER_OPTIONS`, sourced from `inputs.prettier_options`) to `$GITHUB_STEP_SUMMARY` without the required sanitization step (`printf '%s' ... | tr -d '\n\r'`). An attacker-controlled value containing newline characters could inject arbitrary content into the step summary. The offending line is: `|| { PRETTIER_RESULT=$?; echo "Problem running prettier with $INPUT_PRETTIER_OPTIONS"; exit 1; } >> $GITHUB_STEP_SUMMARY`

Locations:

- `entrypoint.sh:60`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all three findings:
1. action.yml line 67: Replaced `${{ github.action_path }}/entrypoint.sh` in the `run:` block with `"$GITHUB_ACTION_PATH/entrypoint.sh"` to eliminate template interpolation inside a shell command string.
2. entrypoint.sh - all unquoted $INPUT_* expansions fixed:
   - Line 44: Quoted `prettier@$INPUT_PRETTIER_VERSION`
   - Lines 47/54: Plugin loop now collects validated plugins into an array; npm install uses `"${VALIDATED_PLUGINS[@]}"`
   - Line 59: Prettier options split into array via `read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"` and passed as `"${PRETTIER_OPTIONS_ARRAY[@]}"`
   - Line 63: `if $INPUT_CLEAN_NODE_FOLDER` → `if [ "$INPUT_CLEAN_NODE_FOLDER" = "true" ]`
   - Lines 77/80: `if [ $INPUT_ONLY_CHANGED = true ]` and `if $INPUT_ONLY_CHANGED` → `if [ "$INPUT_ONLY_CHANGED" = "true" ]`; also fixed broken `[$INPUT_ONLY_CHANGED_PR = true ]` syntax
   - Lines 96/99/106: `if $INPUT_DRY` and `if $INPUT_NO_COMMIT` → string comparisons
   - Line 110: `if $INPUT_SAME_COMMIT` → string comparison
   - Line 118: `git push origin ${INPUT_PUSH_OPTIONS:-}` → array-based safe expansion
3. entrypoint.sh line 60: Sanitized `$INPUT_PRETTIER_OPTIONS` before writing to `$GITHUB_STEP_SUMMARY` using `printf '%s' ... | tr -d '\n\r'`

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in entrypoint.sh at line 47. Replaced unquoted `for plugin in $INPUT_PRETTIER_PLUGINS` with `read -ra PLUGINS_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"` followed by `for plugin in "${PLUGINS_ARRAY[@]}"`. This safely splits the input into an array using `read -ra` (which respects IFS word splitting in a controlled way) and iterates with proper double-quoting, preventing shell metacharacter injection from attacker-controlled plugin names.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed script injection in entrypoint.sh: (1) Sanitized the attacker-controlled GITHUB_BASE_REF variable using `tr -cd 'A-Za-z0-9._/-'` to strip shell metacharacters before constructing BASE_BRANCH (line 95). (2) Quoted $BASE_BRANCH in the `git diff --name-only HEAD "$BASE_BRANCH"` command (line 102) to prevent word splitting and glob expansion.

