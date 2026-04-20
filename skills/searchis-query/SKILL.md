---
name: searchis-query
description: >
  在内部金融调研档案（调研纪要、电话会记录、行业分析）中进行语义搜索。
  输入一个自然语言的研究问题，返回匹配该问题的原文证据列表（带来源文件标题、
  作者、日期、行号）。理解语义不是关键词匹配 — 问题描述越具体（研究对象、时间、
  想了解的维度、已有的假设），返回的证据越精准。每次调用约 60 秒。
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
npx @openduo/searchis query "<研究问题描述>" --json --timeout 180
```

输入是一个自然语言问题，可以是简短的，也可以是带上下文的长描述。Searchis 的内部 agent 会理解问题、拆解成内部搜索策略、检索相关文档、提取原文片段。

## 输出格式

```json
{
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
- `evidences: []` — 未找到相关证据（有效结果，不要伪造）

## 错误处理

| 返回 | 处理 |
|------|------|
| `{"error":"not_activated",...}` | 显示激活引导 |
| `{"error":"query_failed","message":"...401..."}` | 提示 `npx @openduo/searchis refresh` |
| `{"error":"query_failed","message":"...429..."}` | 等待 30s 重试 |
| 超时 (180s) | 报告超时，建议用更精确的查询重试 |

## 使用证据时

- 引用 `quote` 字段原文，不改写
- 标注来源 `[内部调研 {title} {date}]`
- 空结果时如实报告，不伪造
