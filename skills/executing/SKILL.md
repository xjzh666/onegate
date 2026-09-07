---
name: executing
description: Use when a spec and task list have been approved and handed over for implementation
---

# 执行：逐任务循环

spec 已批准 = 你已拿到移交。**围栏之内全自主**：写码、测试、提交、评审、裁决，不停下请示。围栏之外零动作——发现了只汇报。

**核心机制**：just-in-time brief（接口从已完成任务的实际代码抄，不从想象抄）+ 强模型实现者（自担 TDD）+ 每任务一道新鲜评审 + 台账。

## 准备

1. **工作区**（执行默认启用 worktree 时）：
   - 已在隔离区（`GIT_DIR != GIT_COMMON` 且非子模块）→ 直接用，不再建
   - 否则 `git worktree add .worktrees/<分支名> -b <分支名>` 并切入
   - 开工前确认 `.worktrees/` 和 `.agent/` 已在 `.gitignore`，没有就补上并提交
   - 装依赖 → 跑基线测试。失败 → 汇报，问用户是否继续
2. **台账与接手文档**：
   - 台账 `.agent/progress.md`，第一行写身份 `# ledger — spec: <spec 路径>`
     - 会话压缩丢记忆，台账和 git log 不丢。恢复时信它们，不信自己的回忆
     - 每任务一行完成记录，格式固定；恢复执行时，有完成行的任务不重派
   - 接手文档 `.agent/handover.md`（模板见下文"接手文档"一节）——写给换会话后的接班控制者。恢复执行的新会话：先读接手文档恢复语境，再按台账定位从哪个任务继续
3. 读 spec + 任务清单，每任务一个 todo。清单顺序即执行序；写 brief 时发现顺序错了（依赖倒置）→ 调整顺序，记台账一行，继续——这是控制者权限内的小事

## 任务循环（串行，一个接一个）

### 1. 写 brief（just-in-time，模板见 [brief-template.md](brief-template.md)）

五项，缺一不可：
1. **行为需求 + 可判定验收标准**，外加 1-2 个输入→输出 oracle 示例
2. **接口事实**：消费的真实签名（从仓库现状抄）；产出的名字和签名
3. **范围围栏**：许动文件、禁动文件、全局约束逐字复制
4. **验证契约**：TDD——先写失败测试看它失败，再实现；报告附 RED/GREEN 证据
5. **报告契约**：状态 + 提交 SHA + 一行测试摘要 + 顾虑；细节写报告文件，不进对话

brief 存 `.agent/task-N-brief.md`。同形小任务批合并：一份 brief 列 N 处同类改动，一次派发，一次评审。

### 2. 派发

- 派发前记 BASE：`git rev-parse HEAD`
- 强模型，派发时**显式指定**（省略会静默继承会话模型）
- 用 [implementer-prompt.md](implementer-prompt.md) 模板
- 实现者开工前可以提问（NEEDS_CONTEXT 状态）——这是最便宜的 brief 质检，别催它开工
- 实现者禁止再派子代理；绝不并行派多个实现者
- 派发后别干等：写下一份 brief、更新台账

### 3. 处理报告

| 状态 | 动作 |
|---|---|
| DONE | 生成 diff 包，进评审 |
| DONE_WITH_CONCERNS | 先读顾虑：正确性/范围问题先处理再评审；观察性顾虑记台账，进评审 |
| NEEDS_CONTEXT | 补上下文，同模型重派 |
| BLOCKED | 上下文不足 → 补；任务过大 → 拆；spec 有错 → 先裁决，带裁决重派 |

### 4. 评审（新鲜上下文，模板见 [reviewer-prompt.md](reviewer-prompt.md)）

- diff 以**文件**交给评审者，输出不进你的上下文：
  ```bash
  { git log --oneline BASE..HEAD; echo; git diff --stat BASE..HEAD; echo; git diff -U10 BASE..HEAD; } > .agent/task-N-diff.md
  ```
- 评审者判 **Missing / Extra / Misunderstood** + 质量，严重度三档（Critical / Important / Minor）
- Minor → 记台账延期（`Task N: minor (deferred): <一行>`），不进修复循环，最终评审统一分诊
- "⚠️ 无法从 diff 验证"的条目 → 你自己逐条解决（你持有跨任务上下文）；确认是真缺口就按规格失败进修复循环

### 5. 修复循环（最多 2 轮；一轮 = 修复 + 限定复审）

评审发现 Critical / Important 或规格缺口，进入第 1 轮：

1. **修复**：原实现者带**全部**发现修一次（修复报告追加到同一报告文件，附覆盖测试 + 命令 + 输出）
2. **限定复审**：只验每条发现 ADDRESSED 与否 + 修复 diff 有无新破坏（diff 包从上一轮评审看到的 HEAD 到现在）

