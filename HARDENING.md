<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.5

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **creyD--prettier_action/v4.5** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a), a GitHub expression placed directly in a run: block. The composite step 'Prettify code!' runs `${{ github.action_path }}/entrypoint.sh`, so the expression is substituted into the shell script before the shell parses it. Use the double-quoted runner-provided env var instead: "$GITHUB_ACTION_PATH/entrypoint.sh".

Locations:

- `action.yml:85`

### script-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. Sub-rule (b): Multiple unquoted shell variable expansions of env vars holding workflow-controllable (untrusted) input values in entrypoint.sh:
1. Line 43: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — $INPUT_PRETTIER_VERSION (from inputs.prettier_version) is unquoted, allowing shell metacharacter injection.
2. Line 55: `npm install --silent $INPUT_PRETTIER_PLUGINS` — $INPUT_PRETTIER_PLUGINS (from inputs.prettier_plugins) is unquoted, allowing word-splitting and metacharacter injection.
3. Line 62: `prettier $INPUT_PRETTIER_OPTIONS` — $INPUT_PRETTIER_OPTIONS (from inputs.prettier_options) is unquoted, allowing shell metacharacter injection. All three variables must be double-quoted (e.g., "$INPUT_PRETTIER_VERSION") to prevent command injection.

Locations:

- `entrypoint.sh:43`
- `entrypoint.sh:55`
- `entrypoint.sh:62`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 5072dcd..386d978 100644
--- a/action.yml
+++ b/action.yml
@@ -1,8 +1,6 @@
 name: Prettier Action
 description: Automatically runs prettier on all your changes.
-
 author: Conrad Großer <grosserconrad@gmail.com>
-
 inputs:
   commit_message:
     description: Commit message, will be ignored if used with same_commit
@@ -74,15 +72,30 @@ inputs:
     description: Allow other plugins to be installed. By default, we are checking if the plugins are actually prettier plugins.
     required: false
     default: false
-
 runs:
   using: "composite"
   steps:
     - name: Prettify code!
       shell: bash
       run: >-
-        PATH=$GITHUB_ACTION_PATH/node_modules/.bin:$PATH
-        ${{ github.action_path }}/entrypoint.sh
+        PATH=$GITHUB_ACTION_PATH/node_modules/.bin:$PATH "$GITHUB_ACTION_PATH/entrypoint.sh"
       env:
         INPUT_COMMIT_MESSAGE: ${{ inputs.commit_message }}
         INPUT_COMMIT_DESCRIPTION: ${{ inputs.commit_description }}
@@ -102,7 +115,6 @@ runs:
         INPUT_CLEAN_NODE_FOLDER: ${{ inputs.clean_node_folder }}
         INPUT_GIT_IDENTITY: ${{ inputs.git_identity }}
         INPUT_ALLOW_OTHER_PLUGINS: ${{ inputs.allow_other_plugins }}
-
 branding:
   icon: "award"
   color: "green"

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed entrypoint.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/entrypoint.sh b/entrypoint.sh
index 7c8f4bf..ad5c096 100755
--- a/entrypoint.sh
+++ b/entrypoint.sh
@@ -45,12 +45,13 @@ cd "$INPUT_WORKING_DIRECTORY"
 
 echo "Installing prettier..."
 
-npm install --silent prettier@$INPUT_PRETTIER_VERSION
+npm install --silent "prettier@$INPUT_PRETTIER_VERSION"
 
 # Install plugins
 if [ -n "$INPUT_PRETTIER_PLUGINS" ]; then
     if [ "$INPUT_ALLOW_OTHER_PLUGINS" != "true" ]; then
-        for plugin in $INPUT_PRETTIER_PLUGINS; do
+        read -ra _PLUGINS_CHECK_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"
+        for plugin in "${_PLUGINS_CHECK_ARRAY[@]}"; do
             echo "Checking plugin: $plugin"
             # check regex against @prettier/xyz
             if ! echo "$plugin" | grep -Eq '(@prettier\/plugin-|(@[a-z\-]+\/)?prettier-plugin-){1}([a-z\-]+)'; then
@@ -59,20 +60,22 @@ if [ -n "$INPUT_PRETTIER_PLUGINS" ]; then
             fi
         done
     fi
-    npm install --silent $INPUT_PRETTIER_PLUGINS
+    read -ra PRETTIER_PLUGINS_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"
+    npm install --silent "${PRETTIER_PLUGINS_ARRAY[@]}"
 fi
 )
 
 PRETTIER_RESULT=0
 echo "Prettifying files..."
 echo "Files:"
-prettier $INPUT_PRETTIER_OPTIONS \
+read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"
+prettier "${PRETTIER_OPTIONS_ARRAY[@]}" \
   || { PRETTIER_RESULT=$?; echo "Problem running prettier with $INPUT_PRETTIER_OPTIONS"; exit 1; } >> $GITHUB_STEP_SUMMARY
 
 echo "Prettier result: $PRETTIER_RESULT"
 
 # Removing the node_modules folder, so it doesn't get committed if it is not added in gitignore
-if $INPUT_CLEAN_NODE_FOLDER; then
+if [ "$INPUT_CLEAN_NODE_FOLDER" = "true" ]; then
   echo "Deleting node_modules/ folder..."
   if [ -d 'node_modules' ]; then
     rm -r node_modules/
