# Plugin Manifest Guide

A plugin is a JSON manifest file that describes how Omni Code communicates with an external service (ASR, TTS, etc.).

## Quick Start

Create a `manifest.json`:

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "0.1.0",
  "description": "## Usage\n\n1. ...\n2. ...",
  "localized": {
    "zh": {
      "name": "我的插件",
      "description": "## 使用流程\n\n1. ...\n2. ..."
    }
  },
  "capabilities": ["speech.tts"],
  "capability_configs": {
    "speech.tts": {
      "base_url": "https://api.example.com",
      "path": "/v1/audio/speech"
    }
  }
}
```

Register it in `community-plugins.json`:

```json
{
  "schema_version": 1,
  "plugins": [
    {
      "id": "my-plugin",
      "name": "My Plugin",
      "author": "you",
      "description": "Brief one-line description",
      "manifest_path": "plugins/my-plugin/manifest.json",
      "localized": {
        "zh": {
          "name": "我的插件",
          "description": "一句简短描述"
        }
      },
      "capabilities": ["speech.tts"]
    }
  ]
}
```

## Manifest Fields

### Top-level

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique plugin identifier |
| `name` | string | yes | Display name |
| `version` | string | yes | Semantic version |
| `description` | string | no | Markdown description shown in the API key card. Use `## Usage` for a step-by-step guide. Links are clickable. |
| `registration_url` | string | no | URL for "Get API key" button. Set to `""` to hide. |
| `api_key_label` | string | no | Custom label for the API key field. Default: `"API Key"`. Example: `"Access Token"` |
| `requires_api_key` | bool | no | Show the API key field. Default: `true`. Set to `false` to hide |
| `service_commands` | object | no | Start/stop commands for local services. `{"start": "cmd", "stop": "cmd"}` shows buttons instead of text fields |
| `localized` | object | no | Locale-specific UI strings. Keys are locale tags such as `"zh"` or `"zh-CN"` |
| `capabilities` | array | yes | `["speech.batch_asr"]`, `["speech.tts"]`, `["speech.realtime_asr"]`, or combinations |
| `capability_configs` | object | yes | Per-capability transport and API configuration (see below) |
| `setting_fields` | array | no | Custom configuration fields |

### `capability_configs`

Override settings for a specific capability. Keyed by capability name.

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

| Field | Type | Description |
|-------|------|-------------|
| `base_url` | string | API base URL (without path) |
| `path` | string | Endpoint path. Supports `${model}`, `${language}` variables |
| `poll_path` | string | Polling endpoint path. Supports `${task_id}` variable. If set, result is polled after submit |
| `model` | string | Model identifier or resource ID |
| `auth_header` | string | Custom auth header name (e.g. `"X-Api-Key"`, `"Authorization"`) |
| `auth_scheme` | string | Auth value prefix. Default: `"Bearer"` with a trailing space. Examples: `"Bearer;"`, `""` (no prefix) |
| `request_content_type` | string | `"multipart/form-data"` (default), `"application/json"`, or `"audio/wav"` for binary upload |
| `request_field_map` | object | For multipart: rename form fields. `{"file": "audio"}` sends the file as field `audio` |
| `request_body` | object | For JSON: body template with variables. Supports `${text}`, `${speaker}`, `${audio_data_uri}`, `${audio_format}` |
| `extra_headers` | object | Additional HTTP headers. Supports `${resource_id}`, `${uuid}` variables |
| `response_text_path` | string | Dot-separated path to transcription text. `"text"`, `"result.text"`, `"utterances.0.text"` |

## Transport Types

### `openai_compatible` — HTTP

Supports three request modes:

**Multipart form-data** (default):
```json
{
  "base_url": "https://api.openai.com/v1",
  "path": "/audio/transcriptions",
  "request_content_type": "multipart/form-data",
  "request_field_map": { "file": "file", "model": "model" },
  "response_text_path": "text"
}
```
Sends file + model as multipart form, reads `text` from JSON response.

**JSON body** (for APIs expecting JSON):
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
Supports `${text}` and `${speaker}` variables in body. Response contains base64 `data` field.

**Binary upload** (for APIs accepting raw audio bytes):
```json
{
  "request_content_type": "audio/wav",
  "path": "/api/v1/vc/submit?appid=${model}&language=${language}",
  "poll_path": "/api/v1/vc/query?appid=${model}&id=${task_id}",
  "response_text_path": "utterances.0.text"
}
```
Sends raw audio bytes as body. When `poll_path` is set, submits first then polls with GET until complete.

### `realtime_websocket` — WebSocket