复审后仍有未解决的 Critical / Important → 进入第 2 轮：同一实现者，带**剩余**发现，重复上面两步。

第 2 轮复审后仍未解决 → **裁决**。没有第三轮——两轮不收敛说明失败是结构性的，轮数修不了，只有决定能修。

每轮记台账：`Task N: fix round R/2 (X addressed, Y open; commits a1b2c3d..d4e5f6a)`

### 6. 裁决（只有三个出口）

1. **带理由挂起**：`Task N: parked — <发现> — Ruling: <代码为什么站得住>`，留给最终评审
2. **改 brief、换新实现者重派**：裁决记台账，裁决内容写进新 brief
3. **停下浮给用户**：仅当每条前进路径都是瞎猜

每条裁决进台账：`Ruling: <决定> — <理由> — <错了的代价>`。**控制者永不自己修码**——污染上下文且绕过评审。没有第四个出口叫"我顺手修了"。

### 7. 完成

同一时刻做三件簿记：
- 台账追加 `Task N: complete (commits a1b2c3d..d4e5f6a, review clean)`
- **更新接手文档**（见下一节）
- 标记 todo，下一任务

评审有未解决的 Critical/Important 且既没修复也没挂起时，绝不进下一个任务。

全部任务完成 → 移交 **finishing**。

## 接手文档（.agent/handover.md）

写给**换会话后的接班控制者**——会话中断后，新会话里的控制者靠它恢复语境，不必重读全部报告。台账是 append-only 实时日志（定位恢复点），接手文档是一页纸交接 briefing（恢复语境）。每任务完成时与台账同步更新。

```markdown
# 交接：<spec 的一句话主题>
（控制者写给接班控制者；每任务完成时更新）

## 从哪继续
- 已完成：任务 1..N，一行一任务：交付物 + 提交 SHA
- 下一个：任务 N+1 —— 恢复后先读 spec，按 executing 流程写 brief

## 已建立的接口
（写下一份 brief 的"接口事实"从这节抄，仍需对照代码核实）
- `<真实签名>` —— 一句话语义（来源：任务 K，提交 abc1234）

## 裁决与死路（别重新踩）
- Ruling: <决定> — <理由>（任务 K）
- 任务 K：<试过的思路>，两轮未过 —— 别再走，因为 <原因>

## 坑
- <环境怪癖、已知局限、延期未修 minor 的台账位置>
```

写法：只记事实和指针——精确签名、路径、SHA，零过程叙述。全文几十行以内，宁短勿长。

## 单任务模式（Bounded 路径）

一个任务就是全部：brief → 派发 → 评审 → 修复（≤2 轮）→ 汇报。无台账、无接手文档、无最终评审、无 worktree（除非用户要求）。

## 提交规则

- 每任务至少一个提交，跑绿之后提交。提交边界 = 评审边界 = 回滚单元
- **只 `git add <明确路径>`**（围栏内文件 + 测试文件），逐文件 add。**禁止 `git add .` / `git add -A`**
- 提交信息：英文一句话 + 任务编号，如 `feat: add retry guard (task 3)`
- 提交前扫 staged diff：`git diff --cached | grep -E 'BEGIN.*PRIVATE KEY|AKIA[0-9A-Z]{16}|ghp_|sk-|xox[bp]-'`——命中即撤销暂存并报告用户
- **永不 push、永不 merge**——那是 finishing 里用户的菜单

## 禁止提交清单

- **凭证（红线，发现即停）**：`.env*`（`.env.example` 除外）、`*.pem`、`*.key`、`id_rsa*`、`*credential*`、`service-account*.json`、`*.tfvars`、`.npmrc`、`.netrc`、`.aws/`、`.kube/`
- **流程草稿**：`.agent/`、`.worktrees/`、`.claude/settings.local.json`
- **依赖/构建**：`node_modules/`、`.venv/`、`__pycache__/`、`dist/`、`build/`、`target/`
- **杂物**：`.DS_Store`、`*.swp`、`.idea/`、`.vscode/`
- **别误杀（应该提交）**：lock 文件、`.env.example`、CLAUDE.md、团队共享的 `.claude/settings.json`

## 反合理化

| 借口 | 事实 |
|---|---|
| "任务太简单，跳过评审" | 评审是唯一的质量网。简单 = 更快的评审，不是没有评审。 |
| "报告说全过了，信它" | 报告是主张不是证据。评审者看 diff。 |
| "diff 里没测试改动但报告说测了" | 那是伪造证据，按 Missing 上报。 |
| "两轮修不动，再试一轮" | 第三轮不收敛。裁决。 |
| "裁决了，回头补台账" | 没记录的裁决 = 秘密决定。当场记。 |
| "围栏外顺手小修" | 围栏外 = 汇报。 |
| "`git add .` 省事" | 事故入口。逐文件 add。 |
| "实现者自己拉了个评审" | 重复占位。评审是你派的，才算数。 |
