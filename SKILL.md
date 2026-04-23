---
name: ai-daily
description: 采集 AI 一手资讯并生成中文日报。当用户要求查看 AI 新闻、AI 日报、AI 动态时触发，也由 cron 定时任务自动触发。信息源：Reddit、GitHub、HuggingFace、Hacker News、RSS 博客、AI Builder 推文、AI 播客、Anthropic/Claude 博客。输出按平台分块，每条含标题+中文总结+关键词+链接。
---

# AI Daily - AI 一手资讯日报（CC 并行版）

## 架构

```
主Agent（cron 触发）
├── spawn 8 个子采集 Agent（并行）
│   ├── collector-reddit
│   ├── collector-github
│   ├── collector-huggingface
│   ├── collector-hackernews
│   ├── collector-rss
│   ├── collector-builders   ← 25 位 AI builder 推文
│   ├── collector-podcasts   ← AI 播客（Latent Space 等）
│   └── collector-blogs      ← Anthropic/Claude 博客
├── 等待所有子任务完成（TaskOutput 轮询，15分钟超时）
├── 汇总生成日报
│   ├── LLM 生成一句话摘要（推送用）
│   └── LLM 生成中文总结（存档用）
└── 存档存入 Obsidian + 推送摘要到对话
```

## 执行流程

### Step 1: 并行 spawn 子采集 Agent

对每个信息源，使用 Agent tool 并行 spawn 子任务：

```
Agent(
  task="采集 Reddit MachineLearning 等 subreddit 热帖，RSS 抓取...",
  agent="collector-reddit",
  run_in_background=true
)
```

### Step 2: 等待子任务完成

轮询检查每个子任务的状态：

```
start_time = now()
while (now() - start_time) < 780000:  # 13分钟
    for each task_id:
        result = TaskOutput(task_id, block=false)
        if result.status == "running":
            continue
    if all done: break
    sleep(30)
```

### Step 3: 重试失败的源

对状态为 failed 的源，最多重试 2 次。

### Step 4: 汇总生成

调用 orchestrator agent 汇总所有结果：
- 推送摘要：按 `C:\Users\wangc\.claude\projects\E--Workspace\ai-daily\templates\push_summary_template.md` 格式
- 存档：按 `C:\Users\wangc\.claude\projects\E--Workspace\ai-daily\templates\archive_template.md` 格式，存入 `E:\Obsidian\AIFirst\AI-Daily\YYYY-MM-DD-HHMMSS.md`

---

## 信息源配置

| 信息源 | 采集工具 | 采集范围 |
|--------|---------|---------|
| Reddit | PowerShell RSS | 每个 subreddit 最新 5 条 |
| GitHub | gh CLI | AI trending，最新 10 个 |
| HuggingFace | WebFetch API | Trending 模型 10 个 + Daily Papers 5 个 |
| Hacker News | WebFetch Firebase API | 热门 30 条，筛选 AI 相关 |
| RSS 博客 | WebFetch RSS | TechCrunch / Google AI / Microsoft AI 各 5 篇 |
| AI Builders | WebFetch feed | 25 位核心 AI builder 推文（Karpathy、Sam Altman 等） |
| AI 播客 | WebFetch feed | Latent Space / Training Data / No Priors 等 6 个播客 |
| Anthropic/Claude 博客 | WebFetch 页面 | Anthropic Engineering + Claude Blog 最新文章 |

## 超时配置

- 子任务：每个最多 5 分钟
- 主 Agent 等待循环：最多 15 分钟
- 汇总生成：2 分钟
- **全局总超时：18 分钟**

## Cron 配置

```json
{
  "schedule": { "kind": "cron", "expr": "0 8 * * *", "tz": "Asia/Shanghai" },
  "payload": {
    "kind": "agentTurn",
    "message": "执行 AI 日报采集。spawn 子任务并行采集 Reddit、GitHub、HuggingFace、Hacker News、RSS 博客、AI Builders、AI 播客、Anthropic/Claude 博客，主 Agent 等待所有子任务完成后汇总生成中文日报。文件名格式：YYYY-MM-DD-HHMMSS.md，避免覆盖同一天的多次采集。",
    "timeoutSeconds": 900,
    "model": "minimax/MiniMax-M2.7"
  }
}
```

---

## 相关文件

- `agents/orchestrator.md` - 主汇总 Agent
- `agents/collectors/reddit.md` - Reddit 采集子 Agent
- `agents/collectors/builders.md` - AI Builder 推文采集（替换原 collector-x）
- `agents/collectors/podcasts.md` - AI 播客采集
- `agents/collectors/blogs.md` - Anthropic/Claude 博客采集
- `C:\Users\wangc\.claude\projects\E--Workspace\ai-daily\templates\push_summary_template.md` - 推送摘要模板
- `C:\Users\wangc\.claude\projects\E--Workspace\ai-daily\templates\archive_template.md` - 存档模板
