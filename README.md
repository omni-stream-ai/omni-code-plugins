# Omni Code Plugins

Community plugin repository for [Omni Code](https://github.com/omni-stream-ai/omni-code). Add third-party services (ASR, TTS, and more) via JSON manifest files.

## Adding a Plugin

1. Create a `manifest.json` for your plugin — see the [Manifest Writing Guide (EN)](docs/en/plugin-manifest-guide.md) / [插件编写指南 (中文)](docs/zh/plugin-manifest-guide.md).
2. Add your plugin entry to [`community-plugins.json`](community-plugins.json).
3. Open a PR to this repository.

## Available Plugins

| Plugin | Capabilities | Description |
|--------|-------------|-------------|
| Doubao Batch ASR | `batch_asr` | Volcengine video caption / batch speech recognition |
| Doubao TTS | `tts` | Volcengine streaming text-to-speech synthesis |
| Doubao Realtime ASR | `realtime_asr` | Volcengine real-time streaming speech recognition |

## Repository Structure

```
community-plugins.json    # Plugin index
plugins/                  # Plugin manifest files
  {plugin-id}/
    manifest.json
docs/                     # Plugin development guides
  en/
  zh/
```
