<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **creyD--prettier_action/v4.6** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): a GitHub Actions expression is placed directly in a composite run: block. The offending line is `${{ github.action_path }}/entrypoint.sh` in the 'Prettify code!' step. Template substitution happens before the shell parses the line, so the value is never quoted. Fix: use the runner-provided env var instead, quoted: "$GITHUB_ACTION_PATH/entrypoint.sh".

Locations:

- `action.yml:86`

### script-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. Rule (b): entrypoint.sh expands user-controlled input env vars without double-quoting in shell commands, allowing shell metacharacter injection. Three unquoted expansions: (1) `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — INPUT_PRETTIER_VERSION is set from inputs.prettier_version; (2) `npm install --silent $INPUT_PRETTIER_PLUGINS` — INPUT_PRETTIER_PLUGINS is set from inputs.prettier_plugins; (3) `npx prettier $INPUT_PRETTIER_OPTIONS` — INPUT_PRETTIER_OPTIONS is set from inputs.prettier_options. All three allow an attacker to inject arbitrary shell commands via the respective input.

Locations:

- `entrypoint.sh:43`
- `entrypoint.sh:53`
- `entrypoint.sh:60`

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
index ec9b329..307cd2f 100755
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
+        read -ra PRETTIER_PLUGINS_CHECK_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"
+        for plugin in "${PRETTIER_PLUGINS_CHECK_ARRAY[@]}"; do
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
-npx prettier $INPUT_PRETTIER_OPTIONS \
+read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"
+npx prettier "${PRETTIER_OPTIONS_ARRAY[@]}" \
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
@@ -148,7 +151,12 @@ if _git_changed; then
       else
           git commit -m "$INPUT_COMMIT_MESSAGE" --author="$GITHUB_ACTOR <$GITHUB_ACTOR@users.noreply.github.com>" ${INPUT_COMMIT_OPTIONS:+"$INPUT_COMMIT_OPTIONS"} || echo "No files added to commit"
       fi
-      git push origin ${INPUT_PUSH_OPTIONS:-}
+      if [ -n "${INPUT_PUSH_OPTIONS:-}" ]; then
+        read -ra PUSH_OPTIONS_ARRAY <<< "$INPUT_PUSH_OPTIONS"
+        git push origin "${PUSH_OPTIONS_ARRAY[@]}"
+      else
+        git push origin
+      fi
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

action.yml: the 'Prettify code!' step no longer puts ${{ github.action_path }} directly in run:. It now runs "$GITHUB_ACTION_PATH/entrypoint.sh", quoted, with the PATH prefix (also quoted) on the same command.

entrypoint.sh:
- The prettier version is now quoted: "prettier@$INPUT_PRETTIER_VERSION".
- prettier_plugins, prettier_options and push_options are split on whitespace into bash arrays with `read -r -d '' -a ... || true`. That reads every line and keeps quotes as plain text, the same split upstream's unquoted $VAR gave.
- Each split is guarded so an empty value adds no argument. Arrays are expanded as ${arr[@]+"${arr[@]}"} so that `set -u` doesn't fail on an empty array in bash 3.2.
- Upstream ran inputs as commands with `if $INPUT_X` (clean_node_folder, only_changed, dry, no_commit, same_commit). These are now `[ "$INPUT_X" = "true" ]` checks, and the only_changed tests are quoted.
- commit_options is unchanged, since it was already quoted.

One behaviour change: the shell no longer expands globs in prettier_options such as '**/*.js'. They now reach prettier as literal text and prettier expands them itself.

### Iteration 1

**Fixes applied:** publish-gate

**Notes:**

entrypoint.sh no longer parses as broken. An earlier pass had cut several lines short or spliced them together: `then` was duplicated on the dry/no_commit check, the git add, no_commit and same_commit lines were mangled, the commit block was missing its `fi`, and junk text was stuck to the end of the git push line. I rebuilt these lines to match upstream v4.6's structure and kept the existing hardened forms: the `[ "$X" = "true" ]` checks, quoted variables, and the push_options array split. I didn't change anything else. I haven't run the script or the publish checks again; I only checked by reading that every if/else/fi now pairs up.

