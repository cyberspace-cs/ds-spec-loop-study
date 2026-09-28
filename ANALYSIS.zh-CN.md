# ds-spec-loop 深度分析

> 分析对象：[songyang0603/ds-spec-loop](https://github.com/songyang0603/ds-spec-loop)
> 上游快照见 `upstream/`（保留原 LICENSE 与 NOTICE）。
> 本文是个人学习笔记，不代表上游作者观点。

## 1. 它到底是什么

不是框架、不是库、不是 CLI 工具。它就是一份 **给 coding agent 看的 prompt 包**：

```
skills/ds-spec-loop/
├── SKILL.md                    # ~200 行主指令，agent 每次必读
├── agents/openai.yaml          # Codex 的 UI 元数据
└── references/                # 9 个按需加载的细节协议
    ├── system-of-authority.md
    ├── decision-record-lifecycle.md
    ├── requirement-change.md
    ├── decision-classes.md
    ├── acceptance-and-evidence.md
    ├── simplification.md
    ├── documentation-discipline.md
    ├── adoption.md
    └── templates.md
```

作者 songyang0603 通过阅读 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的公开仓库和 Git 历史，反推出 DeepSeek 工程师和 agent 协作时实际在用的"Spec 编程"纪律，抽象成一份与 DeepSeek 无关、可移植的通用 skill。支持 Codex、Claude Code、GitHub Copilot 三家 agent 宿主。

版本：v0.1 2026-08-15，v0.2 2026-08-17。很新，作者自己说这是个人研究项目，不代表 DeepSeek 官方。

## 2. 一句话核心

> **Spec 编程 ≠ 先写一份长计划。而是把仓库本身当作一个"分布式权威系统"，让六类东西在一次 bounded change（PR / 提交序列）里保持一致：**

1. 仓库指令（`AGENTS.md` / `CLAUDE.md` / Copilot instructions）
2. 工作中的 proposal 与已定 decision record（ADR）
3. 当前文档与公共契约
4. 源码、类型、配置、包拓扑、生成产物
5. 可执行证据：单测 / 集成 / replay / snapshot / e2e
6. CI / 仓库脚本里能机器判定的事实

它最硬的一条规则：

> **Every non-mechanical change creates or updates ONE owning decision record.**
> （每个非机械变更，都必须创建或更新一条"对该变更负责"的 decision record。）

豁免条件写得很死：只有"纯机械、局部、不改变行为/契约/结构/流程/测试策略/持久化格式/决策依据"的改动（改错别字、格式化）才豁免。**判断标准不是改动大小、文件数、花了多久，而是上面那七类东西有没有变。**

## 3. 它在解决 coding agent 的哪些真痛点

现在让 Claude Code / Codex / Copilot 改一个中型仓库，最常见的翻车现场：

| 翻车现场 | ds-spec-loop 怎么堵 |
|---|---|
| 改了代码忘了改文档 | "changed authority surfaces converge in one bounded change"——代码、测试、当前文档、owning decision 必须同一次变更里更新 |
| 写完说"done"但没跑测试 | 每条 acceptance 必须绑定一个"能证伪它的直接证据"+ 仓库里真实可跑的命令 |
| 需求改了，旧 proposal 留在仓库里变成僵尸文档 | 三态生命周期：Working / Current / Declined；未交付的 proposal 直接改，已交付的必须新建交叉链接的替换记录，不许假装旧决策没发生过 |
| 一次 PR 顺手重构八竿子打不着的东西 | "Keep the patch scoped to the decision. Record unrelated discoveries instead of folding them in." |
| agent 为了"看起来严谨"自己加一堆 validator / benchmark / snapshot 框架 | "Use the smallest direct evidence set... Do not add a validator merely to make the change look rigorous." |
| `accepted` ADR 不等于真的上线了 | 明确说：一个 accepted ADR 不代表代码已 shipped；只有实现 + 直接证据 + 当前文档三者一致，才叫 Current |
| 静态检查过了就说行为正确 | "Static source inspection is evidence about source, not proof of runtime behavior." |

## 4. 几个值得单独拿出来讲的设计

### 4.1 证据要匹配失败面

它不让 agent 一刀切地"补单测"，而是按你改的东西在哪一层失败，选对应的证据类型：

- 本地逻辑 / 不变量 → 聚焦单测
- 包、服务、loader、运行时组装 → 集成 / 组装测试
- 持久化、恢复、事件重建 → replay / resume 测试
- 用户或模型可见行为 → 真实跑一遍产品路径，看稳定输出
- 外部服务 → 真 e2e；拿不到就明确标注"未验证边界"
- 删代码 → 反向搜索 + 确认从源码、导出、注册、manifest、文档、测试、持久化格式里都消失了
- 依赖方向 → manifest / import / 拓扑检查

这一段几乎是给所有"agent 写测试"场景写的操作手册。

### 4.2 需求变更协议（requirement-change）

这是 v0.2 重点补强的部分，区分得非常干净：

- **同一个还没交付的 PR 里改方向**：直接改活的 proposal，不要搞替换链。
- **已经 shipped 的决策要反转**：新建一条 decision，和旧的交叉链接；**绝不要重写历史假装旧决策不存在**。
- **只替换了旧决策的一部分**：两条 decision 都保持 Current，写清楚各自管哪一条款。
- **合并旧决策**：必须等新决策把旧决策里独特的理由、后果、验证要求、重引入条件都接过来，才能归档。

这套语义比大多数团队自己的变更管理文档都清楚。

### 4.3 父代/owner 复用优先

动手前必须先做一件事：**找到已经拥有这个决策的那条记录，更新它，而不是平行再造一条。** 只有当问题真的是"独立可重新讨论、有自己的 alternatives 和后果"时，才允许新建。改一个文件、一个模块、一个实现阶段、一个测试层，都自动不算一条新 decision。

### 4.4 不预设目录结构

skill 不规定你必须建 `docs/adr/` 还是 `docs/decisions/`。它先去读仓库里已有的 `AGENTS.md` / `CLAUDE.md` / RFC / ADR / design doc，**复用宿主仓库的命名和生命周期词**。宿主没有约定时，才给一个最小兜底：仓库指令 + 当前架构文档 + 一个 `docs/decisions/` 目录。明确反对"为了符合 skill 而在已有 ADR 体系的仓库里再造一棵决策树"。

## 5. 它自己就在吃自己的狗粮

看 `upstream/` 目录结构就知道：

- 根目录有 `AGENTS.md`，列清了每个文件的 authority（SKILL.md 管 agent 核心循环、`docs/decisions/` 管决策理由、NOTICE 管署名……）
- `docs/decisions/2026-08-17-generalize-the-strict-spec-loop.md` 是一条真实的 accepted ADR，Problem / Decision / Alternatives considered / Consequences / Verification 五段齐全
- 那条 ADR 的 Verification 一节，列了 12 条它自己用来验证 v0.2 改动的证据（quick_validate.py、YAML 解析、git diff --check、四种语言的 forward test、独立 review、甚至包括"确认没有 push 错 public main、没有误改用户本地已安装的 skill 副本"）

这是这份 skill 最有说服力的部分：**它不是纸上谈兵，作者真的在自己的小项目里跑了一遍这套 loop。**

## 6. 冷静看：它的边界和代价

不要神化。

- **它解决的是"一致性纪律"，不是"判断力"**。它能保证 agent 不把代码/测试/文档改散架，但它不能告诉 agent 这个功能该不该做、这个方案对不对。和我们上周看的 Frontis-MA1 那篇结论是同一枚硬币的两面：给 agent 一个有分数的目标，它搜得很好；让它自己决定目标，它不行。ds-spec-loop 属于前者。
- **对单人小项目可能过重**。一次改个 typo 它豁免，但任何稍微动到行为的 PR 都要走 decision record，对一个几百行的 side project 是仪式感负担。它的甜区是：中型以上仓库、多人协作、或者 agent 频繁介入需要强约束的项目。
- **它不替你写测试，也不替你跑测试**。它只是逼 agent 声明"我用哪条命令、哪条证据来证明这件事做完了"。仓库里要是没有测试基建，它也变不出来。
- **"accepted ≠ shipped" 这个区分听起来简单，但实际用的时候 agent 很容易偷懒只改状态字段**。v0.2 那条 ADR 里专门提到一次 forward test 第一次跑就踩了这个坑（只改了 Status: accepted 而没把 proposal 正文改写成过去时的稳定决策），后来才把规则收紧。这说明它的规则需要严格执行，agent 自己会找捷径。
- **协议归属**：上游是独立社区项目，非 DeepSeek 官方，且明确"不修改 ../deepseek-harness"。商用前看一眼 `upstream/LICENSE` 和 `upstream/NOTICE`。

## 7. 怎么用

### 直接装（最快）

```bash
npx skills add songyang0603/ds-spec-loop
# 或
gh skill install songyang0603/ds-spec-loop ds-spec-loop --agent codex --scope user
```

然后在 Codex 里 `$ds-spec-loop`，Claude Code / Copilot 里 `/ds-spec-loop`。

三种典型调用方式（README 里给的）：

- **实现一个变更**："Use $ds-spec-loop to implement this change. Reuse the repository's existing decision, documentation, test, and CI conventions."
- **只写提案不动手**："Use $ds-spec-loop to investigate this change and write its proposal. Do not implement it."
- **只做审查不改文件**："Use $ds-spec-loop to check whether proposals, decisions, code, tests, and docs agree. Do not modify files."

### 不装整个包，只偷几条规则

如果你不想在自己的项目里引入完整 skill，单把下面这几条抄进你的 `AGENTS.md` / `CLAUDE.md` 就已经能挡掉大半翻车：

1. 任何改行为/契约/持久化格式的 PR，必须同一次变更里更新对应代码、测试、当前文档，不允许"文档以后补"。
2. 每条 acceptance 必须配一条能在仓库里直接跑的命令和能证伪它的观察。
3. 改需求时，未交付的提案直接改；已交付的决策新建一条交叉链接的 decision，不重写历史。
4. 不要在一次 PR 里顺手重构无关代码；发现了就另开一条 decision 记录。
5. "测试通过"不等于"功能完成"——要从调用方、用户、运维、未来维护者四个视角再过一遍。

## 8. 对 Kaggle / 研究代码仓库的特别适配建议

（结合本仓库所有者的场景：在职研究员，用 GPT/Claude 打 Kaggle。）

Kaggle 比赛代码仓库最常见的烂账是：notebook 里东一块西一块的特征工程、改过的 EDA 脚本没注释、最后提交的版本和脑子里想的对不上。可以把这套 loop 裁剪一下：

- 把"decision record"降级成每个实验一个短 markdown：`exp_<日期>_<想法>.md`，三行就够——试了什么、为什么、LB 分数是多少、能不能进 ensemble。
- 强制规则：**任何进 final ensemble 的模型，必须在同一次 commit 里更新 ensemble 配置 + 推理脚本 + 一句"为什么信任这个模型"**。这能防掉 80% 的"我以为这个模型在 ensemble 里其实没加"类事故。
- Debug 一个 LB 掉点的 case 时，走一次"acceptance → direct evidence"：先写"我猜是 leakage / 是 distribution shift / 是 seed 不稳定"，再写用哪条命令证伪，不要直接上手改。

完整套用到 Kaggle 会太重，但是上面这三条裁出来的规则，对任何一个用 agent 辅助写比赛代码的人都立竿见影。

---

## 仓库结构

```
.
├── README.md              # 本仓库说明
├── ANALYSIS.zh-CN.md      # 你正在读的这份分析
└── upstream/              # songyang0603/ds-spec-loop 的完整快照（2026-09-28 拉取，保留原 LICENSE/NOTICE）
```
