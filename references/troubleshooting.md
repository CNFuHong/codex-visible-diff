# Troubleshooting

## Confirm interception actually happened

Run a tiny pure-heredoc patch (e.g. add a scratch file) and inspect the tool output:
- Intercepted: `Wall time: 0 seconds`, `Exit code: 0`, `Success. Updated the following files: A …`. No real process ran.
- Not intercepted: a normal `Chunk ID` / process exit code / wall time > 0, or a PowerShell `ParserError` (e.g. "Missing file specification after redirection operator" means the heredoc reached PowerShell instead of being intercepted).

Clean up the scratch file with a pure-heredoc `*** Delete File` patch.

## Pure heredoc but the review panel is empty

The panel reflects Git changes (OpenAI Support). Check:
1. View: switch between Staged / Unstaged (and Last turn vs all changes if available).
2. Untracked new files: a brand-new file may not appear until Git tracks it. `git add -N <file>` (intent-to-add) makes it show in diff without staging content; reversible.
3. Repo scope: the panel is scoped to the workspace Git root. Nested repos / submodules may need the right root selected.
4. Refresh: keep the app current; some Windows builds had a "Review tab stuck on Loading diff" bug — update and re-open.

## Whole-file fake diffs (line endings)

With `core.autocrlf=true`, a file written as LF while the repo expects CRLF can show as a whole-file change. Keep edited files consistent with the repo's line endings; native apply_patch preserves existing endings (`CODEX_APPLY_PATCH_PRESERVE_LINE_ENDINGS`). Avoid rewriting a file's endings wholesale.

## Still nothing

- Confirm the session exposes exec_command/apply_patch interception. Some sessions lack the native apply_patch tool; the heredoc exec form is the workaround.
- The inline per-edit card requires the native handler. Edits made earlier via subprocess/python are on disk and in Git but were not emitted as inline cards. Emit subsequent edits via pure heredoc.

## Version caveat

This relies on Codex's current interception behavior. If a future Codex auto-detects all filesystem edits (the upstream feature request), the workaround becomes unnecessary. Re-verify against the running version when diffs behave unexpectedly.
