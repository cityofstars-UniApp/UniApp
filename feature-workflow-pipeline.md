# Feature Workflow 管线图 — Stage 0→9 + 常驻层（skill / workflow 逐站标注）

> **定位**：本文是 [feature-workflow.md](feature-workflow.md) 的管线视图伴生文档——把九阶段主线铺成一张从装机到终老的完整地图，每一站标注该进场的 workflow（机器）与 skill（手册）。
> **引用锚点**：行号如 `:25:45` 均指 `feature-workflow.md`；skill 镜像位于 `.agents/skills/`，workflow 镜像位于 `.github/workflows/`。全部引用经实物核实。
> **分工原则**：墙（workflows）管拦骨架，手册（skills）管教语义判断。Stage 7 是刻意设计的零 skill 纯机器区；6 次 skill 进场全部嵌在机器够不着的语义缝隙里。

## 1. 带注解总图

```
S0  装机接线(§10)：App×2 · vars×3 先全 report · 分支保护勾检查名 · fork 外部 action · 装 lefthook
      │
S1   Issue 立案 ──── W:issue-lifecycle.yml（从此挂机到仓库终老）──► Project Inbox
S2   看板分拣（人）── Inbox → Backlog / Ready / No action        【GitHub 安静】
S3   feat 分支 + Agent Note ── 决策层立案（与工单层双轨）          【GitHub 安静】
S4   实现 + 测试 ─── SK:ci-test-reliability + SK:code-review①     【GitHub 安静】
S5   本地门 ──────── W:lefthook(机器钩子) + SK:pre-push-checks    【GitHub 安静】
S6   开 PR ──────── SK:pr-description + 模板四节 ── 六门同时点亮 ▼
S7   六门机器墙 ──── W:issue-policy · pr-check-on-head · ci · weighted-approval×2 ·
                    pr-check-after-approve · pr-check-on-base     【零 skill 区】
S8   合并 + 笔记流转 ─ SK:code-review② + push master 唤醒 Recheck 门补刀
S9   终身维护 ────── SK:archive-agent-notes
常驻  issue-lifecycle 全程记事 · MERGE_POLICY_* 三档旋钮贯穿 S7
```

图例：`W:` = workflow（机器/计算层，Actions runner 里执行）；`SK:` = skill（手册/提示层，干活的人/AI 按条件加载进上下文，**没有任何 workflow 调用它们**）。SK 进场共 6 次（`dsh-code-review` 两次）；W 七根在 S7 墙上、一根（issue-lifecycle）从 S1 挂机到仓库终老、lefthook 住本地不入 GitHub。

## 2. 逐站明细

