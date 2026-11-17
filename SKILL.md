---
name: codex-visible-diff
description: Restore reviewable file-edit diffs in Codex by editing only through the native apply_patch heredoc form. Use when Codex changes files but the inline diff or review panel shows nothing — especially with third-party or custom model providers (Amazon Bedrock, OpenRouter, local or non-GPT models) where the native apply_patch tool is unavailable and the agent falls back to shell-based writes, and when edits via python, sed, shell redirection, or a subprocess are not reviewable. Trigger phrases include codex diff not showing, apply_patch diff, edit diff 看不见, 审查面板没有改动, review panel empty, third-party model no diff, and make edits visible. For Codex CLI and Desktop including Windows PowerShell; not for non-Codex editors.
metadata:
  short-description: Edit via apply_patch heredoc so Codex shows diffs
version: 1.1.0
---

# Codex Visible Diff

Codex renders an inline edit diff (and feeds the review panel turn diff) only when a file changes through the native apply_patch tool. The exec tool intercepts a command only if it is a pure heredoc and routes it to that handler. Every other edit path writes the file but emits no diff.

## When this matters

- Third-party / custom model providers (Amazon Bedrock, OpenRouter, local or non-GPT models): the provider may reject the freeform/custom apply_patch tool shape, so Codex omits the native tool and the agent falls back to shell-based file writes that emit no diff (openai/codex issue #19416). The heredoc exec form below is intercepted regardless of model and restores the diff.
- Edits made with python, sed, perl, shell redirection, or a subprocess change the file but are not reviewable.
- The inline diff card or review panel is empty after an edit.

## The one rule

Emit exactly this as the exec command — nothing after the closing delimiter, no variables:

```text
apply_patch <<'EOF'
*** Begin Patch
*** Update File: path/to/file
@@
 context line
-removed line
+added line
*** End Patch
EOF
```

- Closing `EOF` on its own line and the LAST line of the command. No trailing shell (`Write-Output`, `Get-Content`, `;`, `&&`, …).
- Literal patch body. Never build it in a variable (`$p`) or pipe it.
- Never call `codex.exe --codex-run-as-apply-patch`, `apply_patch.bat`, python, sed, perl, `echo >`, `cat >`, `tee`, or scripts that rewrite files — they bypass interception, so no diff.
- One patch per command. For several files put multiple `*** Update File` / `*** Add File` / `*** Delete File` sections inside one `*** Begin Patch … *** End Patch`.
- If the body itself contains a line equal to the delimiter, pick a different unique delimiter (e.g. `PATCH`).

## Verify interception

Intercepted patch returns native output with `Wall time: 0 seconds` (no shell process ran) and `Success. Updated the following files: …`. A normal process exit, a `Chunk ID`, or a PowerShell parse error means NOT intercepted — fix the command form.

## Right form but still no diff in the panel

The review panel reflects Git changes. Untracked new files may need tracking; check Staged and Unstaged views. Mind `core.autocrlf` because LF/CRLF mismatch can fake whole-file diffs. Read `references/troubleshooting.md`.

## References (read only as needed)

- `references/mechanism.md` — source-level why and the exact accepted command forms.
- `references/examples.md` — Add/Update/Delete plus Windows/PowerShell and anti-patterns.
- `references/troubleshooting.md` — verification, git panel, line endings, version caveats.

## Scope

Behavioral workaround for Codex's current interception, verified against openai/codex `intercept_apply_patch` and `maybe_parse_apply_patch`. Version-dependent; the durable fix is upstream auto-detection of all filesystem edits. For always-on project enforcement also add an AGENTS.md hard rule.
