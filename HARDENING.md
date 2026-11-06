<!-- markdownlint-disable -->

# Hardening Report: creyD--prettier_action/v4.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **creyD--prettier_action/v4.4** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): a GitHub Actions expression is placed directly inside the run: script of the composite step 'Prettify code!': `${{ github.action_path }}/entrypoint.sh`. Any ${{ }} inside a run: block is substituted into the script text before bash parses it. The fix is to use the runner-provided env var instead, quoted: `"$GITHUB_ACTION_PATH/entrypoint.sh"` (and quote the PATH assignment as well).

Locations:

- `action.yml:81`

### script-injection (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. Rule (b): Multiple unquoted shell variable expansions of workflow-controlled env vars in entrypoint.sh. These env vars are set from caller-supplied inputs (inputs.prettier_version, inputs.prettier_plugins, inputs.prettier_options) and are expanded without double-quoting, allowing shell metacharacter injection: (1) Line 48: `npm install --silent prettier@$INPUT_PRETTIER_VERSION` — unquoted $INPUT_PRETTIER_VERSION; (2) Line 52: `for plugin in $INPUT_PRETTIER_PLUGINS; do` — unquoted $INPUT_PRETTIER_PLUGINS used in word-splitting loop; (3) Line 60: `npm install --silent $INPUT_PRETTIER_PLUGINS` — unquoted $INPUT_PRETTIER_PLUGINS; (4) Line 67: `prettier $INPUT_PRETTIER_OPTIONS \` — unquoted $INPUT_PRETTIER_OPTIONS. An attacker controlling these inputs can inject arbitrary shell commands via metacharacters (`;`, `|`, `&`, `$(...)`, etc.).

Locations:

- `entrypoint.sh:48`
- `entrypoint.sh:52`
- `entrypoint.sh:60`
- `entrypoint.sh:67`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 54dbc65..5fbc820 100644
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
@@ -70,15 +68,30 @@ inputs:
     description: Which identity is used for git name/email when committing changes. Needs to be one of "actions" or "author".
     required: false
     default: "actions"
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
@@ -97,7 +110,6 @@ runs:
         INPUT_GITHUB_TOKEN: ${{ inputs.github_token }}
         INPUT_CLEAN_NODE_FOLDER: ${{ inputs.clean_node_folder }}
         INPUT_GIT_IDENTITY: ${{ inputs.git_identity }}
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
index 7ba5200..fecdac7 100755
--- a/entrypoint.sh
+++ b/entrypoint.sh
@@ -45,11 +45,17 @@ cd "$INPUT_WORKING_DIRECTORY"
 
 echo "Installing prettier..."
 
-npm install --silent prettier@$INPUT_PRETTIER_VERSION
+# Validate prettier version to prevent injection (allow only alphanumeric, dots, hyphens, and 'latest')
+if ! echo "$INPUT_PRETTIER_VERSION" | grep -Eq '^(latest|[0-9]+\.[0-9]+(\.[0-9]+)?([-+][a-zA-Z0-9._-]*)?)$'; then
+    echo "Invalid prettier version: $INPUT_PRETTIER_VERSION"
+    exit 1
+fi
+npm install --silent "prettier@${INPUT_PRETTIER_VERSION}"
 
 # Install plugins
 if [ -n "$INPUT_PRETTIER_PLUGINS" ]; then
-    for plugin in $INPUT_PRETTIER_PLUGINS; do
+    read -ra PRETTIER_PLUGINS_ARRAY <<< "$INPUT_PRETTIER_PLUGINS"
+    for plugin in "${PRETTIER_PLUGINS_ARRAY[@]}"; do
         echo "Checking plugin: $plugin"
         # check regex against @prettier/xyz
         if ! echo "$plugin" | grep -Eq '(@prettier\/plugin-|(@[a-z\-]+\/)?prettier-plugin-){1}([a-z\-]+)'; then
@@ -57,20 +63,21 @@ if [ -n "$INPUT_PRETTIER_PLUGINS" ]; then
             exit 1
         fi
     done
-    npm install --silent $INPUT_PRETTIER_PLUGINS
+    npm install --silent "${PRETTIER_PLUGINS_ARRAY[@]}"
 fi
 )
 
 PRETTIER_RESULT=0
 echo "Prettifying files..."
 echo "Files:"