@@ -89,9 +92,9 @@ fi
 
 # If running under only_changed, reset every modified file that wasn't also modified in the last commit
 # This allows only_changed and dry to work together, and simplified the non-dry logic below
-if [ $INPUT_ONLY_CHANGED = true ] || [ $INPUT_ONLY_CHANGED_PR = true ] ; then
+if [ "$INPUT_ONLY_CHANGED" = "true" ] || [ "$INPUT_ONLY_CHANGED_PR" = "true" ] ; then
   BASE_BRANCH=origin/$GITHUB_BASE_REF
-  if $INPUT_ONLY_CHANGED; then
+  if [ "$INPUT_ONLY_CHANGED" = "true" ]; then
     BASE_BRANCH=HEAD~1
   fi
 
@@ -115,10 +118,10 @@ fi
 # To keep runtime good, just continue if something was changed
 if _git_changed; then
   # case when --write is used with dry-run so if something is unpretty there will always have _git_changed
-  if $INPUT_DRY; then
+  if [ "$INPUT_DRY" = "true" ]; then
     echo "Unpretty Files Changes:"
     git diff
-    if $INPUT_NO_COMMIT; then
+    if [ "$INPUT_NO_COMMIT" = "true" ]; then
         echo "There are changes that won't be commited, you can use an external job to do so."
     else
         echo "Finishing dry-run. Exiting before committing."
@@ -131,13 +134,13 @@ if _git_changed; then
     # Add changes to git
     git add "${INPUT_FILE_PATTERN}" || echo "Problem adding your files with pattern ${INPUT_FILE_PATTERN}"
 
-    if $INPUT_NO_COMMIT; then
+    if [ "$INPUT_NO_COMMIT" = "true" ]; then
       echo "There are changes that won't be commited, you can use an external job to do so."
       exit 0
     fi
 
     # Commit and push changes back
-    if $INPUT_SAME_COMMIT; then
+    if [ "$INPUT_SAME_COMMIT" = "true" ]; then
       echo "Amending the current commit..."
       git pull
       git commit --amend --no-edit --allow-empty
@@ -148,7 +151,9 @@ if _git_changed; then
       else
           git commit -m "$INPUT_COMMIT_MESSAGE" --author="$GITHUB_ACTOR <$GITHUB_ACTOR@users.noreply.github.com>" ${INPUT_COMMIT_OPTIONS:+"$INPUT_COMMIT_OPTIONS"} || echo "No files added to commit"
       fi
-      git push origin ${INPUT_PUSH_OPTIONS:-}
+      push_args=()
+      [ -n "$INPUT_PUSH_OPTIONS" ] && read -ra push_args <<< "$INPUT_PUSH_OPTIONS"
+      git push origin "${push_args[@]+"${push_args[@]}"}"
     fi
     echo "Changes pushed successfully."
   fi

```

Locations:

- `entrypoint.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, prior-fix

**Notes:**

Nothing here was run or tested. All changes are in action.yml and entrypoint.sh.

action.yml: the composite step no longer puts `${{ github.action_path }}` inside `run:`. It now runs `PATH="$GITHUB_ACTION_PATH/node_modules/.bin:$PATH" "$GITHUB_ACTION_PATH/entrypoint.sh"`.

entrypoint.sh:
- `prettier@$INPUT_PRETTIER_VERSION` is now quoted.
- `prettier_plugins` and `prettier_options` go through a new helper, `_split_words`. Upstream left these unquoted, so the shell split them on whitespace and expanded globs (globstar is on). The helper does the same without evaluating anything: it splits on whitespace with `read -r -d '' -a`, which also handles values written over several lines, and it expands glob tokens with `compgen -G`. A pattern that matches nothing is passed through unchanged. This matters for the default `--write **/*.js`, which needs the glob expanded. The previous harden used `read -ra`, which drops globbing and reads only the first line, so I didn't repeat it. I expect `compgen -G` to follow the globstar setting but haven't checked, so a test of the default options is worth running.
- `push_options` is split the same way into an array.
- I re-applied the previous harden's boolean checks. Lines like `if $INPUT_DRY` ran the input value as a command; they are now `[ "$X" = "true" ]`. This covers clean_node_folder, only_changed, only_changed_pr, dry, no_commit and same_commit, with the variables quoted.
- Empty arrays are expanded as `${arr[@]+"${arr[@]}"}` so `set -u` doesn't fail on older bash.

One behaviour change: those flags now only count as on when the value is exactly `true`. Upstream ran the value as a command, so another spelling such as `True` won't switch a flag on any more.

`commit_options` is unchanged from upstream: it is still passed as one quoted argument.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

In entrypoint.sh, the inputs clean_node_folder, dry and no_commit were run as commands (`if $INPUT_X; then`). Each check now compares against the string instead: `if [ "$INPUT_X" = "true" ]; then`. The inputs default to true/false, so a normal run behaves the same as before.

The script also had a broken `# Commit` line left over from an earlier pass. The `same_commit` check had been pasted onto the same line as that comment, so the shell treated it as part of the comment and the script would not parse. I put it back on its own line as `if [ "$INPUT_SAME_COMMIT" = "true" ]; then`.

The edit tool damaged the file while I was making these changes: text got merged together. So I rewrote entrypoint.sh in full; nothing else in it was changed. A search afterwards found no remaining `if $INPUT` checks and no merged text in the script. I haven't run the script. action.yml needed no changes.

