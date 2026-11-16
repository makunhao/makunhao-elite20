# C5A 方案设计：怎么"读懂"一个仓库

## 一、问题

C5A 要求回答四个问题：仓库做什么、目录结构、有哪些技能、哪个最有意思。
难点不是"看到"，而是**说得能被核查**——否则分析会退化成读后感。

## 二、方法：先事实，后判断

```
git clone / 下载源码
      ↓
repo_report.py（脚本）
  文件数 / 语言分布 / 源码行数 / SKILL 清单 / README 章节 / git 历史
      ↓
人读关键模块（linker / mode / ontology / dedup）
      ↓
判断（架构、改进建议）——每条挂一个文件或行号
```

**分工原则**：脚本负责"数得出来的"，人负责"需要判断的"。
不让模型直接写"这个仓库架构清晰"，因为这句话无法核查。

## 三、执行步骤

1. 获取源码。`git clone` 失败（TLS 中断）→ 改用 `codeload` zip，并在文档注明。
2. 跑 `repo_report.py`，得到 112 文件 / 84 py / 7,531 行源码 / 1,993 行测试 / 2 skills / 4 agents。
3. 读 README 的架构与本体层章节，读 `CLAUDE.md` 的 Known Issues。
4. 读 `core/linker.py`（符号链接）、`core/mode.py`（执行模式）、`skills/skill-dedup/scripts/dedup_scan.py`（查重）。
5. 写四个必答问题 + 架构分析 + 6 条改进建议。
6. 每条建议验证"仓库里确实没有"，避免把"我没看到"当成"不存在"。

## 四、改进建议的筛选标准

一条建议必须同时满足：

1. 能指出具体文件/行号；
2. 能说清"不改会怎样"；
3. 改法不超过两句话；
4. 不是"加更多测试/文档"这类通用话。

最终 6 条中，2 条是真实缺陷（失效导入、Windows 无符号链接降级），2 条来自仓库自述的 Known Issues 并给了替代方案，2 条是架构级建议（compose 执行器、图缓存）。

## 五、GitHub 操作部分（已完成）

| 步骤 | 命令 | 状态 |
|---|---|---|
| 创建个人仓库 | GitHub API | ✅ makunhao-elite20 |
| Fork neoskills | API `/repos/neolaf2/neoskills/forks` | ✅ makunhao/neoskills |
| Clone fork | `git clone`（带 token） | ✅ |
| 建分支并提交中文笔记 | `git checkout -b c5a-cn-notes` → commit → push | ✅ 69c6850 / 2814cb4 |
| 截图 | 终端 clone/分支/commit 实录 | ✅ 02_clone_branch.png |

## 六、验收标准（自定）

- 分析文档里每个数字都能用命令复算；
- 每条改进建议都能定位到文件/行；
- GitHub 操作有链接、commit hash 与终端实录。

## 七、边界

- 仓库测试未跑（需要 Python 3.13 + uv）。
- GitHub 页面截图不可得（浏览器无外网），用终端与 API 结果替代。
- PR 未提交（可选）。
