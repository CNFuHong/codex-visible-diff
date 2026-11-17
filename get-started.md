# Get started — codex-visible-diff

本指南供 AI agent（Codex）读取并执行。目标：让 Codex 的文件编辑显示可见 diff——安装本技能或把其规则写入项目。

仓库：`https://github.com/CNFuHong/codex-visible-diff`，raw 前缀：`https://raw.githubusercontent.com/CNFuHong/codex-visible-diff/HEAD/`。

## 方式 A — 安装为 skill（推荐）

1. 确定用户的 Codex skills 目录（如 `~/.codex/skills` 或 `~/.agents/skills`）。
2. 创建 `<skills>/codex-visible-diff/`，从本仓库复制以下文件（保持相对路径）：
   - `SKILL.md`
   - `agents/openai.yaml`
   - `references/mechanism.md`、`references/examples.md`、`references/troubleshooting.md`
可用 raw URL 逐个抓取：`https://raw.githubusercontent.com/CNFuHong/codex-visible-diff/HEAD/<path>`。
3. 新开 Codex 会话；技能由其 description 关键词被发现。

## 方式 B — 项目硬规则（最稳）

把 `SKILL.md` 的「The one rule」段落追加进项目 `AGENTS.md`，使每次编辑都用纯 heredoc apply_patch。

## 验证

用纯 heredoc 形式做一个极小改动（见 `references/examples.md`）。被拦截的输出含 `Wall time: 0 seconds` 与 `Success. Updated the following files: …`。若看到普通进程退出或 PowerShell 解析错误，说明命令形式不对。

## 一行规则

只用 `apply_patch <<'EOF' … *** End Patch … EOF` 改文件：结束定界符单独成行且为最后一行，其后不跟任何命令，不用变量，不用子进程/python/sed。
