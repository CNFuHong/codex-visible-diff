# Mechanism (source-level)

Why only the pure heredoc form produces a visible diff. Verified against openai/codex (main branch).

## Call chain

1. `codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs` builds `prepared.resolved.command` (the shell-wrapped argv) and calls `intercept_apply_patch(&prepared.resolved.command, cwd, fs, …, tracker, call_id, "exec_command")`. If it returns `Some(output)`, the exec tool returns that output directly and no shell process runs.
2. `codex-rs/core/src/tools/handlers/apply_patch.rs`: `intercept_apply_patch` → `prepare_intercepted_patch` → `codex_apply_patch::maybe_parse_apply_patch_verified(command, …)`. On `Body(changes)` it runs `execute_prepared_patch(request, Some(&tracker), tool_ctx)`. The native `ApplyPatchHandler` emits diff deltas through `SharedTurnDiffTracker` / `ToolEmitter` (`ApplyPatchArgumentDiffConsumer.consume_diff` → `push_delta(call_id, diff)`), which the inline diff card and the review panel turn-diff render.
3. `codex-rs/apply-patch/src/invocation.rs`: `maybe_parse_apply_patch(argv, cwd)` accepts exactly two shapes:
   - Direct: `argv == [cmd, body]`, `cmd ∈ {apply_patch, applypatch}`, and `parse_patch(body)` succeeds.
   - Shell heredoc: `parse_shell_script(argv)` matches `[shell, flag, script]` or `[shell, skip_flag, flag, script]`, then `extract_apply_patch_from_bash(script)` pulls the heredoc body.
   - `classify_shell`: bash/zsh/sh with `-lc|-c` → Unix; pwsh/powershell with `-command` → PowerShell; cmd with `/c` → Cmd. `can_skip_flag`: powershell `-noprofile`. Unix/PowerShell/Cmd all use `extract_apply_patch_from_bash`, so the bash-style `apply_patch <<'DELIM' … DELIM` heredoc is recognized regardless of shell.

## Why other forms fail

- On Windows the resolved command is shell-wrapped (e.g. `powershell -command "<script>"`), so the direct `[cmd, body]` shape rarely matches; recognition depends on the heredoc inside the script.
- A variable (`apply_patch $p`), a quoted multi-line argument, trailing commands after the delimiter, or a wrapper/subprocess (`codex.exe --codex-run-as-apply-patch`, `apply_patch.bat`, python/sed) do not present a clean `apply_patch <<'DELIM' … DELIM` heredoc → `NotApplyPatch` / `ShellParseError` → the command runs as an ordinary shell process → the file changes on disk but no diff is emitted to the tracker.

## Consequence

The diff switch is the command string the model emits. No external tool can force interception, because invoking that tool is itself an ordinary shell command. The fix is therefore behavioral (always emit the pure heredoc), not a runtime wrapper.

## Review panel is Git-based

OpenAI Support confirms the Desktop review panel reflects Git changes (check Staged/Unstaged). The native handler additionally feeds the per-turn diff. Filesystem edits made outside the native handler still exist on disk and show in Git, but not as the inline per-edit card.

## Provider and model context

The native apply_patch tool is offered to the model as a tool spec whose shape depends on `apply_patch_tool_type` (`freeform`/custom vs `function`). Some third-party/custom providers (e.g. Amazon Bedrock) accept only `function`/`mcp` tool specs and reject the Responses `custom` shape, so Codex cannot expose the native freeform apply_patch tool for those providers and the agent falls back to shell-based file writes (openai/codex issue #19416, "Fix: use function apply_patch tool for Bedrock model"). Community reports describe the same shell fallback for non-GPT models routed through OpenRouter.

This is why the workaround is the heredoc exec form rather than "call the native tool": the exec interceptor (`intercept_apply_patch`) recognizes the pure heredoc command and routes it to the native diff-emitting handler regardless of whether the model was offered the native apply_patch tool. The diff is restored even when the provider cannot expose the tool directly.