| Stage | 谁在干 / 产出 | 机器（workflows） | 手册（skills） | 关键规则 |
|---|---|---|---|---|
| **0 装机** `:166:177` | 你，一次性 | （7 根 W 合入默认分支才算通电） | — | GitHub App ×2（Issue 治理 + 审批）；secret ×1（Recheck 标签）；vars ×3 `MERGE_POLICY_MASTER_DIFF / PRE_MERGE / HEAD_DIFF` **先全 `report`**；Project #1 七态字段与 `config.json` 对齐；分支保护按**检查名**勾选；fork 外部 check action（master-diff/review/recheck）；自写 `scripts/pre-merge-check.ts` 对应物；`node scripts/install-lefthook.mjs` |
| **1 Issue 立案** `:25:45` | 你/AI 填表单 → 工单层立项 | [issue-lifecycle.yml](.github/workflows/issue-lifecycle.yml) → `policy.mjs lifecycle`（开/编/标签/关/typed/field_* 全事件，把 Issue 摆进 Project、同步 Status） | — | [ISSUE_TEMPLATE/](.github/ISSUE_TEMPLATE/feature.md) 的 feature/bug/task 表单（Motivation+Behavior=可观察验收雏形）；`rules.mjs validateIssue`：type 五选一（Idea/Feature/Bug/Research/Task）；Priority 是看板字段**不是标签**；Issue 禁用 `kind/*`（PR 专用） |
| **2 看板分拣** `:49:56` | 你，人审提案 | **无门**（咬合点在 PR 侧） | — | Inbox → Backlog（值得做不急）/ Ready（下一个做）/ No action（关掉）；`Ready` = 认可"为什么值得做"；深度提案（备选穷举）留给 S3 的 Agent Note |
| **3 分支 + 决策笔记** `:60:71` | `git switch -c feat/<topic> master` | — | —（权威是 [notes/README.md](.agents/notes/README.md)） | 深度决策（架构/安全/所有权）动笔前先写 Agent Note（骨架 + **Alternatives 强制**）；**机制与迁移分离**两个 PR（dsh 案例：机制 175 文件、迁移 104 文件分层合并）；笔记随 PR 同车进仓——工单层与决策层同一个 review |
| **4 实现 + 测试** `:75:84` | 你/AI 写码 | （GitHub 只有 lifecycle 在记事） | [dsh-ci-test-reliability](.agents/skills/dsh-ci-test-reliability/SKILL.md) `:82:84`：四层拓扑、`listen(0)`/`mkdtemp` 原子分配、全局状态捕获-恢复、Windows 平台语义、同步于状态非 sleep、dispose 到静止、negative control、拒 flake-mask 五禁<br>[dsh-code-review](.agents/skills/dsh-code-review/SKILL.md)① `:81`：**Docs match the code** | 提交单元化（refactor/fix/docs/test 分类清晰）；**笔记 AC 的每条断言有对应测试** |
| **5 本地门** `:88:95` | 钩子自动跑，人/AI 选证据 | [lefthook.yml](lefthook.yml)：pre-commit（lint `--fix` 回填 + whitespace）、pre-push（全量 typecheck） | [dsh-pre-push-checks](.agents/skills/dsh-pre-push-checks/SKILL.md) `:93:95`：包行为→聚焦用例；文档/笔记→doc-sync；UI→检查循环+GIF；构建/导出→build+冒烟 | 原则：本地快、CI 全（"CI owns the full repository-wide gate matrix"）；不重复已过的检查 |
| **6 开 PR** `:99:114` | 你/AI 填四节 → 门全亮 | （此瞬 S7 七根神经同时触发） | [dsh-pr-description](.agents/skills/dsh-pr-description/SKILL.md) `:114`（`user-invocable: false`，创建/更新 PR 描述时自动在场）：四节**每节的要求**、测试设计矩阵（行为矩阵/性能前后值/无变化证明三选一）、AI Usage 五级×五阶段评级 | 模板四节：`Fixes/Refs #N`（S7#1 咬合点）、Interface 与 Behavior 分列、Testing 条目+`<details>` Proof 折叠、ai-declaration 徽章 + 五阶段表；标签纪律（`rules.mjs validatePullRequest` 执法）：恰好一个 `kind/*`（六选一）、≥1 `area/*`、≤1 `p0-p3` 且=被解决 Issue 最高 Priority、`source/*` 仅限 Issue；UI 行为变更必附 GIF |
| **7 六门合并链** `:118:138` | **纯机器，零 skill** | #1 [issue-policy.yml](.github/workflows/issue-policy.yml) → `policy.mjs pr`（验证 Issue 挂接+标签纪律）<br>#2#3 [pr-check-on-head-branch.yml](.github/workflows/pr-check-on-head-branch.yml)：外部 action `master-diff` + merge-tree 后跑 `pre-merge-check.ts`（`Master Change` / `Pre-Merge Check` 两个 job）<br>#4 [ci.yml](.github/workflows/ci.yml) CI 矩阵（typecheck/lint/coverage/snapshot/doc-sync/build）<br>#5 [weighted-approval.yml](.github/workflows/weighted-approval.yml) + [weighted-approval-review-event.yml](.github/workflows/weighted-approval-review-event.yml) → `check-approval.mjs`（`approval-policy.json` `requiredPoints: 2`，blame 归属权重，`/delegate` 命令）<br>#6 [pr-check-after-approve.yml](.github/workflows/pr-check-after-approve.yml)：`check: review`，Reviewer Check 恒发布<br>#7 [pr-check-on-base-branch.yml](.github/workflows/pr-check-on-base-branch.yml)：push 到 master 才醒 | — | 三个 workflow 全名 `Policy`，跑 `pull_request_target`，**策略从基分支 checkout**（PR 改不动门）；`persist-credentials: false`；**检查名不可改**（分支保护按名匹配）；门持续重跑（push/打标签/评审/`/delegate`/基分支变动）；模式变量：#2→`MERGE_POLICY_MASTER_DIFF`、#3→`MERGE_POLICY_PRE_MERGE`、#6 Approve Check→`MERGE_POLICY_HEAD_DIFF` |
| **8 合并 + 笔记流转** `:142:150` | 过全链 → 合入 master | push master 唤醒 #7 Recheck 门，对全部开放 PR 重验绿灯 | [dsh-code-review](.agents/skills/dsh-code-review/SKILL.md)② `:149`：**Implemented notes match shipped reality**（笔记写的路径/名称/机制对照实现核实） | 同一提交内完成：`proposed/` → `implemented/`、`Status: implemented`、`## Proposal` → `## Decision`（现在时）、AC+Risks → `## Consequences`；格式门拒 implemented 里残留 `## Proposal`/`## Plan`/`## Migration plan`/`## Acceptance criteria`；pre-merge-commit 钩子（translation pairing + archived notes 检查） |
| **9 终身维护** `:154:162` | 你/AI，无限期 | issue-lifecycle 依旧挂机（Issue 重开、字段变动照样记事） | [dsh-archive-agent-notes](.agents/skills/dsh-archive-agent-notes/SKILL.md) `:157`：取代审计、删/留/归档分类、三元组封存规程 | 事实同步：代码挪路径/改名/改默认值 → 同一改动更新笔记事实；新笔记触发同域旧笔记审计（完全取代可整合删除，部分取代互链）；小 UI / 纯机械改动的 implemented 笔记直接删；rejected 仅"仍是诱人错误"时保留；**判语义不判字数**（校准例：collapsed sidebar 533 词删；event-sourced sessions 248 词留） |
| **常驻层** | — | issue-lifecycle.yml 全程记事；#7 Recheck 每次 S8 合并后补刀；（可选未镜像：stale-pull-requests、dependabot） | — | `MERGE_POLICY_*` 三档 `off`/`report`/`block`，**缺省=report** `:6:7`——单人仓走法：S0 先全 report，验证门真绿，再逐门 block |

## 3. 三条贯穿全程的线

1. **双轨制**（`:20:21`）：Issue = 工单层（GitHub 外部可见、进 Project 看板），Agent Note = 决策层（`.agents/notes/`，AI 治理）——从 S3 起双轨并行，PR 同时挂两者，S8 同一提交里一起流转；
2. **渐进旋钮**（`:6:7`）：每个门读 `MERGE_POLICY_*` 变量，dsh 自己上线新门也是先 `report` 观察再转 `block`——执法强度可渐进收紧，未接线时缺省 `report` 恰好什么都不拦；
3. **机器墙与手册的分工**：S7 是零 skill 的纯机器区；6 次 skill 进场（S4×2、S5、S6、S8、S9）全部嵌在机器够不着的语义缝隙里——**墙管拦骨架，手册管教判断**，两套缺一不可：缺墙则拦不住合并，缺手册则红绿灯全绿而质量裸奔。

## 4. 一句话收束

**Stage 0 装一次线，S1–S3 立两层案，S4–S6 人机协作备货，S7 机器墙裁决，S8–S9 人机协作收尾**——5 本手册 6 次进场、7 根神经在墙上、1 根神经从 S1 挂机到仓库终老。
