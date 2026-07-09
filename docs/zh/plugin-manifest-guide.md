# 插件 Manifest 编写指南

一个插件是一个 JSON manifest 文件，用于描述 Omni Code 如何与外部服务（ASR、TTS 等）通信。

## 快速开始

创建 `manifest.json`：

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "0.1.0",
  "description": "## 使用流程\n\n1. ...\n2. ...",
  "capabilities": ["speech.tts"],
  "capability_configs": {
    "speech.tts": {
      "base_url": "https://api.example.com",
      "path": "/v1/audio/speech"
    }
  }
}
```

在 `community-plugins.json` 中注册：

```json
{
  "schema_version": 1,
  "plugins": [
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "author": "you",
      "description": "一句简短描述",
      "manifest_path": "plugins/my-plugin/manifest.json",
      "capabilities": ["speech.tts"]
    }
  ]
}
```

## Manifest 字段

### 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|-------|------|----------|-------------|
| `id` | string | 是 | 唯一标识符 |
| `name` | string | 是 | 显示名称 |
| `version` | string | 是 | 语义化版本号 |
| `description` | string | 否 | Markdown 描述，展示在 API key 卡片中。建议用 `## 使用流程` 写分步指引，链接可点击 |
| `registration_url` | string | 否 | "Get API key" 按钮跳转链接。设为 `""` 则隐藏按钮 |
| `api_key_label` | string | 否 | API Key 输入框的自定义标签。默认 `"API Key"`。如填写 Access Token 可设为 `"Access Token"` |
| `requires_api_key` | bool | 否 | 是否显示 API Key 输入字段。默认 `true`，设为 `false` 则隐藏 |
| `service_commands` | object | 否 | 本地服务的启停命令。`{"start": "命令", "stop": "命令"}` 显示为按钮而非文本输入 |
| `capabilities` | array | 是 | 能力列表：`["speech.batch_asr"]`、`["speech.tts"]`、`["speech.realtime_asr"]`，可组合 |
| `capability_configs` | object | 是 | 按能力配置传输方式和 API 参数（见下文） |
| `setting_fields` | array | 否 | 自定义配置字段 |

### `capability_configs`

按能力覆盖配置，key 为能力名。

```json
"capability_configs": {
  "speech.batch_asr": {
    "base_url": "https://api.example.com",
    "path": "/v1/transcriptions",
    "model": "whisper-1",
    "auth_header": "X-Api-Key",
    "auth_scheme": "Bearer",
    "request_content_type": "multipart/form-data",
    "request_field_map": { "file": "audio", "model": "model_name" },
    "request_body": { "text": "${text}", "voice": "${speaker}" },
    "extra_headers": { "X-Resource-Id": "${resource_id}" },
    "response_text_path": "text"
  }
}
```

| 字段 | 类型 | 说明 |
|-------|------|-------------|
| `base_url` | string | API 基础 URL（不含路径） |
| `path` | string | 请求路径。支持 `${model}`、`${language}` 变量 |
| `poll_path` | string | 轮询路径。支持 `${task_id}` 变量。设置后会先提交再轮询 |
| `model` | string | 模型标识符或资源 ID |
| `auth_header` | string | 自定义鉴权 Header 名（如 `"X-Api-Key"`、`"Authorization"`） |
| `auth_scheme` | string | 鉴权值前缀。默认 `"Bearer"` 后跟空格。例如：`"Bearer;"`、`""`（无前缀） |
| `request_content_type` | string | 请求格式：`"multipart/form-data"`（默认）、`"application/json"`、`"audio/wav"`（二进制上传） |
| `request_field_map` | object | multipart 格式时重命名字段。`{"file": "audio"}` 把文件字段名改为 `audio` |
| `request_body` | object | JSON 格式时的 body 模板。支持 `${text}`、`${speaker}`、`${audio_data_uri}`、`${audio_format}` 变量 |
| `extra_headers` | object | 额外的 HTTP Header。支持 `${resource_id}`、`${uuid}` 变量。`${resource_id}` 来自模型选择，`${uuid}` 自动生成 |
| `response_text_path` | string | 响应中文本字段的路径，用点号分隔。"text"、"result.text"、"utterances.0.text" |

## 传输模式

### `openai_compatible` — HTTP

支持三种请求方式：

**Multipart 表单**（默认，适用于 OpenAI 兼容的 ASR）：
```json
{
  "base_url": "https://api.openai.com/v1",
  "path": "/audio/transcriptions",
  "request_content_type": "multipart/form-data",
  "request_field_map": { "file": "file", "model": "model" },
  "response_text_path": "text"
}
```
以 multipart 形式发送文件和模型名，从 JSON 响应中读取 `text` 字段。

**JSON body**（适用于火山引擎 TTS 等 JSON 接口）：
```json
{
  "request_content_type": "application/json",
  "request_body": {
    "req_params": {
      "text": "${text}",
      "speaker": "${speaker}",
      "audio_params": { "format": "mp3", "sample_rate": 16000 }
    }
  }
}
```
body 中支持 `${text}` 和 `${speaker}` 变量。响应中的 `data` 字段包含 base64 编码的音频。

