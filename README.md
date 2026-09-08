# onegate

最小化的 AI 编程代理工作流：**人只签一次字**。

一个 spec 的批准是全流程唯一的审批门。门内，强模型代理自主完成任务清单——自己写测试（TDD）、逐任务提交、每个任务过一道新鲜评审、冲突由控制者裁决并全程记录台账；门外，代理只回答问题，发现的任何问题只汇报、不擅改。

```
【咨询模式】（默认）
  问句 → 只回答；发现问题 → 汇报，不动手
  祈使句 → 小活直接干；需要设计的活 → designing
        │
        ▼
  spec（1页）+ 任务清单（1页）+ 执行默认 ── 一并呈现
        │
        ▼  用户批准 = 唯一审批门 = 移交
【执行模式】executing
  spec 落盘 docs/onegate/specs/ → 逐任务串行：brief（just-in-time）→ 强模型实现（TDD）
  → 新鲜评审（Missing/Extra/Misunderstood）→ 修复 ≤2 轮
  → 裁决（挂起 / 重派 / 停下浮给用户）→ 台账一行
  （围栏内全自主；围栏外 / 不可逆 / 推送发布 → 停）
        │  全部任务完成
        ▼
【收尾】finishing
  全量测试 + 整分支通读 → 一轮修复
  → "我做过的裁决"问责清单（穷尽）
  → 收尾菜单（合并 / PR / 保留——用户选）
  → spec 归档 docs/onegate/specs/，清理工作目录
```

## 核心设计

| 机制 | 内容 |
|---|---|
| **同意模型** | 授权范围 = 用户说出的话的范围。问句零授权，祈使句授权其所述范围，spec 批准授权整个计划 |
| **单一审批门** | spec 1 页、任务清单 1 页，人真的会读。批准即开工，中途不再请示 |
| **呈现前强模型批评** | spec 草稿 + 任务清单派强模型攻击（歧义/矛盾/不可判定/缺失决定/范围/誊抄测试），主模型裁决折入后再呈现——批评补的是主模型对话里的盲区 |
| **just-in-time brief** | 每个任务的 brief 在派发前一刻写，接口从已完成任务的实际代码抄——没有基于想象代码的前置计划 |
| **强模型 + 自担 TDD** | 实现者自己写失败测试、自己实现，报告附 RED/GREEN 证据；评审者对照 diff 核实主张 |
| **每任务一道新鲜评审** | 新鲜上下文看 diff，判 Missing / Extra / Misunderstood + 质量，三档严重度 |
| **裁决三出口** | 两轮修不动 → 带理由挂起 / 改 brief 重派 / 停下浮给用户。控制者永不自己修码，每条裁决进台账 |
| **问责清单** | 收尾时把全部裁决穷尽列出——替用户做的每决定都要送达 |
| **自维护文档** | 每任务完成更新接手文档（恢复点/已建接口/裁决与死路/坑），换会话的 agent 无缝接手；收尾把完成状态与偏差折进 spec 归档 |
| **提交安全** | 逐文件 add（禁止 `git add .`）、提交前密钥扫描、凭证/草稿/构建产物四层禁提清单、永不 push |

## 安装（Claude Code）

### 方式一：插件（推荐，含 SessionStart 钩子）

钩子在每个会话开始（含上下文压缩、`/clear` 之后）自动注入同意模型，并检测进行中的执行（`.agent/` 存在时提示恢复路径）：

```
/plugin marketplace add https://github.com/xjzh666/onegate.git
/plugin install onegate@onegate
```

注意用完整 HTTPS URL——`xjzh666/onegate` 简写会被解析成 SSH 协议克隆，带口令的 key 会在非交互环境里直接失败。仓库现为私有：有访问权限且配好 git 凭据的机器可直接装；公开后任何人可装。

无法访问 github.com 的机器，先把仓库放到本地再装：

```
/plugin marketplace add /本地路径/onegate
/plugin install onegate@onegate
```

源文件更新后重跑 install（或提升 plugin.json 的 version）刷新缓存副本。

### 方式二：仅复制技能（无钩子）

```bash
# 个人级：所有项目可用
cp -r skills/* ~/.claude/skills/

# 或项目级：仅当前项目
cp -r skills/* <项目>/.claude/skills/
```

技能按 description 自动触发，但同意模型不会被强制注入。可靠性兜底：在项目 `CLAUDE.md` 加一行

```
任何任务开始前，先读 using-workflow 技能确认同意模型与路由。
```

## 与 superpowers 的区别

| | [obra/superpowers](https://github.com/obra/superpowers) | onegate |
|---|---|---|
| 哲学 | 事前确定性：判断全部前置进计划，执行者是无判断的誊抄机 | 事后验证：判断保留在强模型里，每个判断被新鲜眼光独立复核 |
| 计划 | 几十 KB，含完整代码，人不会读 | 无计划；spec + 任务清单共 2 页，spec 是唯一审批门 |
| 想法→设计 | 独立 brainstorming 技能：分节确认设计 → 落盘 → 再过一道文件审阅门（两次确认） | brainstorm 并入 designing：一次一问问成形，选择题用 AskUserQuestion；单门照旧，批准后落盘 |
| 实现者 | 按计划逐字誊抄，可用最便宜模型 | 按 brief（无代码）自主实现，强模型 |
| 修复循环 | 5 轮 + 断路器 + 模型升级 | 2 轮 + 裁决 |
| 可复现性 | 有（同计划同产出） | 无（同 brief 两次运行产出不同代码）——明码标价的交换 |
| 移交脚本 | 专用脚本 + 工作区管理 | 一个台账文件 + 三条 git 命令 |

## 致谢

设计参考了 [obra/superpowers](https://github.com/obra/superpowers)（Jesse Vincent / Prime Radiant，MIT 协议）——尤其是同意门、反合理化表格、台账、新鲜评审、问责清单这些机制直接源于它。本项目把它们简化为单一审批门 + 强模型 + 事后验证的形态。

## License

建议 MIT（发布前补 LICENSE 文件）。
