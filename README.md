# lazy-memfree

[![Release](https://github.com/microlazy-apps/lazy-memfree/actions/workflows/release.yml/badge.svg)](https://github.com/microlazy-apps/lazy-memfree/actions/workflows/release.yml)

懒猫微服 (LazyCat) 上的 [MemFree](https://github.com/memfreeme/memfree)
自托管 Hybrid AI 搜索引擎。

## 一键安装

到 [Releases](https://github.com/microlazy-apps/lazy-memfree/releases)
下载最新的 `.lpk`，在 LazyCat 应用商店中安装。

## 功能

把笔记 / 书签 / 文档 / 实时网页搜索合并成一个 AI 助手：

- **私有知识库** — 上传 PDF / Word / Markdown / 文本，向量化检索
- **联网搜索** — 通过 [Serper](https://serper.dev) 实时检索
- **多模型** — OpenAI / Claude / Gemini / DeepSeek / Qwen / OpenRouter
- **多输入** — 文本、图片、文件、网页
- **多输出** — 文本、思维导图、AI 图片

## 配置

安装时一次性填写：

| 参数 | 说明 |
| --- | --- |
| `OPENAI_API_KEY` | **必填**。用于嵌入向量与默认模型。可填官方 key 或任何 OpenAI 兼容服务的 key |
| `OPENAI_BASE_URL` | OpenAI 兼容端点 URL，默认 `https://api.openai.com/v1` |
| `SERPER_API_KEY` | 推荐。Google 网络搜索 |
| `ANTHROPIC_API_KEY` | 可选，启用 Claude |
| `DEEPSEEK_API_KEY` | 可选，启用 DeepSeek |
| `QWEN_API_KEY` | 可选，启用通义千问 / 阿里云百炼 |
| `OPENROUTER_API_KEY` | 可选，启用 OpenRouter 聚合 |
| `GOOGLE_GENERATIVE_AI_API_KEY` | 可选，启用 Gemini |
| `JINA_KEY` | 可选，启用 Jina rerank |
| `AUTH_GITHUB_*` | 可选，启用 GitHub 登录 |
| `AUTH_GOOGLE_*` | 可选，启用 Google 登录 |

## 数据持久化

| 路径 | 内容 |
| --- | --- |
| `/lzcapp/var/data/lancedb` | LanceDB 向量库（用户上传的文档） |
| `/lzcapp/var/redis` | Redis 数据（用户、会话、搜索历史） |
| `/lzcapp/var/logs` | 应用日志 |

## 上游

- 上游项目: <https://github.com/memfreeme/memfree>
- 本仓库: <https://github.com/microlazy-apps/lazy-memfree>

## 许可证

[MIT](./LICENSE) — 跟随上游。
