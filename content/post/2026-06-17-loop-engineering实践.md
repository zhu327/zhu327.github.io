---
title: "Loop Engineering 实践: Pi Coding Agent"
date: 2026-06-17T09:16:00+08:00
draft: false
---

我在之前写过一些关于AI Agent在开发流程中应用的经验，比如《[打造适合自己的 AI Harness 工程：从开发流、E2E 测试到自动排障](https://zhu327.github.io/2026/05/09/%E6%89%93%E9%80%A0%E9%80%82%E5%90%88%E8%87%AA%E5%B7%B1%E7%9A%84-ai-harness-%E5%B7%A5%E7%A8%8B%E4%BB%8E%E5%BC%80%E5%8F%91%E6%B5%81e2e-%E6%B5%8B%E8%AF%95%E5%88%B0%E8%87%AA%E5%8A%A8%E6%8E%92%E9%9A%9C/)》。随着对AI Agent的深入实践，我发现人肉打Prompt的日子快要到头了，未来我们更多是设计一个系统，让系统来自动化地与Agent交互。最近我一直在研究 Loop Engineering，并尝试将它落地到我的日常工作中，特别是用 Pi Coding Agent 来实现自动化巡检和代码修复。

### 1. Loop Engineering 是什么，由哪些环境组成

<img src="https://github.com/user-attachments/assets/fe3cb3a5-3b7f-4cbf-8121-474b8f645540" alt="Image1" width="700px" />

以前，我们程序员和AI编码Agent的交互模式是这样的：我写一个Prompt，提供上下文，Agent返回结果，我再根据结果写下一个Prompt。Agent就像一个工具，始终在我手里。但现在这种模式正在被 Loop Engineering 所取代。

Loop Engineering 的核心思想是**构建一个小型系统，让这个系统自己去发现工作、把工作交给Agent、检查Agent的产出、记录发生的一切，并自主决定下一步行动**。我们只需要设计好这个系统一次，之后系统就会自动地与Agent交互。Addy Osmani 将其分解为六个部分：

*   **Automation (自动化)**: 触发器，在特定时间、事件或条件发生时启动循环。
*   **State file (状态文件)**: 记录循环的进度和历史，让Agent在每次运行时都能从上次中断的地方继续。
*   **Verifier (验证器)**: 自动化检查Agent产出的质量，判断工作是否完成或需要返工。
*   **Skill (技能)**: 封装项目特定的知识、规则和最佳实践，避免Agent重复学习上下文。
*   **Connector (连接器)**: 让Agent能与外部工具（如Git、Jira、Slack、监控系统）交互，使其能在真实环境中执行操作。
*   **Sub-agent (子Agent)**: 将复杂任务分解给多个Agent并行处理，通常会有一个“生成者”和一个“检查者”Agent，避免“自说自话”。

<!--more-->

Anthropic 的工程师们在 2024 年实现了每天合并的代码量翻了八倍（虽然他们自己也承认这可能有些夸大）。但无论数字如何，核心机制是明确的：**杠杆点已经从“编写Prompt”转移到了“设计驱动Prompt的循环”**。

当然，不是所有任务都适合 Loop Engineering。在投入资源构建之前，我们需要进行一个“4条件测试”：

*   **任务可重复**：循环的设置成本需要通过多次运行来摊销。一次性任务手动Prompt会更快更便宜。
*   **验证自动化**：循环需要有无需人工干预就能判断工作是否失败的机制（例如测试套件、类型检查、Linter、构建）。
*   **Token 预算充足**：循环会重新读取上下文，尝试和探索，这会消耗大量Token。
*   **Agent 拥有资深工程师的工具**：日志、可复现的环境、运行和调试代码的能力。

如果这四个条件有一个不满足，那么构建循环的成本很可能高于其带来的收益。

### 2. goal/loop 命令意味着什么，分别有什么职能

<img src="https://github.com/user-attachments/assets/4dc6a454-afa2-46fe-89a3-7690b87ee44f" alt="Image2" width="700px" />

在 Loop Engineering 中，`loop` 和 `goal` 是两个非常核心的命令，它们赋予了Agent持续运行和自我验证的能力。

*   `/loop`：定义了循环的** cadence (节奏)**，比如每隔 30 分钟运行一次，或者每天固定时间运行。它专注于“何时”运行任务，无论之前的任务是否完成，都会按时触发。
*   `/goal`：则定义了一个**任务的最终结束条件**。它会持续运行，直到你设定的某个条件真正满足为止。这意味着一个单独的小模型会检查完成情况，确保“制造者”Agent不会自己给自己批改作业。

我通常会将它们结合起来使用，比如：

```text
/loop 30m /goal All tests in test/auth pass and lint is clean. Scan src/auth for new failures, propose fixes in claude/auth-fixes, open draft PR when goal condition holds.
```

这个命令的含义是：每隔 30 分钟，Agent 就会检查 `src/auth` 目录，扫描新的故障，并尝试在 `claude/auth-fixes` 分支中提出修复方案，直到 `test/auth` 中的所有测试都通过且 Lint 干净为止，然后才会打开一个 PR。这实现了任务的自动化执行和验证的 maker-checker 分离。

### 3. 我的 Loop Engineering 的思路

在了解了 Loop Engineering 的基本概念和工作原理后，我开始思考如何将它与我现有的 AI Harness 工程结合起来。在《[打造适合自己的 AI Harness 工程：从开发流、E2E 测试到自动排障](https://zhu327.github.io/2026/05/09/%E6%89%93%E9%80%A0%E9%80%82%E5%90%88%E8%87%AA%E5%B7%B1%E7%9A%84-ai-harness-%E5%B7%A5%E7%A8%8B%E4%BB%8E%E5%BC%80%E5%8F%91%E6%B5%81e2e-%E6%B5%8B%E8%AF%95%E5%88%B0%E8%87%AA%E5%8A%A8%E6%8E%92%E9%9A%9C/)》 这篇文章中，我曾提到在建设 `srecli` 的过程中，CLI 已经具备了查询指标（metrics）和日志（logs）的能力。现在，我又加入了 Sentry API 的事件查询功能，这三者的结合，补齐了 Loop Engineering 中“Trigger（触发器）”的部分。通过生成**巡检报告**，我们可以触发一个自动化的循环来识别并改进代码。

我的 Loop Engineering 思路大致分为以下几个步骤：

#### 1. `app-inspection` 创建了一个巡检用的 skill

这个 `app-inspection` skill 是整个循环的起点，它负责对应用程序进行全面的健康检查。

*   **执行 `make lint/make test/make e2e`**: 这是最基础的静态代码检查和自动化测试，确保代码质量和功能正确性。
*   **查询 24 小时内 metrics 5xx 请求路径，应用 ERROR 日志**: 通过 `srecli` 接入的监控系统，我会让 Agent 检查过去 24 小时内服务的 5xx 错误率和具体的出错路径，以及应用程序的 ERROR 级别日志，从中识别潜在的运行时问题。
*   **查询 24 小时 Sentry events**: Sentry 作为错误监控工具，能够捕获应用程序的异常和错误。Agent 会查询 Sentry 中的新事件，及时发现未被其他监控捕获的问题。
*   **出巡检报告**: 综合以上所有信息，Agent 会生成一份结构化的巡检报告，这份报告是后续判断是否需要代码修复的关键依据。

这个 `app-inspection` skill 扮演了“Trigger”的角色，它自动化地检查了代码和服务的健康状况，作为自动化的发起条件触发为代码改进提供决策前提条件。

#### 2. 记忆 `LOOP.md` 记录每次执行结果

为了让 Agent 拥有“记忆”，我引入了一个仓库根目录下的 `LOOP.md` 文件作为**状态文件 (State File)**。Agent 每次执行循环前，都会读取 `LOOP.md` 的内容，特别是其中的 `In progress / Escalated to humans / Lessons learned` 这三段。

*   **`In progress` (进行中)**：记录上次循环中识别但尚未完成修复的问题。
*   **`Escalated to humans` (已升级至人工处理)**：记录 Agent 尝试修复但无法自动解决，需要人工干预的问题。
*   **`Lessons learned` (经验教训)**：沉淀每次修复的经验，避免 Agent 重复犯错。

正如 Addy Osmani 所说，“Agent 会遗忘，但仓库不会”。`LOOP.md` 确保了 Agent 在每次运行时都能恢复到之前的状态，而不是从零开始，这极大地提升了循环的效率和可靠性。

```markdown
# Loop state · ci-triage

## Last run
2026-06-09 03:30 UTC · 7 failures classified, 3 fixes drafted, 4 escalated

## In progress
- claude/fix-auth-token-refresh — tests passing locally, awaiting CI
- claude/fix-flaky-payment-webhook — retry pattern applied, monitoring

## Completed today
- claude/bump-axios-1.7.4 → merged (CI green, deps loop verified)
- claude/lint-fix-pass-june-9 → merged

## Escalated to humans
- src/billing/refund.ts — tests failing in 3 ways, root cause unclear
- ci/staging-runner — infra timeouts, not a code issue

## Lessons learned (write here, not in chat)
- 2026-06-08: PowerShell hits TLS 1.2 issue on this Windows runner. Use bash.
- 2026-06-07: tests/e2e/checkout requires Stripe webhook secret in env. Skip if missing.

## Stop conditions met since last review
- /goal “all tests pass + lint clean” achieved on commit 3a7b8c1 at 02:14 UTC
```

#### 3. 判断跟进巡检报告，以及记忆内容，判断哪些问题需要代码修复

这是整个循环的“决策中心”。Agent 会综合分析 `app-inspection` 生成的巡检报告和 `LOOP.md` 中的历史记录：

*   **优先接续 `In progress` 中未完成项**：确保上次循环中开始修复的问题能够继续推进。
*   **复核 `Escalated to humans` 是否已可处理**：检查之前需要人工介入的问题，看是否已具备自动处理的条件。
*   **避免重复已完成项**：通过 `LOOP.md` 中的记录，避免重复处理已经解决的问题。

如果巡检报告没有告警或异常，并且 `LOOP.md` 中也没有任何待处理或待复核的开放项，那么 Agent 会判断本期无需处理，直接在 `LOOP.md` 顶部写入“本期无需处理的代码问题”并结束本次循环，不进行后续的代码修复流程。这就像一个“早退闸门”，避免不必要的Token消耗和资源占用。

#### 4. 切分支，走 `go.md` 的 writing plan 后续流程完成代码修复

如果 Agent 判断存在需要修复的代码问题，那么它会基于 `master` 分支切出一个新的修复分支，然后进入我之前文章中提到的 `go` 启动 Prompt 流程。

```markdown
## Required Go Gates

Maintain these gates visibly and include their final statuses in the final report:

- `GO-1: Brainstorming design approved` — status: `completed`, `blocked: <reason>`, or `skipped: <valid reason>`
- `GO-2: Design document saved` — status: `completed`, `blocked: <reason>`, or `skipped: <valid reason>`
- `GO-3: Implementation plan saved` — status: `completed`, `blocked: <reason>`, or `skipped: <valid reason>`
- `GO-4: Subagent execution completed` — status: `completed` or `blocked: <reason>`; may not be skipped
- `GO-5: Validation completed` — status: `completed`, `blocked: <reason>`, or `skipped: <valid reason>` only when no applicable validation commands are available
- `GO-6: Whole-change review completed and required fixes handled` — status: `completed` or `blocked: <reason>`; may not be skipped
- `GO-7: Simplifier completed` — status: `completed` or `blocked: <reason>`; may not be skipped
- `GO-8: Final report delivered` — status: `completed`, `blocked: <reason>`, or `skipped: <valid reason>`
```

在 Loop Engineering 的场景下，我会明确要求 Agent **跳过 `GO-1: Brainstorming design approved` 阶段**。因为问题已经通过巡检报告和 `LOOP.md` 明确识别并锁定了，Agent 无需再进行额外的用户确认。它会直接将“巡检报告 + LOOP.md 判定出的待修复问题”作为锁定需求，从 `GO-2`（Design document saved）阶段开始，自动完成后续的 `writing-plans`、执行、验证、review 和 simplify 等所有流程，不再向用户提问。这极大地减少了人工介入，实现了开发流程的高度自动化。

#### 5. 提交 PR

当 `go.md` 的所有流程（从 `GO-2` 到 `GO-7`）都顺利完成后，Agent 会自动提交 Git Commit，并使用 `kgit mr master` 发起合并请求 (Pull Request)。最后，它会在 `LOOP.md` 中写入本次运行的摘要，并根据本轮结果更新 `In progress / Completed today / Escalated to humans / Lessons learned / Stop conditions` 各段。

### 4. Pi Coding Agent 的实战

为了实现上述 Loop Engineering 的思路，我选择了 `earendil-works/pi` 这个 Agent 框架。

#### 1. 配置 Pi Coding Agent

我的 `pi` 配置仓库 (<https://github.com/zhu327/pi>) 结构如下:

```
.pi/agent/
├── AGENTS.md              # Karpathy 风格的 agent 行为准则
├── extensions/            # 自定义扩展
│   ├── goal.ts            # 长时间目标跟踪与自动续跑（/goal 命令）
│   ├── grep-find.ts       # 确保 grep、find 工具始终激活
│   ├── model-patches.ts   # 模型级 patch（qwen3.7-max xhigh reasoning）
│   ├── question.ts        # 结构化多问题 UI（chips、多选、预览）
│   ├── todo.ts            # 4 状态任务管理 + overlay widget（/todos 命令）
│   └── web-fetch.ts       # URL 抓取转 markdown，含 SSRF 防护
├── prompts/
│   └── handoff.md         # 会话交接 prompt（替代上下文压缩）
└── skills/
    └── skill-creator/     # Skill 创建、评估、迭代优化工具链
```

通过 `pi install` 命令，我安装了两个关键的插件：

*   `@gotgenes/pi-subagents`: 实现了 Claude Code 风格的子代理，支持并行执行任务、隔离会话。这对于实现“制造者”和“检查者”分离的子 Agent 模式至关重要。
*   `@koltmcbride/pi-loop`: 提供了定时/循环 Prompt 调度功能，可以按时间间隔或 Cron 表达式反复执行任务，是整个 Loop Engineering 的核心调度器。

在设计选择上，我遵循了几个原则：

*   **交接优于上下文压缩**：使用 `prompts/handoff.md` 生成结构化交接文档，让新的 Agent 会话能够无缝接续工作，而不是在单个超长会话中压缩上下文，有效避免了上下文丢失和Token浪费。
*   **Karpathy 行为准则**：`AGENTS.md` 约束 Agent 的行为，要求“先想再写、最简实现、手术式改动、目标驱动执行”，减少过度工程和不必要的修改。

#### 2. 用 goal meta 出一个 `goal` 的 Prompt

`goal` 最重要的职能是控制持续运行任务的最终结束验证条件以及一系列约束。为此，我利用了 `https://github.com/joeseesun/qiaomu-goal-meta-skill` 这个技能，它能够很好地实现 `goal` 的各种约束定义。

以下是我为 `kfinops CI` 分诊循环设计的一个 `/goal` Prompt，它详细定义了 Agent 的目标、验证、约束、边界、迭代策略以及完成/暂停条件：

```text
/goal 运行一次 kfinops CI 分诊循环：用 app-inspection skill 跑完整巡检拿到报告；读取仓库根目录 LOOP.md 全文（重点 In progress / Escalated to humans / Lessons learned 三段），结合巡检报告判定本期要处理的真实代码问题——优先接续 In progress 中未完成项、复核 Escalated 是否已可处理、避免重复已完成项。早退闸门：若巡检报告无告警/异常，且 LOOP.md 三段（In progress / Escalated to humans / Lessons learned）中也没有任何可处理或待复核的开放项，则判定本期无需处理，立即在 LOOP.md 顶部 Last run 写明「本期无需处理的代码问题（巡检无异常 + LOOP.md 无开放项）」并直接结束 goal，不切分支、不进入 go.md 流程。否则基于 master 用 `kgit new master <task_name>` 切出修复分支；走 `.pi/prompts/go.md` 的 writing-plans 及后续全流程并自动跑完；提交 git commit 后用 `kgit mr master` 发起合并请求；最后在 LOOP.md 写入新的 Last run 条目，并将 In progress / Completed today / Escalated to humans / Lessons learned / Stop conditions 各段按本轮结果更新（保留历史已完成与已积累的 lessons，只追加不覆盖）。
验证：app-inspection 报告已生成；已读取 LOOP.md 旧状态；修复分支已存在（`git branch` 可见）；go.md 的 GO-2/GO-3/GO-4/GO-5/GO-6/GO-7 各 gate 状态已记录；`git log` 显示本次 commit；`kgit mr master` 返回 MR 编号或 URL；LOOP.md 顶部 Last run 已替换为本次时间戳与摘要，且 Completed today 或 Escalated to humans 已追加本轮分支/MR/结果。
约束：使用 go.md 流程，但跳过 brainstorming 阶段的 `question` 用户确认——把「巡检报告 + LOOP.md 判定出的待修复问题」作为锁定需求直接进入 writing-plans，之后的执行、验证、review、simplify 全部自动完成，不再向用户询问；不直接 push 或提交到 master；不修改 .pi/ 目录、巡检脚本、CI 配置、生产配置和任何密钥；不改动与本问题无关的模块；LOOP.md 只追加/更新本轮相关行，不删除历史记录与已沉淀的 lessons；判定无问题时只更新 LOOP.md 的 Last run 一行即退出，不做其他写入。
边界：只允许写入当前修复分支内的业务代码文件 + LOOP.md（仓库根目录）；禁止触碰 master 分支、其他进行中分支、.pi/、infra/生产 manifest、以及修复目标之外的破坏性操作。
迭代策略：一次只修一个被选中的真实问题；go.md 子代理执行后按验证命令（make lint-full / go test / make e2e 中与改动相关的最小集合）重跑检查；同一问题在 lint/test 上连续 2 次仍不绿必须换证据来源或降级为 escalated；单次循环最多 3 轮聚焦修复，超出则暂停并报告，并把降级项写入 LOOP.md 的 Escalated to humans。
完成条件：本次选中的修复有「分支 + commit + MR + 检查通过或明确失败原因 + LOOP.md 已更新」五项证据齐全；或在巡检 + 读 LOOP.md 后判定本期无真实可处理问题，已仅在 LOOP.md 的 Last run 写明「本期无需处理的代码问题（巡检无异常 + LOOP.md 无开放项）」后立即结束，不切分支、不进入 go.md 流程。
暂停条件：Sentry token 失效或 srecli 不可用导致巡检拿不到数据；修复需要破坏性变更、DB 迁移、付费服务、凭证或产品决策；同一问题连续 2 次无法变绿；MR 发起失败或需要人工审批；所有权或责任归属不清。
```

#### 3. 配置到 Pi Coding Agent 的 Loop 流程中

在实际的 Pi Coding Agent 运行中，我们不能直接使用 `/loop 0 8 * * * /goal <prompt>` 这种方式来同时激活 `loop` 和 `goal`。因为 `pi` 的 `loop` 插件和 `goal` 扩展是分开工作的，`loop` 负责定时触发一个 Prompt，而 `goal` 负责持续推进一个长期目标。

<img src="https://github.com/user-attachments/assets/9a872786-0343-4bb5-b4e6-5d6f210bacea" alt="Image3" width="700px" />

所以，我通过 `LoopCreate` 工具创建了一个更加复杂的 `loop` 定义，它会在每次触发时，借助 `goal` 扩展的 `get_goal/create_goal/update_goal` 工具来管理 `goal` 的状态。

```json
{
  "nextId": 2,
  "loops": [
    {
      "id": "1",
      "prompt": "这是每天 08:00 自动触发的 kfinops CI 分诊循环（cron: 0 8 * * *）。全程自动完成，不要向用户提问。\n\n第一步——goal 状态检查：调用 get_goal 工具。若存在 status 非 complete 的 goal 且其 objective 仍是本 CI 分诊任务（通常是昨天未跑完的遗留），直接继续推进它，跳到第三步；若遗留 goal 与当前任务无关或明显卡死，先把情况记入 LOOP.md 的 Escalated to humans，然后用 update_goal 标记 complete 以便新建。\n\n第二步——创建 goal（仅当无 active goal 时）：调用 create_goal 工具，objective 使用下方 === GOAL OBJECTIVE === 与 === END === 之间的整段原文（不要改写、不要省略）。\n\n=== GOAL OBJECTIVE ===\n运行一次 kfinops CI 分诊循环：用 app-inspection skill 跑完整巡检拿到报告；读取仓库根目录 LOOP.md 全文（重点 In progress / Escalated to humans / Lessons learned 三段），结合巡检报告判定本期要处理的真实代码问题——优先接续 In progress 中未完成项、复核 Escalated 是否已可处理、避免重复已完成项。早退闸门：若巡检报告无告警/异常，且 LOOP.md 三段中也没有任何可处理或待复核的开放项，则判定本期无需处理，立即在 LOOP.md 顶部 Last run 写明「本期无需处理的代码问题（巡检无异常 + LOOP.md 无开放项）」并直接结束 goal，不切分支、不进入 go.md 流程。否则基于 master 用 `kgit new master <task_name>` 切出修复分支；走 `.pi/prompts/go.md` 的 writing-plans 及后续全流程并自动跑完；提交 git commit 后用 `kgit mr master` 发起合并请求；最后在 LOOP.md 写入新的 Last run 条目，并将 In progress / Completed today / Escalated to humans / Lessons learned / Stop conditions 各段按本轮结果更新（保留历史已完成与已积累的 lessons，只追加不覆盖）。\n验证：app-inspection 报告已生成；已读取 LOOP.md 旧状态；修复分支已存在（`git branch` 可见）；go.md 的 GO-2/GO-3/GO-4/GO-5/GO-6/GO-7 各 gate 状态已记录；`git log` 显示本次 commit；`kgit mr master` 返回 MR 编号或 URL；LOOP.md 顶部 Last run 已替换为本次时间戳与摘要，且 Completed today 或 Escalated to humans 已追加本轮分支/MR/结果。\n约束：使用 go.md 流程，但跳过 brainstorming 阶段的 `question` 用户确认——把「巡检报告 + LOOP.md 判定出的待修复问题」作为锁定需求直接进入 writing-plans，之后的执行、验证、review、simplify 全部自动完成，不再向用户询问；不直接 push 或提交到 master；不修改 .pi/ 目录、巡检脚本、CI 配置、生产配置和任何密钥；不改动与本问题无关的模块；LOOP.md 只追加/更新本轮相关行，不删除历史记录与已沉淀的 lessons；判定无问题时只更新 LOOP.md 的 Last run 一行即退出，不做其他写入。\n边界：只允许写入当前修复分支内的业务代码文件 + LOOP.md（仓库根目录）；禁止触碰 master 分支、其他进行中分支、.pi/、infra/生产 manifest、以及修复目标之外的破坏性操作。\n迭代策略：一次只修一个被选中的真实问题；go.md 子代理执行后按验证命令（make lint-full / go test / make e2e 中与改动相关的最小集合）重跑检查；同一问题在 lint/test 上连续 2 次仍不绿必须换证据来源或降级为 escalated；单次循环最多 3 轮聚焦修复，超出则暂停并报告，并把降级项写入 LOOP.md 的 Escalated to humans。\n完成条件：本次选中的修复有「分支 + commit + MR + 检查通过或明确失败原因 + LOOP.md 已更新」五项证据齐全；或在巡检 + 读 LOOP.md 后判定本期无真实可处理问题，已仅在 LOOP.md 的 Last run 写明「本期无需处理的代码问题（巡检无异常 + LOOP.md 无开放项）」后立即结束，不切分支、不进入 go.md 流程。\n暂停条件：Sentry token 失效或 srecli 不可用导致巡检拿不到数据；修复需要破坏性变更、DB 迁移、付费服务、凭证或产品决策；同一问题连续 2 次无法变绿；MR 发起失败或需要人工审批；所有权或责任归属不清。\n=== END ===\n\n第三步——执行：goal 创建（或接续）后，立即按 objective 中的 验证/约束/边界/迭代策略 推进；走 .pi/prompts/go.md 流程但跳过 brainstorming 的 question 确认。\n\n第四步——收尾：达到 objective 的完成条件后，调用 update_goal 工具标记 complete。",
      "trigger": {
        "type": "cron",
        "schedule": "0 8 * * *"
      },
      "status": "active",
      "recurring": true,
      "createdAt": 1781595119955,
      "updatedAt": 1781654404411,
      "expiresAt": 1782199919955,
      "source": "tool",
      "fireCount": 1
    }
  ]
}
```

这个 JSON 配置定义了一个每天早上 8 点触发的循环。每次循环启动时，它会首先检查是否存在未完成的 `goal`，如果存在，则继续推进；否则，它会创建一个新的 `goal`，其 `objective` 就是上面那段长 Prompt。然后，Agent 会按照 `objective` 中定义的验证、约束、边界和迭代策略来执行，并最终在 `goal` 完成时将其标记为 `complete`。

通过这种方式，我成功地让我的 Pi Coding Agent 实现了：每天自动进行应用巡检，根据巡检报告和历史记录判断是否需要代码修复，如果需要则全自动地切分支、走完整个代码编写、测试、审查流程，并最终提交 PR。

<img src="https://github.com/user-attachments/assets/c12d2fc6-092c-40cd-9a99-5c1ffddfffcb" alt="Image4" width="700px" />

### 总结

在过去的两年里，与编码 Agent 协作的重心一直放在 Prompt 上，我们追求更好的 Prompt、更精确的上下文、更优秀的单次输出。但现在，这个阶段正在结束。Agent 已经足够强大，下一个杠杆点已经提升了一个层次：**它转变为设计一个系统，由这个系统来决定 Agent 何时工作、做什么、通过什么门槛、以及哪些状态需要在不同运行之间持久化**。

但坦率地说，并非所有开发者都应该立即冲向 Loop Engineering。只有当任务重复、验证自动化、预算能够承受消耗、并且 Agent 拥有资深工程师的工具时，Loop Engineering 才能发挥其价值。如果错过其中任何一个条件，那么构建循环的成本可能远高于其回报。

如果这些条件都符合，那么就从小处着手：一个自动化、一个技能、一个状态文件、一个验证门。先让手动运行变得可靠，然后将其封装成技能，再用循环来调度。顺序至关重要。跳过这些步骤，你最终可能会为一套无人理解的系统买单。

杠杆点已经移动，我们的工作职责也随之改变。构建循环，但要始终保持工程师的思维。