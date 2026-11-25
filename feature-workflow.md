# Feature Workflow — dsh 原版完整流程（GitHub Actions 式）

> **定位**：本文件描述与 [deepseek-harness](../Harness/reference-repos/deepseek-harness) **完全相同**的流程，不裁剪。
> 本仓根 `C:\Projects\UniApp\` = dsh 同构仓根布局（`.agents/` 治理目录 + `.github/` 门/workflow 目录）；
> GitHub 侧机制已全部实体拷贝到 `.github/`，原版链接一律指向 `../Harness/reference-repos/deepseek-harness/`。
> **执法强度由仓库变量渐进控制**：每个门读 `MERGE_POLICY_*` 变量，`off`（跳过）/ `report`（跑但不拦截）/ `block`（拦截），
> 缺省或未知值一律按 `report`——dsh 自己上线新门就是先 `report` 观察再转 `block`，单人仓库同样适用：先全链 `report`，验证门真的绿，再逐门 `block`。

---

## 0. 流程总图

```
Issue(模板+标签) ─► Project 看板分拣 ─► feat 分支 ─► 实现+测试 ─► 本地门(lefthook)
 ─► PR(四节模板+AI声明) ─► 六门链 ─► 合并 ─► 笔记流转(implemented) ─► 终身维护
        ▲                        │
        └── Issue policy 咬合 ◄──┘（PR 必须 Fixes/Refs 合法 Issue）
