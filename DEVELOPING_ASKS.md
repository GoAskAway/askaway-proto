# 开发 asks 应用指南

基于 [askaway-doc](https://gitlab.huanqiu-ltd.com/platform/askaway-doc) 与 [asks-ts](https://github.com/GoAskAway/asks-ts) 参考实现，指导从零开发一个 AskService。

## 1. ACTR 核心概念

在开始开发前，必须理解 ACTR 框架的四个核心概念：

| 概念 | 说明 | 示例 |
| --- | --- | --- |
| **Realm** | 最高层级隔离边界，同一 Realm 下的 Actor 才能互相发现和通信 | `realm_id = 2368_266_035` |
| **Actor Type** | Actor 的"类"定义，由 `Manufacturer` + `Name` 唯一标识 | `askaway1+AskService` |
| **Proto** | Actor Type 的服务接口契约，定义 RPC 方法和消息格式 | `ask-service/ask.proto` |
| **Actor Node** | Actor Type 的实际运行实例（进程或容器） | asks-ts `npm run dev` |

```
Realm
 └── Actor Type (制造商: askaway1, 名称: AskService)
       └── Proto (ask.proto)
             └── Actor Node (运行中的 asks 进程)
```

**Askaway 平台约定**：

| 角色 | Manufacturer | Name | 说明 |
| --- | --- | --- | --- |
| 服务端 | `askaway` | `app_server` | asks 应用在平台上的注册类型 |
| 客户端 | `askaway` | `browser` | askc 客户端在平台上的注册类型 |

> `asks-ts` 的 `Actr.toml` 中 `actr_type = "askaway1+AskService"` 是独立开发/测试时使用的简化标识。正式接入 Askaway 平台时需改为 `askaway+app_server`。

## 2. 协议契约

### 2.1 Proto 定义

协议源文件位于 [askaway-proto](https://github.com/GoAskAway/askaway-proto) 仓库的 `ask-service/ask.proto`：

```protobuf
service AskService {
  rpc Prompt(UsrPromptRequest) returns (AssistantReply);
  rpc Attach(AttachRequest) returns (AttachResponse);
}
```

### 2.2 UsrPromptRequest — 统一提示请求

`Prompt` RPC 统一处理文本和语音两种问答模式：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `question_id` | string | 问题唯一标识 |
| `session_id` | string | 会话 ID |
| `text` | string | 文本问题（文本模式） |
| `voice_stream_id` | string | 语音流通道 ID（语音模式，客户端→服务端） |
| `location` | Location | 可选位置信息 |
| `attachment_ids` | repeated string | 可选附件 ID 列表 |
| `text_response_stream_id` | string | 文本响应流通道 ID（服务端→客户端） |
| `voice_response_stream_id` | string | 语音响应流通道 ID（服务端→客户端） |

### 2.3 AssistantReply — 即时响应

```protobuf
message AssistantReply {
  string question_id = 1;
  string session_id = 2;
  string text = 3;           // 即时文本回复，status_code != 0 时可为空
  int32 status_code = 4;     // 0 = 成功
  string error_message = 5;  // 错误信息
}
```

> `text` 字段用于返回即时简短回复。流式长回复通过 `sendDataStream` 经 `text_response_stream_id` 通道推送。

### 2.4 AttachRequest / AttachResponse

```protobuf
message AttachRequest {
  string id = 1;
  string filename = 2;
  AttachmentType type = 3;   // IMAGE=1, DOCUMENT=2, AUDIO=3, VIDEO=4, OTHER=99
  bytes data = 4;
}

message AttachResponse {
  string id = 1;
  int32 status_code = 2;     // 0 = 成功
  string error_message = 3;
}
```

## 3. 项目搭建

### 3.1 目录结构

```
my-asks/
├── Actr.toml                  # ACTR 配置
├── package.json               # 依赖 + scripts
├── tsconfig.json
├── askaway-proto/             # git submodule
├── scripts/
│   └── generate-generated.cjs # 代码生成脚本
└── src/
    ├── index.ts               # 入口
    ├── ask_service.ts         # 业务逻辑（开发者编辑）
    ├── ask_service_runtime.ts # 生成的 runtime
    └── generated/
        ├── ask.pb.ts          # proto 消息类型
        └── local.actor.ts     # dispatch 路由
```

### 3.2 初始化

```bash
git clone --recurse-submodules git@github.com:GoAskAway/asks-ts.git my-asks
cd my-asks
npm install
npm run codegen
```

### 3.3 Actr.toml 配置

```toml
edition = 1
exports = ["askaway-proto/ask-service/ask.proto"]

[package]
name = "my-ask-service"
description = "自定义 AskService"

[package.actr_type]
manufacturer = "askaway1"
name = "AskService"           # 开发阶段；平台接入时改为 askaway+app_server

[system.signaling]
url = "wss://actrix1.develenv.com/signaling/ws"

[system.deployment]
realm_id = 2368_266_035

[system.discovery]
visible = true                 # 必须为 true，客户端通过 discover 发现

[system.webrtc]
force_relay = false
stun_urls = ["stun:actrix1.develenv.com:3478"]
turn_urls = ["turn:actrix1.develenv.com:3478"]

[acl]
[[acl.rules]]
permission = "allow"
types = ["askaway1+Client", "acmeopenclaw+AskawayClient"]
```

> ACL 必须允许客户端 Actor Type，否则客户端无法连接。正式平台环境使用 `askaway+browser`。

## 4. 业务逻辑实现

唯一需要编辑的文件：`src/ask_service.ts`。实现 `AskServiceHandlers` 接口的两个方法。

### 4.1 prompt — 处理提示请求

**执行顺序（严格）**：

```
1. 收到 UsrPromptRequest
2. IF 有 voice_stream_id → ctx.registerStream(voiceStreamId, callback)   // 先注册语音流回调
3. 执行业务逻辑（LLM 推理、RAG、数据库查询……）
4. return AssistantReply                                                 // 返回即时响应
5. IF 有 text_response_stream_id → ctx.sendDataStream() 异步推送流式结果
```

**示例代码**：

```typescript
import type { Context, DataStream, StreamSignal } from '@actor-rtc/actr';
import type { Ask_AssistantReply, Ask_UsrPromptRequest } from './generated/ask.pb.js';
import type { AskServiceHandlers } from './ask_service_runtime.js';

export class AskServiceHandler implements AskServiceHandlers {
  async prompt(request: Ask_UsrPromptRequest, ctx: Context): Promise<Ask_AssistantReply> {
    // Step 1: 注册语音流回调（先注册，后返回）
    if (request.voiceStreamId) {
      await ctx.registerStream(request.voiceStreamId, (err, signal) => {
        if (err) {
          console.error(`Voice stream ${request.voiceStreamId} error:`, err);
          return;
        }
        if (!signal) {
          console.log(`Voice stream ${request.voiceStreamId} ended.`);
          return;
        }
        // signal.chunk.sequence — 序列号
        // signal.chunk.payload — Buffer（PCM 音频数据）
        this.handleAudioChunk(signal.chunk);
      });
    }

    // Step 2: 业务逻辑 — 调用你的 LLM/RAG/数据库
    const instantReply = await this.processQuery(request);

    // Step 3: 流式推送（异步，不阻塞 return）
    const targetActrId = ctx.callId(); // 必须在返回前同步获取
    const streamId = request.textResponseStreamId;
    if (targetActrId && streamId) {
      void this.streamResponse(targetActrId, streamId, request);
    }

    // Step 4: 返回即时响应
    return {
      questionId: request.questionId,
      sessionId: request.sessionId,
      text: instantReply,
      statusCode: 0,
      errorMessage: '',
    };
  }

  private async processQuery(request: Ask_UsrPromptRequest): Promise<string> {
    // TODO: 替换为实际 LLM 调用
    // const answer = await llm.chat(request.text, request.location, request.attachmentIds);
    return `Echo: ${request.text}`;
  }

  private async streamResponse(targetActrId: string, streamId: string, request: Ask_UsrPromptRequest) {
    try {
      // TODO: 替换为实际流式 LLM 调用
      const chunks = ['Hello ', 'from ', 'streaming ', 'response'];
      for (let i = 0; i < chunks.length; i++) {
        const dataStream: DataStream = {
          streamId,
          sequence: i + 1,
          payload: Buffer.from(chunks[i]),
          metadata: [],
        };
        await ctx.sendDataStream(targetActrId, dataStream);
      }

      // EOS: 发送流结束信号
      const eos: DataStream = {
        streamId,
        sequence: chunks.length + 1,
        payload: new Uint8Array(),
        metadata: [{ key: 'eos', value: 'true' }],
      };
      await ctx.sendDataStream(targetActrId, eos);
    } catch (err) {
      console.error(`Stream ${streamId} error:`, err);
    }
  }
}
```

### 4.2 attach — 处理附件上传

```typescript
async attach(request: Ask_AttachRequest, _ctx: Context): Promise<Ask_AttachResponse> {
  if (!request.id || !request.filename) {
    return { id: request.id || '', statusCode: 400, errorMessage: 'Missing id or filename' };
  }

  // TODO: 持久化到你的存储（S3、OSS、本地文件系统等）
  await saveToStorage(request.id, request.data, request.filename, request.type);

  return { id: request.id, statusCode: 0, errorMessage: '' };
}
```

## 5. 数据流通道

### 5.1 通道设计与类型隐式约定

Askaway 采用**专用通道 ID** 设计：每个 `streamId` 的用途由其在 Proto 中的字段语义决定，不依赖数据包内类型标记。

| 通道字段 | 方向 | 数据类型 | 注册方 |
| --- | --- | --- | --- |
| `text_response_stream_id` | 服务端→客户端 | 文本片段（UTF-8） | 客户端 |
| `voice_response_stream_id` | 服务端→客户端 | 合成语音（PCM/Opus） | 客户端 |
| `voice_stream_id` | 客户端→服务端 | 录音音频（PCM） | 服务端 |

> 接收端收到 `voice_stream_id` 标记的 Stream，无需查看数据包内部 MIME 类型，直接投递给 `AudioProcessor`。类型信息在通道建立时由 Proto 字段语义约定。

### 5.2 DataStream 结构

```typescript
interface DataStream {
  streamId: string;      // 流 ID，与 UsrPromptRequest 中的字段对应
  sequence: number;      // 序列号，从 1 严格递增
  payload: Uint8Array;   // 数据块
  metadata: Metadata[];  // 元数据，支持扩展信号
}
```

### 5.3 EOS（End of Stream）信号

采用**带内信令**方式标记流结束：

```typescript
// 发送端：最后一个数据包携带 eos 标记
const eosSignal: DataStream = {
  streamId,
  sequence: lastSequence + 1,   // 严格递增
  payload: new Uint8Array(),    // 可携带最后一个数据块
  metadata: [{ key: 'eos', value: 'true' }],
};
await ctx.sendDataStream(targetActrId, eosSignal);
```

**接收端处理**（客户端需实现）：
1. 按 `sequence` 确保有序处理
2. 有 payload → 处理业务数据
3. 遍历 metadata 查找 `key === 'eos'` 且 `value === 'true'` → 关闭流、释放资源
4. **必须先处理 payload，再关闭流**，确保最后一段数据不丢失

### 5.4 先注册后使用

所有流通道遵守此规则，否则提前到达的数据丢失。

**服务端必须遵守的顺序**：

```
ctx.registerStream(voiceStreamId, callback)   // 1. 先注册
return { ... }                                 // 2. 再返回
```

**客户端必须遵守的顺序**：

```
ctx.registerStream(textResponseStreamId, callback)   // 1. 先注册
ctx.registerStream(voiceResponseStreamId, callback)  // 2. 先注册
ctx.callRaw(targetId, PROMPT_ROUTE_KEY, payload)     // 3. 最后发起请求
```

## 6. 环境变量配置

asks 应用支持通过环境变量注入配置，无需修改代码即可调整行为。

### 6.1 `.env.example`

在 asks 包根目录提供 `.env.example`，列出所有可配置项：

```bash
# LLM API 配置
LLM_API_URL=https://api.example.com/v1/chat
LLM_API_KEY=your-api-key-here
LLM_MODEL=gpt-4

# 数据库配置
# DATABASE_URL=postgresql://user:pass@localhost:5432/askaway

# 日志级别 (debug | info | warn | error)
LOG_LEVEL=info

# 文件存储路径
# STORAGE_PATH=/data/attachments
```

格式规则：
- `KEY=VALUE` 为配置行
- `#` 开头为注释
- 仅大写字母、数字、下划线构成的 KEY，不以数字开头

### 6.2 运行时注入

Fuse 启动实例时将用户在详情页填写的配置注入容器：

- **macOS**: 写入临时 `.env` 文件，通过 `set -a && . /mnt/secrets/.env && set +a` 加载
- **Windows**: 通过 `docker run --env-file` 传入

在代码中读取：

```typescript
const llmApiUrl = process.env.LLM_API_URL || 'https://default.example.com';
const logLevel = process.env.LOG_LEVEL || 'info';
```

## 7. 打包与分发

### 7.1 asks 包格式

asks 包（`.asks` 文件）是一个包含以下内容的归档：

| 文件 | 说明 |
| --- | --- |
| `manifest.info` | 包元信息（名称、版本、入口等） |
| `dist/` | 编译产物 |
| `node_modules/` | 依赖 |
| `.env.example` | 环境变量模板（可选） |
| `Actr.toml` | ACTR 配置 |

### 7.2 分发流程

```
开发者 npm run build → 打包为 .asks → 上传至托管服务 → Fuse 拉取 → 启动实例
```

1. **上传**：将 `.asks` 文件上传至 Askaway 包托管服务
2. **Fuse 发现**：Fuse 从托管服务获取应用列表，展示最新版本
3. **安装**：用户点击安装，Fuse 下载并解压
4. **环境变量**：用户在详情页填写环境变量配置
5. **运行**：Fuse 以容器方式启动 asks 实例，注入环境变量
6. **更新**：托管服务发布新版本后，用户可切换版本

## 8. 启动入口

```typescript
import path from 'node:path';
import { ActrSystem } from '@actor-rtc/actr';
import { AskServiceHandler } from './ask_service.js';
import { createAskServiceWorkload } from './ask_service_runtime.js';

async function main() {
  const configPath = process.env.ACTR_CONFIG ?? path.resolve(process.cwd(), 'Actr.toml');
  const system = await ActrSystem.fromConfig(configPath);
  const handler = new AskServiceHandler();
  const workload = createAskServiceWorkload(handler);
  const node = system.attach(workload);
  const actorRef = await node.start();

  console.log('AskService started:', actorRef.actorId());
  await actorRef.waitForShutdown();
}

main().catch(err => {
  console.error('AskService failed:', err);
  process.exitCode = 1;
});
```

## 9. 本地测试

```bash
# 终端 1: 启动 asks
npm run dev

# 终端 2: 启动 askc 客户端 demo
cd askc-ts && npm run dev
```

或通过 Fuse + 扫码连接的方式进行集成测试（参见 [askaway v2 使用流程](https://gitlab.huanqiu-ltd.com/platform/askaway-doc)）。

## 10. 接入 Askaway 平台

独立开发完成后，将 asks 接入 Askaway 平台：

1. **Actor Type 更新**：`Actr.toml` 中 `actr_type` 改为 `askaway+app_server`
2. **Realm 配置**：使用平台分配的 `realm_id`
3. **Signaling 地址**：`url = "wss://actrix1.askaway.chat/signaling/ws"`
4. **ACL 更新**：允许 `askaway+browser` 类型访问
5. **打包上传**：构建 `.asks` 并上传至托管服务
6. **Fuse 安装测试**：在 Fuse 中添加应用，扫码连接 Askaway 客户端

## 11. 参考资源

| 资源 | 链接 |
| --- | --- |
| Proto 定义 | [GoAskAway/askaway-proto](https://github.com/GoAskAway/askaway-proto) |
| asks-ts 参考实现 | [GoAskAway/asks-ts](https://github.com/GoAskAway/asks-ts) |
| askc-ts 参考实现 | [GoAskAway/askc-ts](https://github.com/GoAskAway/askc-ts) |
| ACTR 框架 (TS) | [actor-rtc/actr-ts](https://github.com/actor-rtc/actr-ts) |
| ACTR 框架 (Swift) | [actor-rtc/actr-swift](https://github.com/actor-rtc/actr-swift) |
| ACTR 框架 (Kotlin) | [actor-rtc/actr-kotlin](https://github.com/actor-rtc/actr-kotlin) |
| ACTR 框架 (Python) | [actor-rtc/actr-python](https://github.com/actor-rtc/actr-python) |
| 文档汇总 | [askaway-doc](https://gitlab.huanqiu-ltd.com/platform/askaway-doc) |