-prettier $INPUT_PRETTIER_OPTIONS \
-  || { PRETTIER_RESULT=$?; echo "Problem running prettier with $INPUT_PRETTIER_OPTIONS"; exit 1; } >> $GITHUB_STEP_SUMMARY
+read -ra PRETTIER_OPTIONS_ARRAY <<< "$INPUT_PRETTIER_OPTIONS"
+prettier "${PRETTIER_OPTIONS_ARRAY[@]}" \
+  || { PRETTIER_RESULT=$?; echo "Problem running prettier with $INPUT_PRETTIER_OPTIONS"; exit 1; } >> "$GITHUB_STEP_SUMMARY"
 
 echo "Prettier result: $PRETTIER_RESULT"
 
 # Removing the node_modules folder, so it doesn't get committed if it is not added in gitignore
-if $INPUT_CLEAN_NODE_FOLDER; then
+if [ "$INPUT_CLEAN_NODE_FOLDER" = "true" ]; then
   echo "Deleting node_modules/ folder..."
   if [ -d 'node_modules' ]; then
     rm -r node_modules/
@@ -87,9 +94,9 @@ fi
 
 # If running under only_changed, reset every modified file that wasn't also modified in the last commit
 # This allows only_changed and dry to work together, and simplified the non-dry logic below
-if [ $INPUT_ONLY_CHANGED = true ] || [$INPUT_ONLY_CHANGED_PR = true ] ; then
+if [ "$INPUT_ONLY_CHANGED" = "true" ] || [ "$INPUT_ONLY_CHANGED_PR" = "true" ] ; then
   BASE_BRANCH=origin/$GITHUB_BASE_REF
-  if $INPUT_ONLY_CHANGED; then
+  if [ "$INPUT_ONLY_CHANGED" = "true" ]; then
     BASE_BRANCH=HEAD~1
   fi
 
@@ -113,10 +120,10 @@ fi
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
@@ -129,13 +136,13 @@ if _git_changed; then
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
@@ -146,7 +153,12 @@ if _git_changed; then
       else
           git commit -m "$INPUT_COMMIT_MESSAGE" --author="$GITHUB_ACTOR <$GITHUB_ACTOR@users.noreply.github.com>" ${INPUT_COMMIT_OPTIONS:+"$INPUT_COMMIT_OPTIONS"} || echo "No files added to commit"
       fi
-      git push origin ${INPUT_PUSH_OPTIONS:-}
+      # Build push opti
... (truncated)
```

Locations:

- `entrypoint.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, prior-fix

**Notes:**

action.yml: the run: script no longer contains ${{ github.action_path }}. It now sets PATH="$GITHUB_ACTION_PATH/node_modules/.bin:$PATH" and runs "$GITHUB_ACTION_PATH/entrypoint.sh", both quoted.

entrypoint.sh:
- The prettier version is quoted: "prettier@${INPUT_PRETTIER_VERSION}". I dropped the previous harden's version regex check because it could reject valid npm version specs.
- prettier_plugins, prettier_options and push_options are split on whitespace (newlines included) into bash arrays with `read -r -d '' -a ... || true`, which matches how upstream's unquoted $VAR split them. Each array is guarded by an emptiness check and expanded as ${arr[@]+"${arr[@]}"} so set -u doesn't fail on an empty array in bash 3.2.
- One small behaviour change: the shell no longer expands the globs in prettier_options, so prettier gets patterns like **/*.js as literal text and matches them itself.
- Re-applied the previous harden's fixes: the boolean `if $INPUT_X` checks, which ran the input's value as a command, are now `[ "$X" = "true" ]` tests. The broken `[$INPUT_ONLY_CHANGED_PR = true ]` test is fixed, so only_changed_pr now takes effect where before it was never true. $BASE_BRANCH, $GITHUB_STEP_SUMMARY and $HOME/.netrc are now quoted.
- commit_options was left as upstream's ${INPUT_COMMIT_OPTIONS:+"$INPUT_COMMIT_OPTIONS"}.

### Iteration 1

**Fixes applied:** script-injection

**Notes:**

entrypoint.sh: replaced `if $INPUT_DRY; then` and `if $INPUT_NO_COMMIT; then` with `[ "$VAR" = "true" ]` comparisons, matching the style already used elsewhere in the script (e.g. INPUT_NO_COMMIT check in the non-dry branch). Behavior for true/false inputs is unchanged.