```

双轨制：**Issue = 工单层**（GitHub 外部可见，进 Project 看板）；**Agent Note = 决策层**（`.agents/notes/`，AI 治理）。
两层并存，PR 同时挂着两者——dsh 案例实证（[2026-11-15-client-config-sync](../Harness/reference-repos/deepseek-harness/.agents/notes/implemented/architecture/2026-11-15-client-config-sync.md)：Issue 起头，笔记三件套随 PR 进仓）。

---

## Stage 1 — Issue 立案

**你要做什么**：用表单开 Issue，打标签；Issue 自动进 Project 看板 `Inbox`。

**过程细节**：

1. **入口表单**（沙盘 [.github/ISSUE_TEMPLATE/](.github/ISSUE_TEMPLATE/feature.md) · [原件](../Harness/reference-repos/deepseek-harness/.github/ISSUE_TEMPLATE/feature.md)）：
   `feature.md`（Motivation + Behavior 两节）/ `bug.md` / `task.md`，由 `config.yml` 控制表单集。
   Behavior 节 = 可观察验收的雏形。
2. **Issue 侧纪律**（[rules.mjs](.github/issue-management/rules.mjs) 的 `validateIssue`（镜像））：
   原生 type 五选一（Idea/Feature/Bug/Research/Task，表单 frontmatter 自动挂）；
   必须进 Project 看板并有合法 Status；Priority 是看板字段（空或 P0–P3），**不是标签**；
   **Issue 禁用 `kind/*` 与旧版标签**——`kind/*` 是 PR 专用（见 Stage 6）。
3. **看板**（[issue-management/config.json](.github/issue-management/config.json)（镜像））：GitHub Project #1，
   Status 字段七态 `Inbox / Backlog / Ready / In progress / In review / Done / No action`，
   另有 `Priority` 与 `Start Date` 字段（时区 Asia/Shanghai）。

**门（自动化）**：[issue-lifecycle.yml](.github/workflows/issue-lifecycle.yml)（镜像）监听 Issue 的
opened/edited/labeled/unlabeled/closed/typed/field_* 全部事件 + PR 事件，整个 job 只调
`node .github/issue-management/policy.mjs lifecycle`——把 Issue 摆进 Project、同步 Status。
接线需要 Issue 治理 GitHub App（见 §10）。

---

## Stage 2 — 看板分拣（人审提案）

**你要做什么**：在 Project 看板上把 `Inbox` 里的 Issue 分拣——`Backlog`（值得做但不急）/ `Ready`（下一个做）/ `No action`（关掉）。

**过程细节**：这就是原版的"审提案"环节——feature.md 的 Motivation+Behavior 就是提案本体；
`Ready` = 你认可了"为什么值得做"。决策层的深度提案（备选穷举）在动笔时写 Agent Note（Stage 3）。

**门**：本阶段无自动门。自动化咬合点在 PR 侧（Issue policy 门验证 PR 是否挂上了合法 Issue）。

---

## Stage 3 — 实现分支 + 决策笔记

**你要做什么**：

```sh
git switch -c feat/<topic> master
```

- 深度决策（架构/安全/所有权边界）动笔前先写 Agent Note（骨架与 Alternatives 强制规则见
  [.agents/notes/README.md](.agents/notes/README.md)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/.agents/notes/README.md)），
  笔记随 PR 同车进仓——工单层与决策层在同一个 review 里。
- **机制与迁移分离**：架构落地与存量迁移分两个 PR（dsh 案例：机制 175 文件、迁移 104 文件分层合并）。

---

## Stage 4 — 实现 + 测试

**过程细节**（与前版一致，权威文件已镜像）：

- 提交单元化：`refactor/fix/docs/test` 分类清晰，每个提交一个可复核单元；
- 笔记 AC 的每条断言有对应测试；
- 文档同改动（[code-review 阻断要求 "Docs match the code"](.agents/skills/dsh-code-review/SKILL.md)）；
- 测试按 [.agents/skills/dsh-ci-test-reliability/SKILL.md](.agents/skills/dsh-ci-test-reliability/SKILL.md)（镜像）：
  四层拓扑假设、`listen(0)`/`mkdtemp` 原子分配、全局状态捕获-恢复、Windows 平台语义、
  同步于状态而非 sleep、dispose 到静止、negative control、拒绝 flake-mask 修法。

---

## Stage 5 — 本地门（lefthook）

**钩子自动**（[lefthook.yml](lefthook.yml)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/lefthook.yml)）：
pre-commit：lint `--fix` 回填 + whitespace 检查；pre-push：全量 typecheck。

**人工选证据**：按变更面选最窄证据（[.agents/skills/dsh-pre-push-checks/SKILL.md](.agents/skills/dsh-pre-push-checks/SKILL.md)（镜像））：
包行为→拥有该行为的聚焦用例；文档/笔记→doc-sync；UI→检查循环+GIF；构建/导出→build+冒烟。
原则：本地快、CI 全（lefthook 头注释原话 "CI owns the full repository-wide gate matrix"）。

---

## Stage 6 — PR（模板制度化）

**你要做什么**：开 PR，按 [pull_request_template.md](.github/pull_request_template.md)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/.github/pull_request_template.md) 四节填写——

| 节 | 硬要求 |
|---|---|
| Motivation | `Fixes #N` 或 `Refs #N` 引用 Issue——Issue policy 门咬合点 |
| Changes | Interface 与 Behavior 分列 |
| Testing | 每条测试一个条目 + `<details><summary>Proof</summary>` 折叠证据 |
| AI Usage | [ai-declaration](https://ai-declaration.md) 徽章（v0.1.2）+ 五阶段表：breakdown/design/testing/implementation/review 各标级别与参与度 % |

**PR 标签纪律**（[issue-management/README.md](.github/issue-management/README.md)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/.github/issue-management/README.md)，由 `rules.mjs` 的 `validatePullRequest` 执法）：
恰好一个 `kind/*`（六选一：`kind/feature` / `kind/bug-fix` / `kind/doc` / `kind/testing` / `kind/cleanup` / `kind/dependency`；旧别名如 `kind/bug`、`feature` 已退役封存，重建即红）、
至少一个 `area/*`、至多一个 `p0-p3`，且 **PR Priority 必须等于被解决 Issue 的最高 Priority**（被解决 Issue 全部没设 Priority 时 PR 可省）。`source/*` 仅限 Issue，PR 挂了即红。

格式权威：[.agents/skills/dsh-pr-description/SKILL.md](.agents/skills/dsh-pr-description/SKILL.md)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/.agents/skills/dsh-pr-description/SKILL.md)。UI 行为变更必附 GIF。

---

## Stage 7 — 六门合并链（核心）

三个 workflow **全部命名为 `Policy`**，跑在 `pull_request_target` 上（策略定义取自**基分支**，PR 改不掉门），
覆盖指向 master 或 `release/x.y.z` 的 PR。**分支保护规则按检查名匹配——workflow 名与 job 名不可改**
（沙盘 [.github/AGENTS.md](.github/AGENTS.md)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/.github/AGENTS.md) 是本节权威）。

| # | 检查名（分支保护要求） | workflow（沙盘镜像） | 做什么 | 模式变量 |
|---|---|---|---|---|
| 1 | `Issue policy` | [issue-policy.yml](.github/workflows/issue-policy.yml) | PR 事件触发，**从默认分支 checkout 可信策略** → selective-preflight 预检 → Issue 治理 App 令牌 → `policy.mjs pr`：验证 `Fixes/Refs` 挂接、恰好一个 `kind/*`、≥1 `area/*`、≤1 `p0-p3` 且等于被解决 Issue 的最高 Priority | 无（恒执法） |
| 2 | `Policy / Master Change` | [pr-check-on-head-branch.yml](.github/workflows/pr-check-on-head-branch.yml) | 外部 action `deepseek-harness/deepseek-harness-check@master`，`check: master-diff`，输出 `base_sha`/`since` | `MERGE_POLICY_MASTER_DIFF` |
| 3 | `Policy / Pre-Merge Check` | 同上 `pre-merge` job（needs Master Change） | runner 上 `git merge-tree --write-tree` 把 base 合入 head，再跑仓根 `scripts/pre-merge-check.ts`：发布清单对根版本、**拒绝错位目录的升级指南**、构建 Host/Client 契约 | `MERGE_POLICY_PRE_MERGE` |
| 4 | `all checks passed` | [ci.yml](.github/workflows/ci.yml)（镜像）· [原件](../Harness/reference-repos/deepseek-harness/.github/workflows/ci.yml) | CI 矩阵：typecheck/lint/coverage/snapshot/doc-sync/build（dsh 另有 `ci-windows.yml`/`ci-python.yml` 喂 `windows/python checks passed`） | — |
| 5 | Weighted approval（发布为独立 check） | [weighted-approval.yml](.github/workflows/weighted-approval.yml) + [weighted-approval-review-event.yml](.github/workflows/weighted-approval-review-event.yml) | 从默认分支 checkout 可信审批策略 → Python 装**生产 lexer**（`requirements.txt`）→ `review-ownership/check-approval.mjs`：按 [approval-policy.json](.github/review-ownership/approval-policy.json) 累计审批分，`requiredPoints: 2`、指定 reviewer 各 2 分、默认 1 分（作者分封顶 1.1，见 [review-ownership/README.md](.github/review-ownership/README.md)（镜像））；支持 `/delegate` 评论命令；blame 归属权重由 `blame-ownership.mjs` + `blame-production.py` 计算 | — |
| 6 | `Policy / Approve Check` + `Policy / Reviewer Check` | [pr-check-after-approve.yml](.github/workflows/pr-check-after-approve.yml) | PR 更新 + review 提交/撤销时触发，外部 action `check: review`；Reviewer Check 以独立 App 身份发布：每个需审文件都有 review → `success`；否则 `action_required` 并列出该找谁 | `MERGE_POLICY_HEAD_DIFF`（只控制 Approve Check；Reviewer Check 恒发布） |
| 7 | `Policy / Recheck` | [pr-check-on-base-branch.yml](.github/workflows/pr-check-on-base-branch.yml) | **push 到 master** 时触发 `check: recheck`：基分支变动后对全部开放 PR 重跑头分支检查 | 跟随 Master Diff / Pre-Merge 的模式 |

**安全架构**（照抄，勿改）：`pull_request_target` + 策略代码从默认分支 checkout + `persist-credentials: false`；
头分支 checkout 用稀疏模式只取 `README.md`（YAML 注释原话："needs no files and runs no code"）；
合并检查逻辑在外部 action 仓 `deepseek-harness/deepseek-harness-check`，fork 或自实现 `master-diff/review/recheck` 三个 check。

**失败语义**：`report` 模式下合不了 base / 构建失败 → 警告并跳过；`block` 模式 → 直接红。

---

## Stage 8 — 合并 + 笔记流转

**同一提交内完成**（规则：[notes/README 的 Moving between lifecycles](.agents/notes/README.md)）：

1. PR 过全链后合入 master；
2. 笔记 `proposed/` → `implemented/`：`Status: implemented`、`## Proposal` → `## Decision`（现在时）、
   AC+Risks → `## Consequences`；格式门拒绝 implemented 里残留 `## Proposal`/`## Plan`/`## Migration plan`/`## Acceptance criteria`；
3. 对照实现核实路径/名称/机制（[dsh-code-review](.agents/skills/dsh-code-review/SKILL.md)：Implemented notes match shipped reality）；
4. dsh 本地还有 pre-merge-commit 钩子位（[lefthook.yml](../Harness/reference-repos/deepseek-harness/lefthook.yml)：translation pairing + archived notes 检查）。

---

## Stage 9 — 终身维护

与前版一致（[.agents/notes/README.md](.agents/notes/README.md) Archiving and deletion 节 +
[.agents/skills/dsh-archive-agent-notes/SKILL.md](.agents/skills/dsh-archive-agent-notes/SKILL.md)（镜像））：

- **事实同步**：代码挪路径/改名/改默认值 → 同一改动更新笔记事实（[implemented/AGENTS.md](../Harness/reference-repos/deepseek-harness/.agents/notes/implemented/AGENTS.md)）；
- **取代**：新笔记触发同域旧笔记审计——完全取代可整合删除（保全独特理由，修复入链），部分取代互链；
- **删除**：只描述小 UI / 纯机械改动的 implemented 笔记直接删；rejected 只在"仍是诱人错误"时保留；
- 判语义不判字数（校准例：collapsed sidebar 533 词删；event-sourced sessions 248 词留）。

---

## 10. 上线接线清单（Actions 式部署必做）

| 项 | 内容 |
|---|---|
| GitHub App ×2 | Issue 治理 App（`issues:read`、`org-projects:read`）+ 审批 App（`checks:write`）；对应 vars `DSH_ISSUE_APP_CLIENT_ID`、`DSH_APPROVAL_APP_CLIENT_ID`、`DSH_APPROVAL_APP_SLUG`；secrets `DSH_ISSUE_APP_PRIVATE_KEY`、`DSH_APPROVAL_APP_PRIVATE_KEY` |
| Secret ×1 | `DSH_RECHECK_LABEL_TOKEN`（Recheck 门打标签用） |
| Repo variables ×3 | `MERGE_POLICY_MASTER_DIFF` / `MERGE_POLICY_PRE_MERGE` / `MERGE_POLICY_HEAD_DIFF`——**先全 `report`，观察门真的绿，再逐门 `block`**；缺省=report |
| GitHub Project | Project #1，Status 七态 + Priority + Start Date 字段，与 [config.json](.github/issue-management/config.json) 对齐 |
| 分支保护 | master：require PR；required checks 按检查名勾选：`Issue policy`、`Policy / Master Change`、`Policy / Pre-Merge Check`、`all checks passed`、weighted approval 状态、`Policy / Approve Check`、`Policy / Reviewer Check` |
| 外部 action | `deepseek-harness/deepseek-harness-check`（master-diff / review / recheck 三个 check 的本体）——fork 该仓或自实现 |
| 仓根脚本 | `scripts/pre-merge-check.ts` 必须存在（dsh 版查发布清单/升级指南/契约；换项目自写对应物） |
| 本地 | `node scripts/install-lefthook.mjs` 装 [lefthook.yml](lefthook.yml) |

## 11. 沙盘镜像清单（本轮新增 37 文件）

| 沙盘位置 | 内容 | dsh 原件根 |
|---|---|---|
| `.github/AGENTS.md` | .github 治理总说明（本文件 §7 权威） | [.github/AGENTS.md](../Harness/reference-repos/deepseek-harness/.github/AGENTS.md) |
| `.github/pull_request_template.md` | PR 四节模板 | [.github/pull_request_template.md](../Harness/reference-repos/deepseek-harness/.github/pull_request_template.md) |
| `.github/ISSUE_TEMPLATE/`（4 文件） | feature/bug/task/config 表单 | [.github/ISSUE_TEMPLATE/](../Harness/reference-repos/deepseek-harness/.github/ISSUE_TEMPLATE/feature.md) |
| `.github/issue-management/`（11 文件） | 门脚本目录：policy.mjs + rules.mjs + lifecycle.mjs + github.mjs + pull-request.mjs + 测试 + config.json + selective-preflight.json + 双语 README | [.github/issue-management/](../Harness/reference-repos/deepseek-harness/.github/issue-management/README.md) |
| `.github/review-ownership/`（12 文件） | 门脚本目录：check-approval.mjs + author-weight.mjs + blame-ownership.mjs + blame-production.py + 测试 + approval-policy.json | [.github/review-ownership/](../Harness/reference-repos/deepseek-harness/.github/review-ownership/README.md) |
| `.github/workflows/`（+7） | issue-policy / issue-lifecycle / pr-check-on-head-branch / pr-check-after-approve / pr-check-on-base-branch / weighted-approval / weighted-approval-review-event | [.github/workflows/](../Harness/reference-repos/deepseek-harness/.github/workflows/issue-policy.yml) |
| 此前已有 | lefthook.yml、ci.yml、notes/README.md、5 个 skill | — |

## 12. 已知现实差距（照实记录）

1. **`requiredPoints: 2` 是双人制**：作者自审分封顶 1.1，单人仓库永远凑不齐 2 分。三个选项：
   (a) [approval-policy.json](.github/review-ownership/approval-policy.json) 改 `requiredPoints: 1`；
   (b) 第二个账号（或 bot）当指定 reviewer；
   (c) 该门保持 `report` 模式。配置一行可改，流程其余不动。
2. **两个依赖是产品相关的**：外部 check action 仓与 `scripts/pre-merge-check.ts`——需 fork / 自写对应物，其余门逻辑（`.mjs`）可直接复用。
3. **沙盘文件是设计参考不是即插即用**：`ci.yml`/`lefthook.yml` 引用的 pnpm/脚本工具链在沙盘不存在；`owner: deepseek-harness` 与 App vars 名按你的仓库名替换。
4. **公开仓是镜像展柜，不是流程现场**：`github.com/deepseek-ai/deepseek-harness` 在仓库设置中禁用了 Issues 与 Pull requests（页面无这两个标签，closed issues = 0），社区入口只有 Discussions（CONTRIBUTING 明言暂不接受外部 PR）。提交历史里 `Merge pull request #8736` 等编号指向私有开发仓里的 PR/Issue——六门链、Project 看板、两个 App 都在那边运转。**同理：你的仓里 Issue 由你自己创建、编号从 #1 开始**；门的 `report` 缺省恰好保证未接线时不拦任何东西（发布镜像安然无恙就是明证）。
