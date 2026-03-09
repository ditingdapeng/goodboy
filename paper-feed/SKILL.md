---
name: paper-feed
description: "Agent 方向顶会论文推送引擎。从 OpenReview（ICLR 2026、NeurIPS 2025）和 arXiv 获取最新被接收的 Agent 系统/应用设计方向论文，智能筛选后推荐一篇未读论文。配合 ljg-paper（精读）和 so-send-message（推送到群）使用。当用户说「推荐论文」「paper-feed」「获取论文」「论文推送」「每日论文」「agent 论文」时触发，也在定时任务中自动触发。即使用户只是模糊地提到想看最新的 agent 论文或顶会论文，也应触发此 skill。"
user_invocable: true
version: "1.0.0"
---

# paper-feed: Agent 论文推送引擎

你是一个论文推荐引擎。你的工作是从顶会和预印本中找出一篇高质量的 Agent 系统/应用设计方向论文，供后续精读和推送使用。

## 为什么需要这个 skill

研究 Agent 系统的人需要持续跟踪最新进展，但顶会论文数量庞大（仅 ICLR 2026 就有 400+ 篇 agent 相关论文），手动筛选耗时且容易遗漏。这个 skill 自动完成"获取 → 筛选 → 去重 → 推荐"的全流程，每次返回一篇值得精读的论文。

## 执行流程

### 1. 获取论文

用 Bash 调用 API 获取候选论文池。因为 OpenReview API 在某些网络环境下可能不通，优先使用 Bash+curl：

**OpenReview（ICLR 2026 + NeurIPS 2025）**：

```bash
curl -s --max-time 30 "https://api2.openreview.net/notes/search?query=agent&group=ICLR.cc/2026/Conference&limit=100"
curl -s --max-time 30 "https://api2.openreview.net/notes/search?query=agent&group=NeurIPS.cc/2025/Conference&limit=100"
```

从返回的 JSON 中提取被接收论文：
- `notes[].forumContent.title.value` → 标题
- `notes[].forumContent.abstract.value` → 摘要
- `notes[].forumContent.venue.value` → 如 "ICLR 2026 Poster"（只保留含 Poster/Spotlight/Oral 的）
- `notes[].forumContent.venueid.value` → 排除含 "Rejected" 的
- `notes[].forumContent.pdf.value` → PDF 路径，完整 URL 为 `https://openreview.net` + 该路径
- `notes[].forumContent.authors.value` → 作者列表
- `notes[].forum` → 论文 ID（用于去重）

如果 curl 也不通，用 WebFetch 访问同一 URL。

**arXiv（2026 年预印本）**：

```bash
curl -s --max-time 30 "https://export.arxiv.org/api/query?search_query=cat:cs.AI+AND+all:agent+system&sortBy=submittedDate&sortOrder=descending&max_results=50"
```

arXiv 返回 XML（Atom feed），从中提取 title、summary、id、published。只保留 published >= 2026-01-01 的。

**去重**：读取 `data/pushed.json`，排除已推送的论文 ID。也可运行辅助脚本：

```bash
python3 scripts/fetch_papers.py
```

### 2. 筛选论文

从候选池中，判断每篇论文是否属于 **Agent 系统/应用设计** 方向。

判断依据——看论文的核心贡献是什么：

**属于本方向（选入）**：
- 设计了 Agent 系统架构或框架（如多 Agent 协作平台、Agent 编排系统）
- 提出了 Agent 工具调用 / function calling 方案
- 实现了 Agent 在具体场景的应用落地（代码生成、Web 导航、数据分析等）
- 设计了 Agent 的规划 / 记忆 / 反思机制作为系统组件

**不属于本方向（排除）**：
- 核心贡献是模型训练方法（RL、RLHF、fine-tuning、对齐）
- 核心贡献是提出新的评测基准或数据集
- 使用了 "agent" 一词但实际是博弈论/经济学/强化学习中的 agent 概念
- 核心贡献是注意力机制、模型压缩等算法改进

对每篇候选论文，阅读标题和摘要后做出判断。不需要逐篇输出判断过程，只需筛选出符合条件的论文。

### 3. 选择一篇推荐

从筛选后的论文中选一篇推荐。优先策略：
1. 优先选择尚未推送过的论文
2. 在未推送论文中，优先选择来自 Oral > Spotlight > Poster > arXiv 的
3. 同等条件下，选择系统设计创新性更强的

### 4. 记录已推送

选定论文后，将其 ID 追加到 `data/pushed.json`：

```bash
python3 scripts/mark_pushed.py "<paper_id>"
```

### 5. 输出格式

以如下格式输出推荐结果：

```
## 推荐论文

**标题**: {title}
**作者**: {authors}
**来源**: {venue}（如 ICLR 2026 Poster）
**PDF**: {pdf_url}

### 摘要
{abstract}

### 推荐理由
{一句话说明为什么这篇论文属于 Agent 系统设计方向，以及它的核心贡献}
```

## 与其他 skill 的协作

这个 skill 的输出是整个论文推送流水线的第一环：

1. **paper-feed**（本 skill）→ 推荐一篇论文，输出标题和 PDF 链接
2. **ljg-paper** → 接收 PDF 链接，执行精读 pipeline（拆→榨→白话→费曼→博导审稿）
3. **so-send-message** → 将精读结果推送到群 群号

定时任务触发时，依次调用这三个 skill 即可完成全流程。

## 数据文件

- `data/pushed.json`：已推送论文记录，格式为 `{"pushed": ["id1", "id2", ...]}`
- 如果文件不存在，脚本会自动创建

## 约束

- 只推送 2026 年 1 月 1 日之后的论文
- 论文池有限（约 400-500 篇），合理控制推送节奏
- OpenReview API 无需认证，arXiv API 无需认证，均为免费公开接口
