# askaway-proto

AskC/AskS 的 Protobuf 协议定义仓库。

## 结构

```
askaway-proto/
├── ask-service/       # 远程 AskService proto（业务逻辑）
├── client-service/    # 本地 AskLocalService proto（UI 桥接）
├── images/            # 流程图
└── askc-asks-spec.md  # 详细规范文档
```

## Proto

**AskService** (`ask-service/ask.proto`):

| RPC | 请求 | 响应 | 说明 |
| --- | --- | --- | --- |
| `Prompt` | `UsrPromptRequest` | `AssistantReply` | 文本/语音 QA |
| `PrepareAttachmentUpload` | `PrepareAttachmentUploadRequest` | `PrepareAttachmentUploadResponse` | 准备附件上传流，最终结果通过 `result_stream_id` 回传 |

**AskLocalService** (`client-service/`): 本地网关，接收 Shell 请求转发到 AskService。

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