**二进制上传**（适用于火山引擎视频字幕等接口）：
```json
{
  "request_content_type": "audio/wav",
  "path": "/api/v1/vc/submit?appid=${model}&language=${language}",
  "poll_path": "/api/v1/vc/query?appid=${model}&id=${task_id}",
  "response_text_path": "utterances.0.text"
}
```
直接发送音频二进制数据作为 body。设置 `poll_path` 后会先提交任务，再用 GET 轮询直到完成。

### `realtime_websocket` — WebSocket 实时流

```json
{
  "websocket_url": "wss://openspeech.bytedance.com/api/v3/sauc/bigmodel_async",
  "auth_header": "X-Api-Key",
  "event_map": { "protocol": "volcengine_sauc" }
}
```

建立持久 WebSocket 连接，用于低延迟流式处理。

## `setting_fields`

在插件配置卡片中显示的自定义表单字段。

```json
"setting_fields": [
  {
    "key": "model",
    "label": "Model",
    "help": "选择模型版本。",
    "required": true,
    "placeholder": "ep-xxx",
    "options": [
      { "value": "seed-tts-2.0", "label": "豆包语音合成 2.0" },
      { "value": "seed-icl-2.0", "label": "豆包声音复刻 2.0" }
    ],
    "capabilities": ["speech.tts"]
  },
  {
    "key": "resource_id",
    "label": "Speaker",
    "help": "从控制台音色库获取的音色 ID。",
    "placeholder": "zh_female_tianmei",
    "capabilities": ["speech.tts"]
  }
]
```

| 字段 | 类型 | 说明 |
|-------|------|-------------|
| `key` | string | 字段 key：`model`、`base_url`、`path`、`websocket_url`、`resource_id`、`auth_header`、`auth_scheme` |
| `label` | string | 显示标签 |
| `help` | string | 帮助文字 |
| `required` | bool | 是否必填 |
| `placeholder` | string | 占位文字 |
| `options` | array | 下拉选项，每项包含 `value`、`label`、`help` |
| `capabilities` | array | 该字段适用于哪些能力 |

## 变量替换

以下变量可用于 `path`、`poll_path`、`extra_headers`、`request_body`：

| 变量 | 解析为 | 使用场景 |
|----------|-------------|-------|
| `${resource_id}` | `config.model`（用户选择的模型） | Header、路径 |
| `${uuid}` | 自动生成的 UUID v4 | Header |
| `${text}` | TTS 输入文本 | JSON body |
| `${speaker}` | 用户配置的音色 ID | JSON body |
| `${audio_data_uri}` | 音频文件的 base64 data URI | JSON body |
| `${audio_format}` | 文件扩展名（wav、mp3） | JSON body |
| `${model}` | `config.model` | 路径查询参数 |
| `${language}` | 用户配置的语种（来自 `resource_id` 字段） | 路径查询参数 |
| `${task_id}` | 提交后返回的任务 ID（仅用于 `poll_path`） | 轮询路径 |

## 鉴权配置

三种鉴权模式：

**标准 Bearer**（默认）：
```json
{ "auth_header": "Authorization", "auth_scheme": "Bearer" }
// 生成: Authorization: Bearer {api_key}
```

**自定义 Header，无前缀**：
```json
{ "auth_header": "X-Api-Key", "auth_scheme": "" }
// 生成: X-Api-Key: {api_key}
```

**自定义方案带分隔符**：
```json
{ "auth_header": "Authorization", "auth_scheme": "Bearer;" }
// 生成: Authorization: Bearer; {api_key}
```

## 索引文件 (community-plugins.json)

```json
{
  "schema_version": 1,
  "plugins": [
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "author": "you",
      "version": "0.1.0",
      "registration_url": "https://console.example.com",
      "description": "插件选择器中显示的一句话摘要",
      "manifest_path": "plugins/my-plugin/manifest.json",
      "capabilities": ["speech.tts"]
    }
  ]
}
```

索引中的 `description` 是**一句话摘要**，显示在插件选择器中。manifest 的 `description` 是**完整的 Markdown**，显示在 API key 配置卡片中。

## 现有插件对照

| 模式 | 插件 | 传输 | 内容类型 | 关键特性 |
|---------|--------|-----------|-------------|--------------|
| Multipart REST | OpenAI ASR | `openai_compatible` | `multipart/form-data` | 文件上传、Bearer 鉴权 |
| JSON REST | 豆包 TTS | `openai_compatible` | `application/json` | body 模板、base64 响应 |
| 二进制上传 + 轮询 | 豆包 Batch ASR | `openai_compatible` | `audio/wav` | 原始字节、GET 轮询、数组路径 |
| WebSocket 流式 | 豆包 Realtime ASR | `realtime_websocket` | — | 持久 WS、SAUC 协议 |
