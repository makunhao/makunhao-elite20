# C5A AI 日志

## Round 1 · 读规格，先定"什么算读懂"

**prompt 要点**：
> 「C5A 要求回答四个问题：仓库做什么、目录结构、有哪些技能、哪个最有意思。先告诉我：一份'能被人核查'的仓库分析应该包含哪些可验证事实？不要写感受。」

**输出**：文件数、语言分布、源码行数、测试行数、入口与模块、技能清单、版本与依赖。
**采纳**：把它变成 `repo_report.py` 的输出项，先出事实再写判断。

---

## Round 2 · clone 失败，换通道

**动作**：`git clone --depth 50 https://github.com/neolaf2/neoskills.git`
**结果**：`error: RPC failed; curl 7 GnuTLS recv error (-110)`，重试三次均失败。

**AI 建议**：「调大 `http.postBuffer` / 换 `--depth 1` / 关 SSL 校验。」
**我的处理**：先试 `--depth 1` 与 `http.version=HTTP/1.1`，仍失败。关 SSL 校验会掩盖问题，不采纳。
**最终方案**：`curl https://codeload.github.com/.../zip/refs/heads/main` 成功（248 KB）。

**记录**：这不是"clone 成功"，交付里如实写"clone 失败，改用 zip"。

---

## Round 3 · 生成客观事实

**prompt 要点**：
> 「用 repo_report.py 输出这个仓库的结构、文件类型、SKILL.md 清单与 README 章节。不要推断，只列能数出来的。」

**结果**：112 文件 / 84 py / src 7,531 行 / tests 1,993 行 / 2 skills / 4 agents / 1 command。

**我的修正**：脚本把"非 git 仓库"写进报告（zip 没有 `.git`），这是事实，保留；没有伪造提交数。

---

## Round 4 · 读架构（人读，AI 只做索引）

**做法**：先让 AI 列出 `src/neoskills/` 的模块职责，再逐个打开 `core/linker.py`、`core/mode.py`、`ontology/graph.py`、`cli/main.py` 核对。

**AI 的一处错误**：它说"技能通过复制部署到 agent 目录"。
**核对后纠正**：`linker.py` 明确用 `symlink_to`，README 也写 "zero-copy"。复制只发生在"目标已有真实目录"时的备份。
**教训**：架构描述必须以代码为准，不能以"常见做法"推断。

---

## Round 5 · 找可核查的改进点

**prompt 要点**：
> 「不要给'建议加更多测试'这类空话。每条建议必须满足：① 能在仓库里指出具体文件/行；② 能说清不改会怎样；③ 改法不超过两句话。」

**产出 6 条**，其中两条是真实缺陷：

1. `dedup_scan.py:27-34` 导入 `neoskills.mappings.resolver` / `neoskills.bank.store`，这两个模块在 v0.4 不存在 → `NEOSKILLS_AVAILABLE` 永远 False（死代码）。
2. `linker.py:53` 与 `dedup_scan.py:466` 直接用 `symlink`，Windows 无权限时无降级路径。
3. 版本号漂移：`pyproject.toml` 0.4.2 vs `CLAUDE.md`/`TASKS.md` 0.4.1。
4. `ontology validate` 对 domain 边误报（仓库自己在 Known Issues 里承认）。
5. `compose` 只有 spec 没有 executor（TASKS 第 4 条）。
6. 图缓存用 mtime 失效不可靠，建议内容哈希。

**AI 建议过**：「可以说'测试覆盖不足'。」
**我否掉**：没有覆盖率数据，且 172 个测试是仓库自述；未跑过就不下结论，改放"未验证"一节。

---

## Round 6 · 截图

- GitHub 页面截图失败：Chromium `ERR_CONNECTION_REFUSED`；同一时刻 curl 也时通时断。
- 改为渲染真实终端输出：`facts.txt`（五条命令的原始输出）→ `01_terminal_facts.png`。
- 文件命名与内容一致，不把渲染图说成"网页截图"。

---

## Round 7 · 拿到 token 后的线上操作

**输入**：用户名 + 临时 PAT（repo + workflow，7 天）。

**做了什么**：
1. `GET /user` 验证身份 → `makunhao`。
2. 建个人仓库 `makunhao-elite20`，用 Contents API 写入 README。
3. Fork `neolaf2/neoskills` → `makunhao/neoskills`。
4. 带 token `git clone` fork —— **成功**（之前无 token 时 TLS 中断）。
5. 建分支 `c5a-cn-notes`，提交 `C5A_中文笔记.md`（API 提交 `69c6850`，本地补充 `2814cb4`）。
6. `git fetch --unshallow` 补全历史：上游 main 40 commits（2026-02-06 → 2026-03-30）。

**结论**：之前的"clone 失败"与认证/网关都有关，不完全是网络差。
**没有做的**：PR（可选）、GitHub 页面截图（浏览器无外网）。

---

## 真实 bug / 阻塞清单

| # | 类型 | 状态 | 说明 |
|---|---|---|---|
| 1 | 网络 | 已绕过 | `git clone` 无 token 时 TLS 中断 ×3 |
| 2 | 网络 | 未解决 | Chromium 无外网，GitHub 页面截图失败；改用终端/API 证据 |
| 3 | 凭据 | 已解决 | 拿到 token 后完成账号/仓库/fork/clone/分支/commit/push |
| 4 | 环境 | 未解决 | 项目要求 Python ≥3.13，本机 3.10，未跑仓库测试 |

## 被否掉的建议

| 建议 | 否掉理由 |
|---|---|
| 关闭 SSL 校验重试 clone | 掩盖问题，且不能解决网关中断 |
| 写"技能通过复制部署" | 与 `linker.py` 代码不符 |
| 写"测试覆盖不足" | 没有覆盖率数据，属于无依据判断 |
| 用假截图占位 | 交付要求可核查，不能用占位图 |
