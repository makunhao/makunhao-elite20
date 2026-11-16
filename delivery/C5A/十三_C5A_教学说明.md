# C5A 教学说明：30 分钟读懂一个仓库

## 一、准备

- 能联网的机器（GitHub 有时会 TLS 中断；备选 `codeload.github.com` zip）
- Python 3（跑 `repo_report.py`）
- 目标仓库：https://github.com/neolaf2/neoskills

## 二、步骤

```bash
# 1) 获取源码
git clone https://github.com/neolaf2/neoskills.git
# 若失败：
curl -L -o neoskills.zip https://codeload.github.com/neolaf2/neoskills/zip/refs/heads/main
unzip neoskills.zip && cd neoskills-main

# 2) 出客观事实
python3 <skill>/repo_report.py . --out facts.md

# 3) 按四个问题读
#    - 一句话：它是什么？
#    - 目录：主要文件夹各自负责什么？
#    - 技能：skills/ 下每个技能做什么？
#    - 哪个最有意思：为什么？（要有证据）
```

## 三、怎么写"有意思"

不要写"我觉得 XX 很酷"。用这个句式：

> **它有意思，是因为它把 ____ 拆成了 ____，而大多数工具只做 ____。**
> 证据：`文件:行号`。

例：skill-dedup 把"重复"拆成 exact / diverged / name-similar 三类，而大多数查重工具只比哈希。

## 四、怎么提改进建议

一条合格的建议 = 位置 + 后果 + 改法（≤2 句）。

不合格：「建议增加测试」「文档可以更完善」。
合格：「`dedup_scan.py:27` 导入不存在的模块，导致集成代码永远不生效；改为 `core/cellar.py` 并加导入冒烟测试。」

## 五、GitHub 操作清单（有 token 时）

1. 建个人仓库 `<username>-elite20`，README 写姓名/专业/一句话。
2. Fork `neolaf2/neoskills`。
3. `git clone` 自己的 fork。
4. `git checkout -b c5a-cn-notes`，加一份中文笔记。
5. `git add / commit / push`，截图 commit 历史。
6. 回填 `十三_C5A_GitHub账号.txt`。

## 六、常见问题

| 现象 | 处理 |
|---|---|
| `git clone` TLS 中断 | 换 codeload zip；或在有代理的网络重试 |
| 浏览器打不开 GitHub | 截图改用本地 README 渲染，并注明 |
| 不知道技能从哪看 | 先 `find . -name SKILL.md` |
| 建议写不出来 | 先读 `CLAUDE.md` 的 Known Issues 和 `TASKS.md` 的 Future |
