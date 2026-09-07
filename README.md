# 劳动法规 RAG 知识库问答机器人（n8n + Milvus + DeepSeek）

一个**端到端**的 RAG（检索增强生成）知识库问答系统：把劳动法规 PDF 自动解析、分块、向量化存入 Milvus，再通过钉钉企业机器人实现群内 @ 提问、检索条文、生成带引用的回答。

> 从 PDF 入库到钉钉群问答，全链路零代码编排，除模型 API 外全部组件本地运行。

## 架构

```
【入库流水线】
劳动法 PDF → Read Files → Extract from File → Milvus Insert
                                              ├─ Embeddings: BAAI/bge-m3 (SiliconFlow)
                                              └─ Default Data Loader

【问答链路（钉钉生产版）】
钉钉群 @机器人 → DingTalk Trigger (Stream 长连接，免公网 IP)
             → Edit Fields (提取 question + sessionId)
             → AI Agent ─┬─ Chat Model: DeepSeek (deepseek-v4-flash)
                         ├─ Memory: Simple/Window Buffer Memory (按群会话隔离，支持多轮追问)
                         └─ Tool: Milvus Vector Store (As Tool for AI Agent, Top-4 检索)
                                    └─ Embeddings: BAAI/bge-m3
             → HTTP Request (POST sessionWebhook，以机器人身份回复到群)

【问答链路（调试版）】
Chat Trigger → Question and Answer Chain → Retriever → Milvus（n8n 内置聊天面板直接测试）
```

## 技术栈

| 组件 | 选型 | 说明 |
|---|---|---|
| 工作流编排 | n8n 2.36.9（npm 自托管） | 可视化编排，LangChain 节点生态 |
| 向量数据库 | Milvus 2.4.15（Docker Compose 单机版） | etcd + MinIO + Milvus 三容器 |
| Embedding | BAAI/bge-m3（1024 维，SiliconFlow API） | OpenAI 兼容接口接入 |
| 生成模型 | DeepSeek deepseek-v4-flash | 需关闭 thinking 模式（见踩坑记录） |
| IM 集成 | 钉钉企业内部应用 + Stream 模式 | 长连接接收 @ 消息，sessionWebhook 回复 |
| 可视化 | Attu 2.2.6 | Milvus 图形界面，验证集合与数据 |

## 仓库结构

```
├── workflows/                          # n8n 工作流 JSON（可直接导入）
│   ├── 1_ingest_pdf_to_milvus.json    # PDF 入库：解析 → 分块 → 向量化 → 写入 Milvus
│   ├── 2_qa_chat.json                 # 问答调试版：Q&A Chain + Retriever
│   └── 3_dingtalk_agent_qa.json       # 问答生产版：钉钉触发 + AI Agent + 记忆 + 工具检索
├── milvus/
│   └── docker-compose.yml             # Milvus 单机版部署（数据卷可通过 DOCKER_VOLUME_DIRECTORY 指定盘符）
└── docs/
    └── 主线手册.md                     # 完整搭建/配置/踩坑记录
```

## 快速开始

### 1. 启动 Milvus

```bash
cd milvus
docker compose up -d          # 等 3 个容器全部 Up (healthy)
docker run -d --name attu -p 3000:3000 -e MILVUS_URL=host.docker.internal:19530 attu/attu:v2.2.6  # 可选：可视化界面
```

### 2. 启动 n8n

```bash
npm install -g n8n            # 需要 Node.js >= 24
n8n start                     # 浏览器访问 http://localhost:5678
```

Windows 下如需读取本地 PDF，先给 n8n 配置文件系统白名单（永久环境变量）：

```powershell
[Environment]::SetEnvironmentVariable("N8N_RESTRICT_FILE_ACCESS_TO_CUSTOM_DIRS", "true", "User")
[Environment]::SetEnvironmentVariable("N8N_FILESYSTEM_ALLOWED_DIRECTORIES", "D:\rag-docs", "User")
```

### 3. 导入工作流

n8n 画布右上角菜单 → **Import from File**，依次导入 `workflows/` 下的 3 个 JSON，然后：

- 创建凭证：DeepSeek API Key、SiliconFlow API Key（用 OpenAI 类型凭证，Base URL 填 `https://api.siliconflow.cn/v1`）、Milvus（Host `127.0.0.1` / Port `19530`）
- 按工作流内节点提示把凭证挂回去
- 先跑 `1_ingest_pdf_to_milvus` 入库，再激活 `3_dingtalk_agent_qa`

### 4. 钉钉机器人（生产版需要）

1. [open-dev.dingtalk.com](https://open-dev.dingtalk.com) 创建**企业内部应用**，添加机器人能力，消息接收模式选 **Stream**（长连接，本地服务免公网 IP），发布应用并把机器人加进群
2. n8n 中安装社区节点 `@cryozerolabs/n8n-nodes-dingtalk`，用 Corp ID / Client ID / Client Secret 建凭证
3. 在群里 @机器人 提问即可

## 实现效果

- 群内 @机器人 问「劳动合同签几年？」→ 数秒内收到基于法条原文、注明条文依据的回答
- 支持多轮追问（同一群共享会话记忆，跨群记忆隔离）
- 检索结果携带 `loc` 行号元数据，可溯源到 PDF 原文位置

## 踩坑记录（真实排障经验）

| 坑 | 原因与解法 |
|---|---|
| n8n 报 `Access to the file is not allowed` | n8n 默认禁止节点读任意路径，需设置 `N8N_FILESYSTEM_ALLOWED_DIRECTORIES` 白名单并重启 |
| DeepSeek V4 + AI Agent 报 400（reasoning_content 未回传） | V4 系列默认开启思考模式，Agent 工具调用必须回传思维链而 n8n 原生节点不支持；用社区节点 `n8n-nodes-deepseek-v4-thinking-fix` 关闭 thinking 解决 |
| Milvus 连接 `14 UNAVAILABLE` | 容器还没 healthy 就测试了；`docker compose up -d` 后等状态变 healthy 再跑工作流 |
| 钉钉触发器收到消息但工作流无反应 | Trigger 节点未订阅具体事件类型，需在节点里选择「机器人接收消息」事件 |
| HTTP Request 发 JSON 报 `not valid JSON` | 请求体含中文/换行的动态文本时，用 `={{ JSON.stringify({...}) }}` 表达式整体构造请求体，而非字符串拼接 |
| 群里 @ 了机器人但没触发 | @ 的是 webhook 自定义机器人（只出不进），企业应用机器人和自定义机器人是两套通道 |

## License

MIT — 仅供学习交流。
