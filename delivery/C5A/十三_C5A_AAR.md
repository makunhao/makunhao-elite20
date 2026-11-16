# C5A AAR：GitHub 入门与仓库理解

## 1. 目标（R̂）

拿到 neoskills 源码 → 读懂结构 → 写一份能被人核查的仓库分析（含架构、技能清单、改进建议）→ 完成账号/仓库/fork/clone/分支的 GitHub 操作闭环。
**R̂ = 0.8**。

## 2. 过程

1. 读 C5A CHALLENGE，确认四级任务与评分。
2. 第一次 `git clone`（无 token）三次 TLS 中断 → 改用 `codeload` 下载 zip，先完成分析。
3. 用 `github-repo-analysis` 技能跑客观事实：112 文件 / 84 py / src 7,531 行 / tests 1,993 行。
4. 逐文件读 `README`、`CLAUDE.md`、`TASKS.md`、`core/linker.py`、`core/mode.py`、`skills/skill-dedup/`。
5. 写 `十三_C5A_仓库分析.md`：一句话定义、目录、技能清单、架构、6 条改进建议。
6. 拿到 token 后完成线上操作：
   - 建个人仓库 `makunhao-elite20` 并写 README；
   - fork `neolaf2/neoskills`；
   - 用 token clone fork（成功）；
   - 建分支 `c5a-cn-notes`，提交 `C5A_中文笔记.md`（2 次提交）；
   - 补全 git 历史：上游 main 40 次提交（2026-02-06 → 2026-03-30）。

## 3. 结果

| 产出 | 路径 / 链接 | 状态 |
|---|---|---|
| 仓库分析（7.5KB） | `十三_C5A_仓库分析.md` | ✅ |
| 客观事实 | `facts.txt` + `screenshots/01_terminal_facts.png` | ✅ |
| 个人仓库 | https://github.com/makunhao/makunhao-elite20 | ✅ |
| Fork | https://github.com/makunhao/neoskills | ✅ |
| 贡献分支 | https://github.com/makunhao/neoskills/tree/c5a-cn-notes | ✅ |
| Commit | 69c6850 / 2814cb4 | ✅ |
| clone/分支证据 | `screenshots/02_clone_branch.png` | ✅ |
| AI 日志 / 拿来说明 | 同目录 | ✅ |

## 4. 证据

- `facts.txt`：五条命令的原始输出（112 / 7531 / 1993 / 0.4.2 / 2-4-1）。
- `十三_C5A_仓库分析.md` 第十节：复算命令。
- 每条改进建议都带文件与行号（如 `dedup_scan.py:27-34`、`linker.py:53`）。
- 分支提交可在 GitHub 上直接查看 diff。

## 5. 失败与偏差

| 事件 | 预期 | 实际 | ΔR |
|---|---|---|---|
| 无 token 时 clone | 0.9 | 三次 TLS 中断，改用 zip | −0.5 |
| GitHub 页面截图 | 0.8 | 浏览器无外网；用终端与 API 证据替代 | −0.3 |
| 发现仓库问题 | 0.6 | 6 条可核查建议（含 2 个真实缺陷） | +0.3 |
| 线上操作闭环 | 0.7 | 全部完成（账号/仓库/fork/clone/分支/commit） | +0.2 |

综合 **R = 0.85**，ΔR = +0.05。**比预期好，主要因为拿到 token 后 clone 反而成功，补到了完整 git 历史。**

## 6. 根因

1. **clone 失败不是网速**：git 的 TLS 握手被中断，而 codeload 可用；有 token 后又成功——说明与认证/网关都有关，不能只归因于"网络差"。
2. **截图失败不是脚本错**：Chromium 无网络出口，curl 有——两个通道策略不同。改用终端输出与 GitHub API 结果作为证据，比伪造截图可信。
3. **线上操作不是难点**：拿到 token 后 10 分钟内完成；真正的难点一直是"读懂仓库"。

## 7. 改进

- 遇到 `git clone` TLS 失败：先试带 token 的 HTTPS，再换 codeload，不在重试上耗时间。
- 需要截图验收时，先验证浏览器能否访问外网；不能就用终端 + API 证据，并注明。
- 凭据类阻塞在开工时就提出来，不要等做到一半。

## 8. 可迁移经验

> **"网络可达"要按工具分别验证。** curl 通 ≠ git 通 ≠ 浏览器通；这次三个通道的结论都不一样。
