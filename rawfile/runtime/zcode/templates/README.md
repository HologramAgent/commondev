# ZCode LLM 源配置模板

ZCode 用自己的 provider 配置（`llmConfig: "none"`，插件不碰它）。
数据目录由 `ZCODE_DATA_BASE_DIR` 决定（driver.json 里已设为 `{dataDir}/runtime/zcode`），
配置落在：`<ZCODE_DATA_BASE_DIR>/.zcode/v2/provider_config.json`。

```bash
D=<插件数据>/runtime/zcode
mkdir -p $D/.zcode/v2
cp provider_config.json.example $D/.zcode/v2/provider_config.json
# 编辑它：把 apiKey 换成真 key
chmod 600 $D/.zcode/v2/provider_config.json   # ★ 里面有 key
```

## 三个必踩的坑（都真机复现过）

1. **`schemaVersion: 1` 必须有**，`providerRules` 嵌在 `config.providerConfigRules` 下。
   放顶层不报错 —— ZCode 会**静默恢复成默认值**（改写你的文件），
   表现为"改了没生效"，排查方向完全错。
2. **`group: "standard-personal"` 必须有**。少了它配置**被读到但不被准入**，
   报 `Error: Model creation failed`；加 `--verbose` 才看得到真因
   `Select a model before continuing`。
3. `api.type` 合法值只有 `anthropic-messages` / `openai-chat-completions` / `openai-responses`。
