# neolaf2/neoskills 仓库分析

> 分析对象：`https://github.com/neolaf2/neoskills`（v0.4.2，MIT）
> 方法：下载 `main` 分支源码（2026-11-16），用 `repo_report.py` 出客观事实，再逐文件读关键模块。
> 本文所有数字都能用文末命令复算；判断都挂事实依据。

---

## 一、一句话概括

**neoskills 是一个「Homebrew 式」的 AI Agent 技能包管理器**：把技能当作可版本控制的软件包，用 git tap 管理来源、用符号链接部署到 Claude Code / OpenCode / OpenClaw，并用可选的 `ontology.yaml` 属性图管理技能之间的关系与生命周期。

它不是"技能集合"，而是"技能的包管理器 + 依赖图"。

---

## 二、客观事实

| 项 | 值 | 依据 |
|---|---|---|
| 版本 | 0.4.2 | `pyproject.toml` |
| 提交数 | 上游 `main` 40 次（2026-02-06 → 2026-03-30） | `git rev-list --count origin/main` |
| 主要作者 | neolaf2（32 次）、Richard Tong（8 次） | `git shortlog -sne` |
| 文件数 | 112（84 个 `.py`） | `repo_report.py` |
| 源码规模 | 7,531 行（`src/`） | `find src -name "*.py" \| xargs wc -l` |
| 测试规模 | 1,993 行（13 个测试模块） | `find tests -name "*.py"` |
| 许可 | MIT | `LICENSE` |
| Python | ≥3.13 | `pyproject.toml` |
| 依赖 | click / rich / pyyaml / jinja2 / gitpython / python-dotenv | 同上 |
| 技能 | 2 个（bank-status、skill-dedup） | `skills/` |
| Agent 定义 | 4 个（scanner/importer/deployer/dedup） | `agents/` |
| 斜杠命令 | 1 个（`ns`） | `commands/` |
| CI | 有（GitHub Actions） | `.github/workflows/` |

**一处已发现的文档漂移**：`CLAUDE.md` 与 `TASKS.md` 都写 "Current version: 0.4.1"，而 `pyproject.toml` 是 0.4.2。见改进建议 3。

---

## 三、目录结构

```
neoskills/
├── README.md            13 KB，安装/架构/本体/CLI 全量文档
├── CLAUDE.md            5 KB，给 AI 编码代理的项目指南（含已知问题）
├── TASKS.md             3 KB，v0.4 状态 + 7 条后续增强
├── pyproject.toml       hatchling 构建，CLI 入口 neoskills
├── skills/              2 个技能（技能本身也用它管理的格式写）
│   ├── bank-status/     SKILL.md：读注册表，报告技能库状态
│   └── skill-dedup/     SKILL.md + scripts/dedup_scan.py（800 行）
├── agents/              4 个 subagent 定义（scanner/importer/deployer/dedup）
├── commands/ns.md       斜杠命令入口
├── docs/ontology-design.md  本体层设计文档
├── src/neoskills/
│   ├── cli/             18 个 Click 命令模块（main 里懒加载）
│   ├── core/            cellar / tap / linker / checksum / resolver / manifest
│   ├── adapters/        claude / opencode / openclaw 三个适配器
│   ├── ontology/        属性图：models/graph/loader/writer/engine/taxonomy/
│   │                    lifecycle/versioning/composition/export/scaffold
│   ├── runtime/claude/  MCP 插件（把本体工具暴露给宿主 agent）
│   ├── meta/            enhancer（技能增强）
│   └── translators/     跨 agent 格式转换
└── tests/
    ├── unit/            12 个模块（ontology 占 47 个测试）
    └── integration/     e2e 工作流
```

结构上的一个特点：**它自己的技能也放在 `skills/` 里，用自己定义的格式写**。
这是"自举"（dogfooding）——`skill-dedup` 就是用来清理技能库本身的。

---

## 四、技能清单（逐个说明输入/输出/场景）

| 技能 | 输入 | 输出 | 适用场景 |
|---|---|---|---|
| **bank-status** | `~/.neoskills/registry.yaml`、`state.yaml`、`LTM/mappings/targets/` | 技能库总数、目标配置、已嵌入技能、最近 5 条导入记录 | 用户问"我现在有哪些技能 / 装到哪了" |
| **skill-dedup** | bank 与各 target 的技能目录（默认 Claude/OpenCode，可选 GitHub repo） | 三类重复分组报告（JSON 可选）+ 可选自动合并 | 技能库出现同名/同内容副本，或导入前查重 |

Agent 定义（`agents/`）不是技能，是"角色说明书"：

- **skill-scanner**：只读（Read/Grep/Glob），扫描各 agent 生态里有什么技能；
- **skill-importer**：把目标里的技能导入 bank；
- **skill-deployer**：把 bank 里的技能部署/嵌入到目标；
- **skill-dedup**：与同名技能配套的重复处理角色。

