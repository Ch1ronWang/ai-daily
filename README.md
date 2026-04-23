# AI Daily - AI 一手资讯日报

基于 Claude Code 的并行采集 Skill，自动采集 8 个信息源生成中文 AI 日报。

## 信息源

| 信息源 | 采集方式 | 内容 |
|--------|---------|------|
| AI Builders | HTTP fetch feed | 25 位核心 AI builder 推文 |
| Hacker News | WebFetch API | AI 相关热门文章 |
| HuggingFace | WebFetch API | Trending 模型 + Daily Papers |
| GitHub | gh CLI | AI trending repos |
| Reddit | RSS feed | 7 个 AI 相关 subreddit |
| Anthropic/Claude 博客 | WebFetch 页面 | 官方工程博客和产品博客 |
| RSS 博客 | WebFetch RSS | TechCrunch / Google AI / Microsoft AI |
| AI 播客 | HTTP fetch feed | Latent Space / No Priors 等 6 个播客 |

## 架构

```
主Agent（cron 触发）
├── spawn 8 个子采集 Agent（并行）
├── 等待所有子任务完成（15分钟超时）
├── 汇总生成中文日报
└── 存档到 Obsidian + 推送摘要
```

## 文件结构

```
ai-daily/
├── SKILL.md                          # Skill 配置和架构定义
├── agents/
│   ├── orchestrator.md               # 主汇总 Agent
│   └── collectors/
│       ├── builders.md               # AI Builder 推文采集
│       ├── podcasts.md               # AI 播客采集
│       ├── blogs.md                  # Anthropic/Claude 博客采集
│       ├── reddit.md                 # Reddit 采集
│       ├── github.md                 # GitHub 采集
│       ├── huggingface.md            # HuggingFace 采集
│       ├── hackernews.md             # Hacker News 采集
│       └── rss.md                    # RSS 博客采集
└── templates/
    ├── archive_template.md           # 存档模板
    └── push_summary_template.md      # 推送摘要模板
```

## 使用

### 手动触发
在 Claude Code 中说："执行 AI 日报采集"

### 定时触发
配置 Claude Code cron：每天 8:00 AM (Asia/Shanghai)

## 输出

- **存档**：`E:\Obsidian\AIFirst\AI-Daily\YYYY-MM-DD-HHMMSS.md`
- **推送**：Telegram / 对话摘要

## 数据源依赖

- AI Builders 和 AI 播客数据来自 [follow-builders](https://github.com/zarazhangrui/follow-builders) 项目
- Anthropic/Claude 博客直接抓取官网
