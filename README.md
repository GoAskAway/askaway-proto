# askaway-proto

AskC/AskS 的 Protobuf 协议定义仓库。

## 结构

```
askaway-proto/
├── ask-service/       # 远程 AskService proto（业务逻辑）
├── client-service/    # 本地 AskLocalService proto（UI 桥接 + 反向调用能力）
├── images/            # 流程图
└── askc-asks-spec.md  # 详细规范文档
```

## Proto

**AskService** (`ask-service/ask.proto`):

| RPC | 请求 | 响应 | 说明 |
| --- | --- | --- | --- |
| `Prompt` | `UsrPromptRequest` | `AssistantReply` | 文本/语音 QA |
| `PrepareAttachmentUpload` | `PrepareAttachmentUploadRequest` | `PrepareAttachmentUploadResponse` | 准备附件上传流，最终结果通过 `result_stream_id` 回传 |

**AskLocalService** (`client-service/ask_local.proto`): 本地网关，接收 Shell 请求转发到 AskService，并暴露可被 AskService 反向调用的能力。

| RPC | 请求 | 响应 | 说明 |
| --- | --- | --- | --- |
| `AskUserQuestion` | `AskUserQuestionRequest` | `AskUserQuestionResponse` | AskService 处理请求期间向用户发起结构化提问（含预设选项、单/多选、「Other」自由输入），阻塞至作答或取消 |

## 架构

```
Shell → AskLocalService → AskService
          (本地 Actor)      (远程 Actor)
```

三层：展示层(UI) → 状态桥接层(AskLocalService) → 远程 Actor(AskService)

## 代码生成

```bash
npm run codegen
```

## 详细文档

见 [askc-asks-spec.md](./askc-asks-spec.md)
