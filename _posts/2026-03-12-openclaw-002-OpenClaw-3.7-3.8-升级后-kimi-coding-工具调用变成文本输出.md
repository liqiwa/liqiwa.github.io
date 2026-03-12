---
title: "OpenClaw 3.7/3.8 升级后 kimi-coding 工具调用变成文本输出"
date: 2026-03-12
last_modified_at: 2026-03-12
categories:
  - OpenClaw
  - 排障
tags:
  - kimi
  - tool-call
  - upgrade
  - 3.8
  - 3.11
  - regression
author: Liqiwa
toc: true
toc_label: 目录
toc_icon: list-ul
openclaw_ver: "2026.3.8"
fixed_ver: "2026.3.11"
---

> 📦 **OpenClaw 版本**：`2026.3.8` → `2026.3.11`　**分类**：bug　**发布**：2026-03-12

## 🔍 现象

升级到 OpenClaw 2026.3.7 / 3.8 后，kimi-coding (k2p5) 模型不再正确执行工具调用：
- Agent 有时直接回复"我已完成"而未实际执行任何工具
- 有时在聊天窗口中打印出 `exec({...})` 或 `<function_calls>` 的原始文本
- Agent 陷入无限循环（以为调用了工具，实际上没有）
- 错误日志出现：`Unexpected event order, got message_start before receiving message_stop`


## 🧠 根本原因

OpenClaw 2026.3.7 在 kimi-coding 适配层引入了回归（regression）：

1. **工具 payload 格式错误**：kimi-coding 使用 Anthropic Messages 协议，
   但 3.7 版本错误地向 kimi API 发送了 OpenAI 格式的 tool schema，
   导致 kimi 返回的工具调用标记被 OpenClaw 误解析为普通文本。

2. **临时补丁副作用**：之前配置了 `compat.requiresOpenAiAnthropicToolPayload: true`
   作为临时绕过，但在 3.7 引入新的解析路径后，该标志反而触发了错误的分支。

官方修复：PR #40008 `fix(kimi-coding): use native Anthropic tool schema instead of OpenAI`


## 💥 影响范围

- 所有使用 kimi-coding (k2p5) 模型的 agent 对话均受影响
- Cron 定时任务（如 Alex 英语话题）连续 timeout 10 次，每次卡 10 分钟
- 主通道（lane=main）等待超过 4 分钟（waitedMs=236042）
- 系统整体响应显著变慢


## ✅ 解决步骤

### 方案：升级到 2026.3.11（已包含 PR #40008 修复）

**步骤一：备份配置**
```bash
cp -r ~/.openclaw ~/.openclaw-backup-$(date +%Y%m%d_%H%M%S)
```

**步骤二：停止服务**
```bash
openclaw gateway stop
```

**步骤三：升级**
```bash
npm update -g openclaw
openclaw --version   # 验证：应显示 2026.3.11
```

**步骤四：移除过时的 compat 标志**

编辑 `~/.openclaw/agents/main/agent/models.json`（以及其他 agent 的同名文件），
找到 kimi-coding provider 下的 k2p5 model，删除 `compat` 字段：

```json
// 删除这段（旧的临时补丁，升级后不再需要）：
"compat": {
  "requiresOpenAiAnthropicToolPayload": true
}
```

**步骤五：重启服务**
```bash
openclaw gateway install
```


## 🔎 验证方法

```bash
# 1. 确认版本
openclaw --version
# 期望输出：OpenClaw 2026.3.11

# 2. 确认 compat 标志已移除
grep -r 'requiresOpenAiAnthropicToolPayload' ~/.openclaw/agents/*/agent/models.json
# 期望：无输出

# 3. 确认服务正常（HTTP 200）
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:18789/
# 期望：200

# 4. 发一条带工具调用的消息，观察 agent 是否实际执行（不再打印原始文本）
```


## 💬 排查过程中使用的提示词

### Step 1：搜索版本冲突信息 （search）

**提示词 / 命令**
```
WebSearch: "kimi coding plan 3.8 conflict version issue 2026"
```

**结果摘要**

找到 kimi coding plan 基本信息，未直接命中 3.8 冲突，继续细化搜索。

### Step 2：精确搜索 openclaw kimi 冲突 （search）

**提示词 / 命令**
```
WebSearch: "openclaw 3.8 kimi coding plan tool_call 泄露 function_calls 文本输出 解决"
```

**结果摘要**

命中知乎文章和 GitHub issue #39907、#39839，确认是 3.7 引入的 regression。官方 PR #40008 已修复，最新版为 2026.3.11。

### Step 3：检查本地版本和配置 （diagnosis）

**提示词 / 命令**
```
openclaw --version
cat ~/.openclaw/agents/main/agent/models.json | grep -A5 'kimi-coding'
npm info openclaw version
```

**结果摘要**

本地版本 2026.3.8（有问题），最新版 2026.3.11（含修复）。models.json 中存在 `requiresOpenAiAnthropicToolPayload: true` 旧补丁。

### Step 4：备份 + 停服 + 升级 （fix）

**提示词 / 命令**
```
cp -r ~/.openclaw ~/.openclaw-backup-20260312_185520
openclaw gateway stop
npm update -g openclaw
```

**结果摘要**

备份 797M 完成，升级成功到 2026.3.11。

### Step 5：移除 compat 标志 （fix）

**提示词 / 命令**
```
# 编辑 models.json，删除 kimi-coding model 下的 compat 字段：
# 删除：
#   "compat": {
#     "requiresOpenAiAnthropicToolPayload": true
#   }
# 适用于：main 和 jake 两个 agent
```

**结果摘要**

main/agent/models.json 和 jake/agent/models.json 均已去除旧标志。

### Step 6：验证服务恢复 （verify）

**提示词 / 命令**
```
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:18789/
tail -5 ~/.openclaw/logs/gateway.log
```

**结果摘要**

HTTP 200，日志显示 6 个飞书工具注册正常，ws client ready。

---

*本文由 OpenClaw 知识库自动生成，信息已脱敏处理。*
*OpenClaw 版本：2026.3.8*