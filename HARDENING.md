<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **creyD--prettier_action/v4.4** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Rule (a): A ${{ }} expression is directly interpolated inside a run: shell command string in action.yml. The line `${{ github.action_path }}/entrypoint.sh` embeds the github.action_path context value directly into the shell command before the shell ever sees it, allowing YAML template substitution to inject arbitrary shell metacharacters. Even though github.action_path is typically safe, any ${{ ... }} directly in a run: block is a script-injection finding per the check rules.

Locations:

- `action.yml:81`

### script-injection (severity: high)

Rule (b): Multiple unquoted shell variable expansions of workflow-controlled env vars in entrypoint.sh. These env vars are set from caller-supplied inputs (inputs.prettier_version, inputs.prettier_plugins, inputs.prettier_options) and are expanded without double-quoting, allowing shell metacharacter injection: (1) Line 48: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — unquoted $INPUT_PRETTIER_VERSION; (2) Line 52: `for plugin in $INPUT_PRETTIER_PLUGINS; do` — unquoted $INPUT_PRETTIER_PLUGINS used in word-splitting loop; (3) Line 60: `npm install --silent $INPUT_PRETTIER_PLUGINS` — unquoted $INPUT_PRETTIER_PLUGINS; (4) Line 67: `prettier $INPUT_PRETTIER_OPTIONS \` — unquoted $INPUT_PRETTIER_OPTIONS. An attacker controlling these inputs can inject arbitrary shell commands via metacharacters (`;`, `|`, `&`, `$(...)`, etc.).

Locations:

- `entrypoint.sh:48`
- `entrypoint.sh:52`
- `entrypoint.sh:60`
- `entrypoint.sh:67`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

Fixed two script-injection findings:

1. action.yml (line 81): Replaced `${{ github.action_path }}/entrypoint.sh` with `"$GITHUB_ACTION_PATH/entrypoint.sh"`. GitHub Actions sets GITHUB_ACTION_PATH as an environment variable equivalent to github.action_path, so using it avoids YAML template substitution entirely.

2. entrypoint.sh (lines 48, 52, 60, 67): Fixed four unquoted variable expansions:
   - Line 48: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` → `npm install --silent "prettier@$INPUT_PRETTIER_VERSION"` (quoted to prevent metacharacter injection)
   - Lines 52/60: Replaced unquoted `$INPUT_PRETTIER_PLUGINS` in for-loop and npm install with `read -ra PRETTIER_PLUGINS_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"` and `"${PRETTIER_PLUGINS_ARRAY[@]}"` array expansion (safe word-splitting without shell metacharacter injection)
   - Line 67: Replaced unquoted `$INPUT_PRETTIER_OPTIONS` with `read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"` and `"${PRETTIER_OPTIONS_ARRAY[@]}"` array expansion (same pattern)

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all script injection vulnerabilities in entrypoint.sh:
1. Replaced all `if $INPUT_BOOL; then` patterns (INPUT_DRY, INPUT_CLEAN_NODE_FOLDER, INPUT_SAME_COMMIT, INPUT_NO_COMMIT, INPUT_ONLY_CHANGED, INPUT_ONLY_CHANGED_PR) with safe string comparisons `if [ "$INPUT_VAR" = "true" ]; then` — prevents attacker-controlled values from being executed as shell commands.
2. Fixed unquoted `[ $INPUT_ONLY_CHANGED = true ]` to `[ "$INPUT_ONLY_CHANGED" = "true" ]` — prevents word-splitting and glob expansion.
3. Fixed `git push origin ${INPUT_PUSH_OPTIONS:-}` by building a bash array with `read -ra push_args <<< "$INPUT_PUSH_OPTIONS"` and expanding as `"${push_args[@]+"${push_args[@]}"}"` — prevents shell metacharacter injection.
4. Added strict regex validation of `INPUT_PRETTIER_VERSION` before using it in `npm install --silent "prettier@${INPUT_PRETTIER_VERSION}"` — prevents command substitution injection within the double-quoted string.