四者覆盖「发现 → 导入 → 部署 → 清理」的完整生命周期，比只给一个"万能 agent"更清楚。

---

## 五、架构分析

### 5.1 四层结构

```
CLI (Click, 18 模块, 懒加载)
   ↓
Ontology (可选属性图：节点 + 6 种边 + 生命周期状态机)
   ↓
Core (Cellar 工作区 / Tap git 仓库 / Linker 符号链接 / Checksum)
   ↓
Adapters (claude-code | opencode | openclaw 的路径与格式差异)
```

**关键设计**：本体层是可选的、增量的。
`ontology.yaml` 不存在时，技能仍然能安装、能链接；存在时才获得图能力（依赖查询、组合、生命周期）。
这降低了采用门槛——老技能不用改就能被管理。

### 5.2 部署模型：符号链接 + 单一事实源

```
~/.claude/skills/kstar-loop  -->  ~/.neoskills/taps/mySkills/skills/kstar-loop
```

- 一份技能源（在 tap 里，受 git 管理）
- 多个 agent 可同时链接
- 修改立即生效，无需复制
- `unlink` 只删链接，不删内容

`Linker` 明确不维护 `state.yaml`，所有状态从文件系统推导（`linker.py` 注释："No state.yaml — derives all state from filesystem inspection"）。
这是一个很硬的取舍：**用推导代替状态同步**，避免状态与事实不一致。

### 5.3 本体层：把"关系"从目录结构里解放出来

传统技能库靠文件夹分类，关系是隐式的。neoskills 用 `ontology.yaml` 显式声明六种边：

`requires` / `extends` / `composes_with` / `conflicts_with` / `supersedes` / `derived_from`

于是可以回答目录结构回答不了的问题：
"装这个技能会连带装什么？"（`ontology deps --transitive`）
"这两个技能冲突吗？"（`conflicts_with`）
"哪些技能依赖我？"（`rdeps --tree`）

生命周期状态机（candidate → validated → operational → refined → deprecated → archived）把"技能质量"变成可推进的状态，而不是靠 README 里写"beta"。

### 5.4 执行模式

`core/mode.py` 定义两种模式：`AGENT`（独立 CLI）与 `PLUGIN`（嵌在宿主 agent 里）。
检测顺序：环境变量 `NEOSKILLS_MODE` → 宿主标记（`CLAUDE_CODE_ENTRY`）→ 默认 AGENT。
这是"同一套代码，两种入口"的常见做法，避免维护两份。

---

## 六、最有意思的技能：skill-dedup

原因不是它代码最多（800 行），而是它**把"重复"拆成了三个不同的问题**：

| 类别 | 判定 | 处理 |
|---|---|---|
| Exact | 目录内容 SHA256 相同 | 目标副本可安全替换为指向 bank 的符号链接 |
| Diverged | 同名但内容不同 | 选"更丰富"的副本（文件更多）建议导入，人工确认 |
| Name-similar | 不同 ID 但名称/描述相似 | 只报告，人工判断 |

两个细节值得学：

1. **校验和只算"内在内容"**：`_is_intrinsic()` 排除 `metadata.yaml`、`provenance.yaml`、`__pycache__`、`.pyc`。
   否则"同一个技能"会因为生成文件不同而被判成不同，查重直接失效。
2. **自动处理必须可预演**：`--resolve exact --dry-run` 先给出 `would_symlink` 动作，再执行；执行前把目标目录移到 `.<name>.dedup-backup`。

**但它有一个真实 bug**（见改进建议 1）：脚本顶部尝试导入 `neoskills.mappings.resolver` 与 `neoskills.bank.store`，这两个模块在 v0.4 的 `src/neoskills/` 里并不存在（已改成 `core/resolver.py` 与 `core/cellar.py`）。
`ImportError` 被 `try/except` 吞掉，`NEOSKILLS_AVAILABLE` 永远为 False——这段集成代码是死的。

---

## 七、改进建议（每条附证据与改法）

### 建议 1：修掉 dedup_scan.py 的失效导入（高优先级）

**证据**：`skills/skill-dedup/scripts/dedup_scan.py:27-34`

```python
try:
    from neoskills.core.workspace import Workspace
    from neoskills.mappings.resolver import SymlinkResolver   # 模块不存在
    from neoskills.bank.store import SkillStore              # 模块不存在
    NEOSKILLS_AVAILABLE = True
except ImportError:
    NEOSKILLS_AVAILABLE = False
```

`src/neoskills/` 下没有 `mappings/` 与 `bank/`；`Workspace` 已被 `Cellar` 取代（`core/cellar.py` 注释写明 "Simplified from Workspace — no LTM/STM/bank/mappings hierarchy"）。
且 `NEOSKILLS_AVAILABLE` 在文件中只被赋值、从未被读取。

**改法**：删除这段死代码；若要复用核心逻辑，改为 `from neoskills.core.cellar import Cellar` 与 `from neoskills.core.resolver import ...`，并加一条导入冒烟测试。

