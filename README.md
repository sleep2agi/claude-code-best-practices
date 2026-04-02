# Claude Code 最佳实践

> 经过实战验证的多 Session、多模型、多服务器 Claude Code 运维模式。
> 源自 48+ 小时、4 台服务器、15+ 并发 Agent Session 的真实生产经验。

---

## 目录

1. [多 Session 管理](#1-多-session-管理)
2. [跨服务器通信（MCP SSE 星型架构）](#2-跨服务器通信)
3. [Channel 配置（Telegram / 微信 / 飞书）](#3-channel-配置)
4. [Codex Plugin 安装和用法](#4-codex-plugin)
5. [自进化 Loop](#5-自进化-loop)
6. [指挥室模式](#6-指挥室模式)
7. [Session 迁移（云存储中转）](#7-session-迁移)
8. [多模型后端配置](#8-多模型后端配置)

---

## 1. 多 Session 管理

### 1.1 Session Fork（克隆会话）

需要同一项目的多个 Session（如一个做开发、一个做 Review）时：

```bash
# 1. 找到要 fork 的 session 文件
ls ~/.claude/projects/-home-user-myproject/*.jsonl

# 2. 复制 .jsonl，用新 UUID 作文件名
cp ~/.claude/projects/-home-user-myproject/{original,new-uuid}.jsonl

# 3. 启动 fork 出来的 session
claude --resume new-uuid
```

Fork 出的 Session 继承完整对话历史，从复制点开始分叉。

### 1.2 Session Resume（恢复会话）

始终用 `--resume` 保持上下文连续性：

```bash
# 恢复指定 session
claude --resume <session-id>

# session-id 就是 .jsonl 文件名（不含扩展名）
```

**为什么重要**：不用 `--resume` 每次都是冷启动；用了之后 Agent 保留所有历史上下文、决策和经验。

### 1.3 Session 专业化分工

每个 Session 分配明确角色，不混用：

| Session 名 | 角色 | 工作目录 |
|------------|------|---------|
| `hub` | 指挥室（只调度不干活） | `~/command-center` |
| `frontend-dev` | 前端开发 | `~/myapp` |
| `backend-dev` | 后端开发 | `~/myapp-api` |
| `reviewer` | 代码审查（Codex） | `~/myapp` |
| `video-gen` | 内容生成 | `~/content` |

**铁律**：一个 Session 一个项目目录，永远不要在 `~` 裸跑。

### 1.4 tmux 作为 Session 基础设施

所有 Session 跑在 tmux 中，断连不丢失：

```bash
# 创建命名 session
tmux new-session -d -s frontend-dev -c ~/myapp

# 接入查看
tmux attach -t frontend-dev

# 列出所有 session
tmux ls
```

tmux 会话在 SSH 断开后仍然存活——Agent 持续运行。

---

## 2. 跨服务器通信

### 2.1 tmux send-keys 的问题

朴素方案——跨 SSH 用 `tmux send-keys` + `capture-pane`——有致命缺陷：

| 问题 | 后果 |
|------|------|
| 无法区分 Shell 和 Claude Code 界面 | 发错命令类型（危险） |
| send-keys 末尾忘了 `Enter` | 命令打出来但没执行 |
| capture-pane 混杂 ANSI 转义码 | 输出解析持续崩溃 |
| 跨服务器需要 SSH 嵌套 | 超时链路，连接脆弱 |
| 无消息队列 | 发给忙碌 Agent 的命令丢失 |

**实测可靠性约 20%。仅作最后手段。**

### 2.2 MCP SSE 星型架构（推荐方案）

已确认的架构：中心 Commander MCP Server + 持久 SSE 连接：

```
                    ┌────────────────────────────┐
                    │   Commander MCP Server      │
                    │   x.x.x.x:9200             │
                    │                              │
                    │   MCP SSE  +  HTTP REST     │
                    │   （双接口）                  │
                    └──────────┬───────────────────┘
                               │
          ┌────────┬───────┬───┴───┬───────┬────────┐
          │        │       │       │       │        │
       Claude   Claude  Claude  Codex   Codex   Claude
       Code #1  Code #2 Code #N  #1      #2     Code #M
```

**核心设计决策**：

1. **星型拓扑** -- 30 个 Session = 30 条 SSE 连接（不是 N^2 的网状）
2. **SSE 而非轮询** -- 持久连接、实时推送、零浪费 Token
3. **双接口** -- MCP SSE 供 Agent 用 + HTTP REST 供监控面板/脚本用
4. **跨模型中转** -- Claude Code 和 Codex 通过 Commander 互相通信

### 2.3 客户端配置

每个 Agent 在 `settings.json` 里加一行即可：

```json
{
  "mcpServers": {
    "commander": {
      "url": "http://x.x.x.x:9200/sse"
    }
  }
}
```

### 2.4 Commander MCP 工具

**子 Agent 工具**（4 个）：

| 工具 | 用途 |
|------|------|
| `report_status` | 心跳 + 当前任务 + 进度（返回 inbox 数量） |
| `report_completion` | 任务完成，提交结果和产出物 |
| `get_inbox` | 拉取 Hub 下发的待办命令 |
| `ack_inbox` | 确认收到命令 |

**Hub 工具**（5 个）：

| 工具 | 用途 |
|------|------|
| `get_all_status` | 全局状态面板 |
| `get_session_status` | 查看单个 Session 详情 |
| `send_task` | 带优先级下发任务 |
| `broadcast` | 群发（可按服务器/状态过滤） |
| `get_completions` | 获取已完成任务结果 |

### 2.5 通信规则（写入 CLAUDE.md）

```markdown
## Commander 通信规则
- 开始任务时调用 report_status(status="working", task="...")
- 有重大进展时更新 report_status(progress=50)
- 每次 report_status 后检查返回的 inbox_count
- 如果 inbox_count > 0，立即调用 get_inbox()
- 收到命令后用 ack_inbox(message_id) 确认
- 任务完成时调用 report_completion(task, result, artifacts)
```

### 2.6 方案对比

| 方案 | 可靠性 | 延迟 | 跨服务器 | 建设成本 |
|------|--------|------|---------|---------|
| tmux send-keys | 20% | 3-5 分钟 | 支持（SSH） | 零 |
| Codex MCP Tool | 95% | 30 秒 | 不支持 | 零 |
| Agent Teams | 90% | 内置 | 不支持 | 1 个环境变量 |
| Commander MCP (SSE) | 99% | 实时 | 支持 | 1-2 天 |

**原则**：MCP 调用比 tmux 高效 10 倍。本地操作优先用 MCP，跨服务器用 Commander。

---

## 3. Channel 配置

### 3.1 Telegram Channel 安装

**安装插件**：
```bash
/plugin marketplace add claude-plugins-official
/plugin install telegram@claude-plugins-official
/telegram:configure    # 粘贴你的 Bot Token
```

**启动时加载 Channel**：
```bash
claude --channels plugin:telegram@claude-plugins-official
```

> **注意**：不加 `--channels`，Bot 会静默无法连接。

**配对流程**：
1. 在 Telegram 给你的 Bot 发任意消息
2. Claude Code 终端出现 6 位配对码
3. 在 Claude Code 中执行 `/telegram:access pair <配对码>`
4. 设置策略：`/telegram:access policy allowlist`

### 3.2 多 Bot 隔离（TELEGRAM_STATE_DIR）

每个 Bot 拥有独立的凭证和访问控制目录：

```
~/.claude/channels/
├── telegram-main/          # 主控 Bot
│   ├── .env                # TELEGRAM_BOT_TOKEN=xxx
│   └── access.json         # {"dmPolicy":"allowlist","allowFrom":["123"]}
├── telegram-project-a/     # 项目 A Bot
│   ├── .env
│   └── access.json
└── telegram-project-b/     # 项目 B Bot
    ├── .env
    └── access.json
```

**指定 Bot 启动**：
```bash
TELEGRAM_STATE_DIR=~/.claude/channels/telegram-project-a \
claude --dangerously-skip-permissions \
  --channels plugin:telegram@claude-plugins-official \
  --teammate-mode in-process
```

**最佳实践**：一个 Bot 对应一个 Session 对应一个项目，绝不共享。

### 3.3 微信 Channel

使用 ClawBot ilink API 做消息桥接：

```bash
claude --dangerously-load-development-channels server:wechat
```

**功能**：文字/语音/图片、扫码登录、长轮询
**限制**：单用户单 Session（ClawBot 限制）

### 3.4 飞书 Channel

使用飞书 WebSocket 实时消息：

```bash
claude --dangerously-load-development-channels server:feishu
```

**功能**：原生多用户、图片上传下载、App ID + Secret 认证

### 3.5 多 Channel 共存

单个 Session 可同时加载多个 Channel：

```bash
TELEGRAM_STATE_DIR=~/.claude/channels/telegram-main \
claude \
  --channels plugin:telegram@claude-plugins-official \
  --dangerously-load-development-channels server:wechat server:feishu
```

消息通过 `source` 属性自动路由：
```xml
<channel source="telegram" ...>消息内容</channel>
<channel source="wechat" ...>消息内容</channel>
<channel source="feishu" ...>消息内容</channel>
```

---

## 4. Codex Plugin

### 4.1 安装

```bash
# 添加 marketplace
/plugin marketplace add openai/codex-plugin-cc

# 安装插件
/plugin install codex@openai-codex

# 配置
/codex:setup
```

### 4.2 核心命令

| 命令 | 功能 | 使用场景 |
|------|------|---------|
| `/codex:review` | 只读代码审查 | 每个 PR、每次重大改动 |
| `/codex:adversarial-review` | 对抗性审查（质疑设计决策） | 鉴权改动、数据库迁移、基础设施脚本 |
| `/codex:rescue` | 把卡住的任务交给 Codex | 当前思路走不通时 |
| `/codex:status` | 查看后台任务进度 | 长时间运行的审查 |
| `/codex:result` | 获取已完成的输出 | 后台任务完成后 |

### 4.3 MCP Tool 直接调用

```json
mcp__codex__codex({
  "prompt": "审查这个组件，找出最严重的 3 个问题。",
  "cwd": "/path/to/project",
  "sandbox": "danger-full-access",
  "approval-policy": "never"
})
```

**关键参数**：
- `sandbox: "danger-full-access"` -- 服务器上没有 bwrap namespace 权限时必须用
- `approval-policy: "never"` -- 自主执行（不弹确认）
- `cwd` -- 必须指向实际项目目录

### 4.4 Prompt 四要素

每条有效的 Codex 提示需要：

1. **目标**：改什么、为什么改
2. **上下文**：相关文件、报错信息
3. **约束**：代码风格、架构边界
4. **完成标准**：明确的成功判定

```
审查 UserProfile.tsx（1200 行）。问题：上帝组件导致不必要的
重渲染。要求：1) 找出可提取的子组件 2) 列出性能热点 3) 提出
拆分方案。约束：不改现有 API/Props。完成标准：输出完整拆分
计划，每个组件附职责说明。
```

### 4.5 Codex vs Claude Code 选择

| 任务类型 | 用 Codex | 用 Claude Code |
|---------|---------|---------------|
| 代码审查 | 适合（只读、快速） | 不适合 |
| 独立 Bug 修复 | 适合 | 不适合 |
| 写测试 | 适合 | 不适合 |
| 多步骤协调 | 不适合 | 适合 |
| 浏览器自动化 | 不适合 | 适合 |
| MCP 工具链 | 不适合 | 适合 |

**最佳组合**：Claude Code 做规划，Codex 做执行。Claude Code 审查 Codex 的产出。

### 4.6 常见问题排查

| 症状 | 修复方法 |
|------|---------|
| "Re-connecting" 循环 | `codex logout && rm ~/.codex/auth.json`，重新登录 |
| bwrap 报错 | 改用 `sandbox: "danger-full-access"` |
| 401 错误 | `unset OPENAI_API_KEY`，用 `codex login` 替代 |
| 命令被拦截 | 把 sandbox 改成 `workspace-write` 或 `danger-full-access` |

---

## 5. 自进化 Loop

### 5.1 概念

自进化 Loop 是指 Agent 在无人干预的情况下，迭代改进自身产出。只有 Claude Opus 能可靠地自循环。

### 5.2 模型自循环能力

| 模型 | 自循环 | 行为表现 |
|------|--------|---------|
| Claude Opus | 强（26+ 轮自主运行） | 设定目标、走开、它自己迭代 |
| MiniMax / Qwen | 不能（每轮空转） | Hub 必须每 5 分钟推一次"继续" |
| Codex | 单次执行 | 跑一次、交付、停止 |

### 5.3 自循环 Prompt 模板

写入 CLAUDE.md 或 Session 指令中：

```markdown
完成当前任务后不要停止。自动执行：
1. 对照检查清单评估产出质量
2. 找出最需改进的 3 个点
3. 立即开始下一轮迭代
4. 每轮结束后更新经验文档
5. 上下文快满时，把经验总结为 Skills，然后开新 Session 继续
```

### 5.4 TreeSkill 模式（结构化自进化）

TreeSkill 是一个免训练的 Skill 进化框架：

1. **定义 Skill**：明确输入/输出规格
2. **跑迭代**：带质量评分
3. **分支改进**：树状结构（非线性）
4. **剪掉失败分支**，保留最优方案
5. **持久化获胜 Skill**，跨 Session 复用

### 5.5 视频制作 Loop

内容生成（视频、PPT、图片）的典型循环：

```
第 1 轮：生成初版
第 2 轮：对照质量清单自检
第 3 轮：修复发现的问题
第 4 轮：重新渲染并验证
第 N 轮：直到达到质量阈值
```

**严重警告**：AI 自我评分不可信。

实战中，某 Session 给视频打了 9.1/10 分，实际却是黑屏 + 中文字幕渲染成方块。**交付前必须人工抽检。**

### 5.6 质量验证

永远不要相信自评分。加客观检查：

```bash
# 视频：检测黑帧
ffprobe -f lavfi -i "movie=output.mp4,blackdetect=d=0.1" -show_entries ...

# 图片：验证尺寸和内容
identify output.png

# 代码：跑真实测试
npm test
```

---

## 6. 指挥室模式

### 6.1 核心原则

指挥室（Hub）Session **只调度、不执行**：

- 不写代码
- 不编辑文件
- 不做渲染
- 不跑耗时任务

它只负责协调、监控和派活。其他一切都委派。

### 6.2 非阻塞操作

**铁律**：永远不要阻塞指挥室。所有重活都在子 Session 中跑。

| 操作 | 错误做法 | 正确做法 |
|------|---------|---------|
| 代码审查 | 指挥室自己 review | 派给 Codex |
| Bug 修复 | 指挥室写代码 | 派给开发 Session |
| 视频渲染 | 指挥室渲染 | 派给渲染 Session |
| 文件传输 | 指挥室等 scp | 后台执行 |

### 6.3 巡查循环

指挥室定期巡查所有子 Session：

```
每 5 分钟巡查一轮：

1. get_all_status()
   ├── 离线 Session → 告警
   ├── 阻塞 Session → 分析原因，尝试解除
   ├── 空闲 Session → 分配新任务
   └── 报错 Session → 记录日志，尝试重启

2. get_completions(since=上次检查)
   ├── 更新任务看板
   └── 重要完成事项通知操作员

3. 根据操作员指令：
   ├── send_task() → 派任务
   ├── broadcast() → 群发通知
   └── get_session_status() → 查看详情
```

### 6.4 任务派发策略

按任务类型匹配合适的 Agent：

| 任务类型 | 派给谁 | 原因 |
|---------|--------|------|
| 复杂推理 | Claude Opus Session | 唯一能自循环的模型 |
| 代码审查 | Codex（MCP 或 Plugin） | 30 秒出结构化结果 |
| 高频迭代 | Qwen Session | 响应最快 |
| 长期独立任务 | 任意空闲 Session | 按预算选模型 |
| 视觉内容 | 专属渲染 Session | 需要 GPU/大资源 |

### 6.5 常见误区

1. **不要在指挥室干活** -- 一切都派出去
2. **不要等完成** -- 派出去就巡查，不阻塞等结果
3. **不要微观管理** -- 设定目标+约束，检查结果
4. **要维护任务看板** -- 跟踪什么任务分给了谁
5. **要抽检产出** -- AI 自评不可信

---

## 7. Session 迁移

### 7.1 迁移场景

- 把 Session 迁到资源更好的服务器（GPU、带宽）
- 多机负载均衡
- 硬件维护

### 7.2 迁移清单

| 组件 | 方法 | 大小 | 说明 |
|------|------|------|------|
| 代码 | `git clone --depth 1` | 不定 | 浅克隆加速 |
| .env 文件 | `scp` | < 10 KB | 直接复制 |
| Claude Memory | `scp` | 几 KB | `~/.claude/projects/...` |
| Session .jsonl | 云存储中转 | 10-50 MB | 见下文 |
| Channel 配置 | `scp` 目录 | 几 KB | Bot Token + access.json |

### 7.3 云存储中转大文件

Session 历史文件（.jsonl）可达 10-50 MB。走慢速隧道（FRP、SSH）要传很久。用云存储做中转：

```bash
# 源机器：上传到云存储（S3、OSS 等）
aws s3 cp session.jsonl s3://your-bucket/migration/

# 目标机器：从 CDN 下载
wget https://your-cdn.example.com/migration/session.jsonl
```

**性能对比**：
- FRP 隧道：~2 KB/s（50 MB 要好几小时）
- 云存储 CDN：~29 MB/s（50 MB 几秒钟）

### 7.4 路径映射

Claude Code 根据工作目录自动生成项目路径：

```
Linux:  /home/user/myproject → ~/.claude/projects/-home-user-myproject/
macOS:  /Users/user/myproject → ~/.claude/projects/-Users-user-myproject/
```

跨操作系统迁移时，先在目标机器创建正确的路径再复制 Session 文件。

### 7.5 迁移后启动

```bash
# 1. 确认代码已克隆、依赖已安装
cd ~/myproject && npm install

# 2. 确认 .env 存在
ls -la .env

# 3. 把 session 文件复制到正确的 Claude 项目目录
mkdir -p ~/.claude/projects/-home-user-myproject/
cp session.jsonl ~/.claude/projects/-home-user-myproject/

# 4. 复制 memory 文件
cp -r memory/ ~/.claude/projects/-home-user-myproject/memory/

# 5. 带 resume 启动
TELEGRAM_STATE_DIR=~/.claude/channels/telegram-mybot \
claude --dangerously-skip-permissions \
  --channels plugin:telegram@claude-plugins-official \
  --teammate-mode in-process \
  --resume <session-id>
```

---

## 8. 多模型后端配置

### 8.1 模型对比

| 维度 | Claude Opus 4.6 | MiniMax M2.7 | Qwen 3.6 | Codex GPT-5.4 |
|------|-----------------|--------------|-----------|---------------|
| 推理 | 最强 | 中等 | 中偏高 | 强（代码领域） |
| 速度 | 中等 | 快 | 非常快 | 30秒-30分钟 |
| 自循环 | 26+ 轮自主运行 | 不能 | 不能 | 单次执行 |
| 上下文 | 1M tokens | 中等 | 中等 | 1.5M tokens/任务 |
| 成本 | 高 | 低 | 低 | 约 Sonnet 价格 |
| 最佳用途 | 指挥调度、复杂推理 | 长期独立任务 | 高频迭代 | 代码审查、重构 |

### 8.2 配置方式

Claude Code 通过环境变量支持切换模型后端：

```bash
# MiniMax 后端
ANTHROPIC_BASE_URL="https://your-minimax-endpoint/anthropic" \
ANTHROPIC_AUTH_TOKEN="your-token" \
claude --model MiniMax-M2.7

# Qwen 后端
ANTHROPIC_BASE_URL="https://your-qwen-endpoint/anthropic" \
ANTHROPIC_AUTH_TOKEN="your-token" \
claude --model qwen-plus-latest
```

**重要注意事项**：
- 用 `ANTHROPIC_AUTH_TOKEN` 而不是 `ANTHROPIC_API_KEY`（认证机制不同）
- URL 必须包含 `/anthropic` 后缀（兼容端点要求）
- 不同模型的工具支持不同（如部分模型不支持 WebFetch）

### 8.3 模型选择规则

```
需要自循环 → Claude Opus（唯一能自主迭代的模型）
需要代码审查 → Codex（30 秒 MCP Tool 结构化审查）
需要快速迭代 → Qwen（响应最快，但需要定期推一下）
长期独立任务 → 按预算选择
```

### 8.4 非自循环模型管理

MiniMax 和 Qwen 完成一轮就空转。Hub 需要主动推：

```
Hub 巡查循环（每 5 分钟）：
1. 检查 MiniMax/Qwen Session 是否空闲
2. 如果空闲，推一下："继续下一轮迭代"
3. 重复直到任务完成
```

这是多模型运维中最累人的部分。需要持续自主性的任务建议用 Claude Opus。

### 8.5 Codex 配置

```bash
# 安装 Codex CLI
npm install -g @openai/codex

# 登录（保存凭证）
codex login

# full-auto 模式运行
codex --full-auto --model gpt-5.4

# 从 Claude Code 中作为 MCP Tool 调用
mcp__codex__codex({
  "prompt": "审查这段代码",
  "cwd": "/path/to/project",
  "sandbox": "danger-full-access",
  "approval-policy": "never"
})
```

### 8.6 Agent Teams

在单个 Claude Code Session 内启用多 Agent 协作：

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

**启动**：
```bash
claude --teammate-mode in-process
```

**能力**：并行任务派发、worktree 隔离、每个 Agent 独立选模型、后台执行。

**限制**：仅限本机。跨服务器请用 Commander MCP。

---

## 附录 A：常见踩坑

| 坑 | 表现 | 解法 |
|----|------|------|
| SOCKS 代理崩溃 | `UnsupportedProxyProtocol` | 关掉 SOCKS，只用 HTTP 代理 |
| tmux 漏了 Enter | 命令打出来但没执行 | send-keys 末尾永远加 `Enter` |
| Shell/Claude Code 混淆 | 发错命令类型 | 每次 send-keys 前先确认当前界面 |
| 僵尸 Shell 堆积 | 过夜后 60+ 个窗口 | 定期重建 Session |
| CJK 字体缺失 | 中文渲染成方块 | 安装 CJK 字体，drawtext 指定字体路径 |
| 黑屏视频过了检查 | 文件存在但内容为空 | 用 ffprobe 做帧级内容检测 |
| AI 自评分不可信 | 9/10 分实际是废品 | 人工抽检必须做 |
| FRP 隧道凌晨断连 | 2-4 点 Agent 失联 | SSH 隧道备用 + 重试逻辑 |

## 附录 B：Session 启动模板

```bash
# 完整的 Session 启动命令
tmux new-session -d -s my-session -c ~/my-project

# 在 tmux session 内执行：
TELEGRAM_STATE_DIR=~/.claude/channels/telegram-my-bot \
claude \
  --dangerously-skip-permissions \
  --channels plugin:telegram@claude-plugins-official \
  --teammate-mode in-process \
  --resume <session-id>
```

## 附录 C：协议全景（MCP vs A2A vs ACP）

| 维度 | MCP (Anthropic) | A2A (Google) | ACP (BeeAI/IBM) |
|------|----------------|-------------|-----------------|
| **协议层** | 工具接入 | Agent 间协作 | 编排调度 |
| **解决什么** | LLM 调用外部工具 | Agent 互相发现和协作 | 多 Agent 工作流 |
| **传输** | stdio / Streamable HTTP | JSON-RPC 2.0 + HTTP + SSE | HTTP REST + SSE |
| **发现机制** | 手动配 URL | Agent Card (/.well-known/) | 注册中心 |
| **任务生命周期** | 无（同步调用） | 有（submitted→working→done） | 有（Run 状态机） |
| **生态成熟度** | 最大（数千个 Server） | 快速增长（50+ 支持者） | 早期阶段 |

**建议**：MCP 管工具接入（成熟、原生支持），A2A 管 Agent 发现（快速增长中）。两者互补不冲突——MCP 是下层（工具），A2A 是上层（Agent 协作）。

---

## License

MIT
