# codex-visible-diff

**一个 Codex 技能：让所有文件编辑都走原生 `apply_patch` 的纯 heredoc 形式，从而恢复可审查的编辑 diff。**

语言：**中文** · [English](./README.en.md)

Codex 只有在文件经原生 `apply_patch` 工具改动时，才会渲染内联编辑 diff 并送入审查面板。用 shell 命令、Python、`sed`、重定向或子进程改文件，磁盘上的确变了，却**不发 diff**。本技能把「让改动可审查」的唯一可靠规则固化下来，跨模型、跨平台通用。

- 仓库：`CNFuHong/codex-visible-diff`
- 技能入口：[`SKILL.md`](./SKILL.md) · 安装指南：[`get-started.md`](./get-started.md)
- 许可：MIT

## 何时使用

| 场景 | 表现 |
| --- | --- |
| 第三方 / 自定义模型提供商（Amazon Bedrock、OpenRouter、本地或非 GPT 模型） | 提供商拒绝 freeform/custom 形状的 `apply_patch` 工具，Codex 因而不暴露原生工具，agent 退化为 shell 写文件——无 diff（[openai/codex#19416](https://github.com/openai/codex/issues/19416)）。 |
| 用 `python` / `sed` / `perl` / `echo >` / `cat >` / `tee` / 重定向 改文件 | 文件改了，但从不进入 diff 视图。 |
| 编辑后内联 diff 或审查面板为空 | 改动没有以可审查的 diff 呈现。 |

修复与模型无关：纯 `apply_patch` heredoc 会被 exec 工具拦截并转交原生发 diff 的 handler，无论当前跑的是哪个模型。

## 工作原理

Codex 的 exec handler 调用 `intercept_apply_patch` → `maybe_parse_apply_patch`，**只**识别纯 heredoc 调用并转交原生 `ApplyPatchHandler`。该 handler 经 `SharedTurnDiffTracker` 发出 diff 增量，生成内联 diff 卡片与审查面板的本轮 diff。其它命令形式都按普通 shell 进程执行，不发任何 diff。源码级调用链见 [`references/mechanism.md`](./references/mechanism.md)。

## 安装

### Copy prompt（推荐）

把下面这句粘贴给你的 AI agent，它会读取安装指南并自动装好本技能：

```text
Read https://raw.githubusercontent.com/CNFuHong/codex-visible-diff/HEAD/get-started.md and set up codex-visible-diff
```

若仓库已在本地，agent 也可直接读项目目录里的 `get-started.md`。

### 手动

- **作为 skill：** 把本仓库内容放进 `<skills>/codex-visible-diff/`（如 `~/.codex/skills/codex-visible-diff/`），保持 `SKILL.md` 在该目录根。新开会话。
- **作为项目规则（最稳）：** 把 `SKILL.md` 的「The one rule」段落抄进项目 `AGENTS.md`。
- **从 Git：** `git clone https://github.com/CNFuHong/codex-visible-diff`，再按上面放置。

## 规则

整个 exec 命令就是下面这样——结束定界符是最后一行，其后不跟任何内容，不用变量：

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

不要使用变量、引号参数、尾部 shell 命令、`codex.exe --codex-run-as-apply-patch`、`apply_patch.bat` 或脚本写文件。完整示例与反模式见 [`references/examples.md`](./references/examples.md)。

## 验证

被拦截的补丁返回原生输出：`Wall time: 0 seconds` 与 `Success. Updated the following files: …`（没有真实 shell 进程运行）。若看到普通进程退出、`Chunk ID` 或 PowerShell 解析错误，说明**没被拦截**。排错见 [`references/troubleshooting.md`](./references/troubleshooting.md)。

## AI 入口

如果你是 AI agent，用户报告下列任一情况时，读取 [`SKILL.md`](./SKILL.md) 并照做（按需再读 `references/`）：

`codex diff not showing` · `apply_patch diff` · `edit diff 看不见` · `审查面板没有改动` · `review panel empty` · `third-party model no diff` · `make edits visible` · 用 python/sed/shell 改文件不可审查。

## 参考来源

- OpenAI 官方文档 — Apply Patch：<https://developers.openai.com/api/docs/guides/tools-apply-patch>
- OpenRouter 官方文档 — Apply Patch 服务端工具（第三方模型场景）：<https://openrouter.ai/docs/guides/features/server-tools/apply-patch>
- openai/codex issue #19416 — 第三方提供商与 freeform/custom `apply_patch` 工具形状：<https://github.com/openai/codex/issues/19416>
- OpenAI 官方论坛 — 只有 `apply_patch` 进 diff；审查面板反映 Git 变更：<https://community.openai.com/t/1395141>
- Codex 源码 — `codex-rs/core/src/tools/handlers/apply_patch.rs`、`codex-rs/apply-patch/src/invocation.rs`。

## 局限

- 针对 Codex 当前拦截行为的行为层方案，版本相关；根治在上游自动检测所有文件系统改动。
- 审查面板反映 Git 变更：未跟踪的新文件可能需先跟踪；注意 `core.autocrlf` 行尾。
- 适用 Codex CLI / Desktop（含 Windows PowerShell），不适用非 Codex 编辑器。

## 许可

[MIT](./LICENSE)。