### 建议 2：Windows 下的符号链接没有退路

**证据**：`core/linker.py:53` 直接 `link_path.symlink_to(source_path)`；`dedup_scan.py:466` 直接 `os.symlink(...)`。
Windows 上创建符号链接需要开发者模式或管理员权限，否则抛 `OSError`。

**改法**：启动时探测 `os.symlink` 能力；不可用时降级为「复制 + 变更检测」（或 junction），并在 `doctor` 里明确提示当前模式。README 的 Requirements 只写了 Python/Git/uv，没提这个限制。

### 建议 3：版本号仍有两处，且已经漂移

**证据**：`pyproject.toml:3` 是 `0.4.2`；`CLAUDE.md` 写 "Current version: 0.4.1"；`TASKS.md` 写 "Version: 0.4.1"。
`CLAUDE.md` 自己也承认版本存在两个地方（`pyproject.toml` 与 `__init__.py`），要求手动同步。

**改法**：单一事实源——`__init__.py` 用 `importlib.metadata.version("neoskills")` 读取；文档用脚本从包元数据生成版本行，避免第三次漂移。

### 建议 4：`ontology validate` 对 domain 边误报

**证据**：`CLAUDE.md` 的 Known Issues 明确写了：`belongs_to -> domain` 边会被报成 broken，因为 domain 与 skill 节点分开存储。

**改法**：两种都行——(a) 把 domain 也建成节点（类型 `domain`），验证器统一按节点查；(b) 把 `belongs_to` 从边改为节点字段，验证器只校验边指向技能节点。前者更符合属性图语义，后者改动小。

### 建议 5：`compose` 只有规格，没有执行器

**证据**：`TASKS.md` 第 4 条："`compose` creates a `CompositionSpec` but there's no executor that actually chains skills at runtime."
`composition.py` 只产出规格，`OntologyEngine` 没有 run 接口。

**改法**：加一个最小执行器：按 `composes_with` 边做拓扑排序 → 逐技能读取 `SKILL.md` 的 Input/Output → 检查上游输出是否满足下游输入契约 → 不满足就停在原地并给出缺口。先不做自动修复。

### 建议 6（加分）：给图加载加缓存，但要用内容哈希而不是 mtime

**证据**：`TASKS.md` 第 1 条提出缓存图，按 mtime 失效。
mtime 在 git checkout / 文件复制后会变，但内容没变；反之也可能内容变了而 mtime 被保留。

**改法**：缓存 key 用 `ontology.yaml` 与 `SKILL.md` 的相对路径 + SHA256，成本与 dedup 的 intrinsic checksum 相同，可直接复用 `_is_intrinsic` 的思路。

---

## 八、学习收获：关于 Agent 技能设计

1. **技能元数据放 sidecar，不塞进 SKILL.md。** 元数据会频繁变，正文应该稳定；分开后校验和、diff、人工阅读都更简单。
2. **渐进增强比一次性规范更容易被采用。** L0（只有 SKILL.md）就能用，L1–L3 是加分项；如果一上来要求写全图元数据，现有技能不会迁移。
3. **用文件系统当数据库，用推导代替状态文件。** 少一个 `state.yaml`，就少一类"状态与实际不一致"的 bug。
4. **符号链接是零拷贝部署的关键，但它是平台相关的。** 设计部署模型时，必须把"没有符号链接权限"当成一等公民。
5. **查重必须先定义"什么算同一个技能"。** 把生成文件排除在校验和之外，是这类工具的核心细节。
6. **重构会留下死代码，而 try/except 会把它藏起来。** 这次发现的失效导入就是例子：不报错，但功能永远不生效。

---

## 九、我没验证的部分（如实写）

1. **没有跑测试**：项目要求 Python ≥3.13，本机是 3.10；没有用 uv 建 3.13 环境。`CLAUDE.md` 声称 172 个测试全过，我没有复核。
2. **没有跑 CLI**：同理，`neoskills --help` 未执行；所有行为判断来自源码阅读与文档。
3. **git 历史**：无 token 时 `git clone` TLS 中断（`GnuTLS recv error -110`），先用 codeload zip 做结构分析；拿到 token 后完整 clone fork，补到上游 `main` 40 次提交（2026-02-06 → 2026-03-30）与两位主要作者。分析正文的结论仍以代码为准，提交历史只作时间线参考。
4. **没有验证本体图的运行时性能**：README 说"in-memory graph, O(1) faceted lookup"，我只看了索引代码，没有基准测试。

## 十、复算命令

```bash
find . -type f | wc -l                     # 112
find src -name "*.py" | xargs wc -l | tail -1   # 7531
find tests -name "*.py" | xargs wc -l | tail -1 # 1993
grep -n "version" pyproject.toml | head -1      # 0.4.2
ls skills agents commands                       # 2 / 4 / 1
```
