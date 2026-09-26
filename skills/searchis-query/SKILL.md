---
name: searchis-query
description: >
  在内部金融调研档案（调研纪要、电话会记录、行业分析）中进行语义搜索。
  输入一个自然语言的研究问题，返回匹配该问题的原文证据列表（带来源文件标题、
  作者、日期、行号）。理解语义不是关键词匹配 — 问题描述越具体（研究对象、时间、
  想了解的维度、已有的假设），返回的证据越精准。每次调用约 60-180 秒。
tools: Bash
---

# Searchis 证据检索

语义搜索工具，在内部金融调研档案中根据自然语言问题返回原文证据。

## 前置检查

```bash
npx @openduo/searchis status --json
```

根据 `activated` 字段决定下一步。

### 未激活 → 引导用户

输出以下信息并**停止**：

```
⚠️ Searchis 内部文档库未激活。

Searchis 提供内部金融调研纪要、电话会记录和行业分析的语义搜索服务。

激活步骤：
1. 联系管理员获取邀请码
2. 运行: npx @openduo/searchis activate <邀请码>
```

## 调用

```bash
npx @openduo/searchis query "<研究问题描述>"
```

输入是一个自然语言问题，可以是简短的，也可以是带上下文的长描述。Searchis 的内部 agent 会理解问题、拆解成内部搜索策略、检索相关文档、提取原文片段。

调用通常需要 60-180 秒完成（agent 会执行多轮内部搜索）。CLI 使用流式协议持续接收结果，stderr 上有持续的进度输出表明 agent 正在工作。遇到网络或后端真的卡住才会中断（默认 dial 30s、idle 60s），不要额外包装超时。

需要程序解析时加 `--json` 切换成 JSON 输出（默认是带引用的 markdown）。

## 输出（默认 markdown）

主要内容是带 `[1]` `[2]` 等内联引用的研究答案，下接 `Sources` 区列出每条证据的标题、作者、日期、行号、原文。引用编号是文件内的下标，对接给用户时用 `[内部调研 {title} {date}]` 这类可读形式。

### `--json` 输出

```json
{
  "answer": "## 标题\n... [1] ... [2] ...",
  "evidences": [
    {
      "id": "abc123",
      "date": "2026-03-27",
      "title": "光芯片专家交流-2026年EML、CW光源全球供给情况...",
      "source": { "author": "Shirley" },
      "quote": "原文引用，未经改写",
      "lineRange": [40, 45],
      "context": "该证据的一句话描述"
    }
  ],
  "stats": { "duration_ms": 65000, "files_searched": 4 }
}
```

## 字段说明

- `quote` — 从原始文档精确复制的文本，未经 LLM 改写
- `date` — 从文件名提取的文档日期
- `title` — 调研纪要的标题（来自 frontmatter）
- `source.author` — 纪要作者
- `id` — 文件 hash，同文件的多条证据共享
- `context` — 对该证据的一句话摘要
- `answer` — 服务内部对证据的草稿组织，**仅作参考线索**：可能漏关键证据、可能误读，最终结论由调用方基于 evidences 自行整理
- `evidences: []` — 未找到相关证据（有效结果，不要伪造）
- `source.contributor` / `source.ingestedAt` — 仅当证据来自**用户上传**的资料时出现：`contrib-xxxxxxxx` 是上传者的稳定假名，`ingestedAt` 是收录时间（UTC）。`source.publishTimeIso` / `source.publishTimePrecision` 是原文时间及其精度（day/minute/second）。引用这类证据时标注"用户贡献"，与内部调研纪要区分开

## 错误处理

| 返回 | 处理 |
|------|------|
| `{"error":"not_activated",...}` | 显示激活引导 |
| `{"error":"query_failed","message":"...401..."}` | 提示 `npx @openduo/searchis refresh` |
| `{"error":"query_failed","message":"...429..."}` | 等待 30s 重试 |
| dial / idle timeout（网络或后端卡死） | 报告超时，建议用更精确的查询重试 |

## 使用证据时

- 引用 `quote` 字段原文，不改写
- 标注来源 `[内部调研 {title} {date}]`；用户贡献的证据标注 `[用户贡献 {contributor} {title} {date}]`
- 空结果时如实报告，不伪造

## 上传资料（需要上传权限）

只有被 root 授予上传权限的账号可以上传。上传后资料进入**所有用户共享**的检索库，**只能新增，不能修改或删除**，所以务必先 `--dry-run`。

### 流程

1. 查看字段规范（以服务器返回为准）：`npx @openduo/searchis upload --schema`
2. 把资料整理成一个 markdown 文件，开头是 YAML frontmatter：

```markdown
---
title: "原文标题"
publish_time: "2026-09-24T23:48:55+08:00"
original_target: [MongoDB, Snowflake]
institution: "来源机构"
author: "作者或专家"
source_url: "https://…"          # 可选
---
正文……

![营收图](figs/revenue.png)
```

3. 预检：`npx @openduo/searchis upload doc.md --dry-run`，按返回的逐条问题修正，直到通过
4. 正式上传：`npx @openduo/searchis upload doc.md`

### 必须遵守

- **`publish_time` 是原文时间，必须带时区**：`2026-09-25+08:00`（只到日）、`2026-09-25T08:00+08:00`（到分）、`2026-09-25T08:00:00Z`（到秒）。原文没写时区时，**问你的用户，不要猜**；原文只有日期就只填日期，不要编时刻。收录时间由服务器自动记录。
- **`original_target` 只列原文实际点名的公司**，不要推测补充；一家都没点名就写 `[]`。
- **图片只能引用随文件一起的本地图片**（相对 md 文件的路径，文件名只用 `A-Za-z0-9._/-`）；不允许网络图片、`<img>` 标签、图片标题语法。CLI 会自动把引用到的本地图片一起上传。
- 不要在正文里写 frontmatter 以外的元数据；`contributor` / `ingested_at` 由服务器写入，自己填会被拒绝。
- 上传失败后重试要复用 stderr 打印的 `--idempotency-key`，避免重复入库（系统不去重）。

### 上传后

- `search` 立即可搜；`hybrid_search` 最多 30 分钟后可搜；wiki 由后台异步吸收。
- 错误 `stored_not_linked`：已持久化，稍后自动可搜，**不要重传**。