```json
{
  "websocket_url": "wss://openspeech.bytedance.com/api/v3/sauc/bigmodel_async",
  "auth_header": "X-Api-Key",
  "event_map": { "protocol": "volcengine_sauc" }
}
```

Uses a persistent WebSocket connection for streaming.

## `setting_fields`

Custom configuration fields shown in the plugin's settings card.

```json
"setting_fields": [
  {
    "key": "model",
    "label": "Model",
    "help": "Select the model version.",
    "required": true,
    "placeholder": "ep-xxx",
    "localized": {
      "zh": {
        "label": "模型",
        "help": "选择模型版本。"
      }
    },
    "options": [
      {
        "value": "seed-tts-2.0",
        "label": "TTS 2.0",
        "localized": { "zh": { "label": "语音合成 2.0" } }
      },
      { "value": "seed-icl-2.0", "label": "ICL 2.0" }
    ],
    "capabilities": ["speech.tts"]
  },
  {
    "key": "resource_id",
    "label": "Speaker",
    "help": "Timbre ID from the console.",
    "placeholder": "zh_female_tianmei",
    "capabilities": ["speech.tts"]
  }
]
```

| Field | Type | Description |
|-------|------|-------------|
| `key` | string | Field key: `model`, `base_url`, `path`, `websocket_url`, `resource_id`, `auth_header`, `auth_scheme` |
| `label` | string | Display label |
| `help` | string | Help text |
| `required` | bool | Whether the field is required |
| `placeholder` | string | Placeholder text |
| `options` | array | Dropdown options with `value`, `label`, `help` |
| `capabilities` | array | Which capabilities this field applies to |
| `localized` | object | Locale-specific `label`, `help`, and `placeholder` values |

## Localization

Manifest objects, repository entries, setting fields, and setting options can include `localized`:

```json
"localized": {
  "zh": {
    "name": "我的插件",
    "description": "中文说明",
    "label": "模型",
    "help": "选择模型版本。",
    "placeholder": "ep-xxx",
    "api_key_label": "Access Token"
  }
}
```

Omni Code first matches the full locale tag, such as `zh-CN`, then the language code, such as `zh`, and finally falls back to the default field value.

## Variable Substitution

These variables can be used in `path`, `poll_path`, `extra_headers`, `request_body`:

| Variable | Resolves to | Usage |
|----------|-------------|-------|
| `${resource_id}` | `config.model` (user-selected model/resource) | Headers, path |
| `${uuid}` | Auto-generated UUID v4 | Headers |
| `${text}` | TTS input text | JSON body |
| `${speaker}` | User-configured speaker/timbre | JSON body |
| `${audio_data_uri}` | Base64 data URI of audio file | JSON body |
| `${audio_format}` | File extension (wav, mp3) | JSON body |
| `${model}` | `config.model` | Path query params |
| `${language}` | User-configured language (from `resource_id` field) | Path query params |
| `${task_id}` | Task ID from submit response (only in `poll_path`) | Poll path |

## Auth Configuration

Three auth modes:

**Standard Bearer** (default):
```json
{ "auth_header": "Authorization", "auth_scheme": "Bearer" }
// Produces: Authorization: Bearer {api_key}
```

**Custom header, no prefix**:
```json
{ "auth_header": "X-Api-Key", "auth_scheme": "" }
// Produces: X-Api-Key: {api_key}
```

**Custom scheme with separator**:
```json
{ "auth_header": "Authorization", "auth_scheme": "Bearer;" }
// Produces: Authorization: Bearer; {api_key}
```

## Index File (community-plugins.json)

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
      "description": "Brief one-line summary for the selector",
      "manifest_path": "plugins/my-plugin/manifest.json",
      "localized": {
        "zh": {
          "name": "我的插件",
          "description": "插件选择器摘要"
        }
      },
      "capabilities": ["speech.tts"]
    }
  ]
}
```

The index `description` is shown as a one-line summary in the plugin selector. The manifest `description` (markdown) is shown in the API key card.

## Example Plugin Reference

| Pattern | Plugin | Transport | Content Type | Key Features |
|---------|--------|-----------|-------------|--------------|
| Multipart REST | OpenAI ASR | `openai_compatible` | `multipart/form-data` | File upload, Bearer auth |
| JSON REST | Doubao TTS | `openai_compatible` | `application/json` | Body template, base64 response |
| Binary upload + poll | Doubao Batch ASR | `openai_compatible` | `audio/wav` | Raw bytes, GET polling, array path |
| WebSocket streaming | Doubao Realtime ASR | `realtime_websocket` | — | Persistent WS, SAUC protocol |
