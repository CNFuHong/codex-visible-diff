# codex-visible-diff

**A Codex skill that restores reviewable file-edit diffs by routing every edit through the native `apply_patch` heredoc form.**

Language: [中文](./README.md) · **English**

Codex renders an inline edit diff—and feeds the review panel—only when a file is changed through its native `apply_patch` tool. Edits made with shell commands, Python, `sed`, redirection, or a subprocess modify the file on disk but emit **no diff**. This skill encodes the one reliable rule that keeps changes reviewable, across models and platforms.

- Repository: `CNFuHong/codex-visible-diff`
- Skill entrypoint: [`SKILL.md`](./SKILL.md) · Setup guide: [`get-started.md`](./get-started.md)
- License: MIT

## When to use

| Scenario | Symptom |
| --- | --- |
| Third-party / custom model providers (Amazon Bedrock, OpenRouter, local or non-GPT models) | The provider rejects the freeform/custom `apply_patch` tool shape, so Codex omits the native tool and the agent falls back to shell-based writes — no diff ([openai/codex#19416](https://github.com/openai/codex/issues/19416)). |
| Edits via `python` / `sed` / `perl` / `echo >` / `cat >` / `tee` / redirection | The file changes on disk but never enters the diff viewer. |
| Empty inline diff or review panel after an edit | Changes are not presented as reviewable diffs. |

The fix is model-independent: the pure `apply_patch` heredoc is intercepted by the exec tool and routed to the native diff-emitting handler regardless of which model is running.

## How it works

Codex's exec handler calls `intercept_apply_patch` → `maybe_parse_apply_patch`, which recognizes **only** a pure heredoc invocation and forwards it to the native `ApplyPatchHandler`. That handler emits diff deltas through `SharedTurnDiffTracker`, producing the inline diff card and the review-panel turn diff. Any other command form runs as an ordinary shell process and emits nothing. See [`references/mechanism.md`](./references/mechanism.md) for the source-level call chain.

## Installation

### Copy prompt (recommended)

Paste this to your AI agent; it reads the setup guide and installs the skill automatically:

```text
Read https://raw.githubusercontent.com/CNFuHong/codex-visible-diff/HEAD/get-started.md and set up codex-visible-diff
```

If the repository is already local, the agent can read `get-started.md` directly from the project directory.

### Manual

- **As a skill:** place this repository's contents in `<skills>/codex-visible-diff/` (e.g. `~/.codex/skills/codex-visible-diff/`), keeping `SKILL.md` at that folder's root. Start a new session.
- **As a project rule (most reliable):** copy the "The one rule" section of `SKILL.md` into the project's `AGENTS.md`.
- **From Git:** `git clone https://github.com/CNFuHong/codex-visible-diff`, then install as above.

## The rule

Emit exactly this as the exec command — the closing delimiter is the last line, with nothing after it and no variables:

```text
apply_patch <<'EOF'
*** Begin Patch
*** Update File: path/to/file
@@
 context line
-old line
+new line
*** End Patch
EOF
```

Do **not** use variables, quoted arguments, trailing shell commands, `codex.exe --codex-run-as-apply-patch`, `apply_patch.bat`, or script-based file writes. Full examples and anti-patterns: [`references/examples.md`](./references/examples.md).

## Verification

An intercepted patch returns native output with `Wall time: 0 seconds` and `Success. Updated the following files: …` (no shell process ran). A normal process exit, a `Chunk ID`, or a PowerShell parse error means the command was **not** intercepted. Troubleshooting: [`references/troubleshooting.md`](./references/troubleshooting.md).

## Agent entrypoint

If you are an AI agent and the user reports any of the following, read [`SKILL.md`](./SKILL.md) and follow it (then `references/` as needed):

`codex diff not showing` · `apply_patch diff` · `edit diff 看不见` · `审查面板没有改动` · `review panel empty` · `third-party model no diff` · `make edits visible` · edits via python/sed/shell not reviewable.

## References

- OpenAI docs — Apply Patch: <https://developers.openai.com/api/docs/guides/tools-apply-patch>
- OpenRouter docs — Apply Patch server tool (third-party model scenario): <https://openrouter.ai/docs/guides/features/server-tools/apply-patch>
- openai/codex issue #19416 — third-party providers and the freeform/custom `apply_patch` tool shape: <https://github.com/openai/codex/issues/19416>
- OpenAI community forum — diffs only via `apply_patch`; review panel reflects Git changes: <https://community.openai.com/t/1395141>
- Codex source — `codex-rs/core/src/tools/handlers/apply_patch.rs`, `codex-rs/apply-patch/src/invocation.rs`.

## Limitations

- Behavioral workaround for Codex's current interception; version-dependent. The durable fix is upstream auto-detection of all filesystem edits.
- The review panel reflects Git changes: untracked new files may need tracking; mind `core.autocrlf` line endings.
- Applies to Codex CLI / Desktop (including Windows PowerShell), not to non-Codex editors.

## License

[MIT](./LICENSE).
