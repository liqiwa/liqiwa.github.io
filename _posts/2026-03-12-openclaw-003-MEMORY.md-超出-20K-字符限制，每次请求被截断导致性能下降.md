---
title: "MEMORY.md 超出 20K 字符限制，每次请求被截断导致性能下降"
date: 2026-03-12
last_modified_at: 2026-03-12
categories:
  - OpenClaw
  - 排障
tags:
  - memory
  - performance
  - bootstrap
  - rag
  - truncation
toc: true
toc_label: 目录
toc_icon: list-ul
openclaw_ver: "2026.3.8"
---

> 📦 **OpenClaw 版本**：`2026.3.8`　**分类**：performance　**发布**：2026-03-12

## 🔍 现象

错误日志持续出现：
```
[agent/embedded] workspace bootstrap file MEMORY.md is 35359 chars (limit 20000);
truncating in injected context
```
- Agent 响应变慢
- Agent 经常"记不住"之前记录过的信息（后半段被截断）
- 每次 cron task 和对话都要执行截断处理，增加额外开销


## 🧠 根本原因

OpenClaw 的 `MEMORY.md` 是"workspace bootstrap file"——每次 agent 启动时，
会将该文件内容直接注入到上下文中，上限 20,000 字符。

设计意图：
- MEMORY.md = 核心常驻信息（用户档案、偏好）
- memory/*.md = 每日日志，通过 RAG 搜索访问

实际问题：
- Agent 每次更新记忆时，把新的历史记录直接追加进 MEMORY.md
- 经过数月积累，文件从 7KB 膨胀至 69KB（35,000+ 字符）
- OpenClaw 没有自动压缩机制，只会截断，且截断后的内容完全丢失


## 💥 影响范围

- MEMORY.md 实际只有前 20,000 字符被使用，后半段（63,270 字符）完全被丢弃
- 每次请求（包括 17 个每半小时触发的 cron 任务）都执行截断，增加处理耗时
- Agent 的长期记忆实际上已经失效（历史记录不可用）


## ✅ 解决步骤

### 原则：MEMORY.md 只保留核心档案，历史记录归档到 memory/ 子目录

**步骤一：确认切割点**
```bash
grep -n '^## 近期技术合作进展' ~/.openclaw/workspace/MEMORY.md
# 找到历史记录从哪一行开始（本例为第 186 行）
```

**步骤二：备份完整版本**
```bash
cp ~/.openclaw/workspace/MEMORY.md    ~/.openclaw/workspace/MEMORY.md.full-$(date +%Y%m%d)
```

**步骤三：截断 MEMORY.md，只保留核心档案**
```bash
head -185 ~/.openclaw/workspace/MEMORY.md > /tmp/MEMORY_trimmed.md

# 加说明注释
echo "" >> /tmp/MEMORY_trimmed.md
echo "---" >> /tmp/MEMORY_trimmed.md
echo "> 历史记录已迁移至 memory/YYYY-MM-DD.md，可通过 memory search 检索。"      >> /tmp/MEMORY_trimmed.md

cp /tmp/MEMORY_trimmed.md ~/.openclaw/workspace/MEMORY.md
wc -m ~/.openclaw/workspace/MEMORY.md   # 应 < 20000
```

**步骤四：重建 memory 搜索索引**
```bash
openclaw memory index --force
openclaw memory status
```

**步骤五：验证历史记录仍可搜索**
```bash
openclaw memory search "GitHub协作项目"
# 应能从 memory/2026-02-12.md 等文件中找到结果
```


## 🔎 验证方法

```bash
wc -m ~/.openclaw/workspace/MEMORY.md
# 期望：< 20000

openclaw memory search "某个历史关键词"
# 期望：从 memory/*.md 中返回结果，score > 0.5
```

日志中应不再出现：
```
workspace bootstrap file MEMORY.md is XXXXX chars (limit 20000); truncating
```


## 💬 排查过程中使用的提示词

### Step 1：发现截断报错 （diagnosis）

**提示词 / 命令**
```
grep "truncating\|MEMORY.md" ~/.openclaw/logs/gateway.err.log | tail -5
```

**结果摘要**

确认 MEMORY.md 35,359 字符，超出 20,000 上限，每次都被截断。

### Step 2：分析文件结构 （diagnosis）

**提示词 / 命令**
```
wc -l ~/.openclaw/workspace/MEMORY.md
# 检查 workspace 目录有无 memory/ 子目录
ls ~/.openclaw/workspace/memory/ | head -5
# 精确计算可切割部分的字符数
tail -n +185 ~/.openclaw/workspace/MEMORY.md | wc -m
```

**结果摘要**

1552 行，memory/ 子目录有 35 个日志文件已被 RAG 索引。第 185 行后为历史记录，共 63,270 字符完全多余。

### Step 3：截断并重建索引 （fix）

**提示词 / 命令**
```
# 备份
cp ~/.openclaw/workspace/MEMORY.md ~/.openclaw/workspace/MEMORY.md.full-20260312

# 截断至第 185 行
head -185 ~/.openclaw/workspace/MEMORY.md > /tmp/MEMORY_trimmed.md
cp /tmp/MEMORY_trimmed.md ~/.openclaw/workspace/MEMORY.md

# 重建 RAG 索引
openclaw memory index --force
```

**结果摘要**

MEMORY.md 从 35,359 字符压缩至 7,498 字符，索引重建 35/35 文件正常。

### Step 4：验证历史记录仍可搜索 （verify）

**提示词 / 命令**
```
openclaw memory search "GitHub协作项目"
```

**结果摘要**

score=0.666，从 memory/2026-02-12.md 中找到目标内容，RAG 正常。

---

*本文由 OpenClaw 知识库自动生成，信息已脱敏处理。*
*OpenClaw 版本：2026.3.8*
