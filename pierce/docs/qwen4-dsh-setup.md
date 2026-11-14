# Qwen4-Max-Preview 接进 DSH（2026-11-14）

目标模型是 Qwen4-Max-Preview，走阿里百炼 **Token Plan**（套餐专用端点，不是 dashscope 按量计费）。
本文记录实测出来的接入参数、踩到的两个坑，以及"哪一层已生效、哪一层要重启"。

---

## 1. 实测参数

| 项 | 值 | 怎么确认的 |
|---|---|---|
| Model ID | `qwen4-max-preview` | 官方页 + 实调回包 |
| 端点 | `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1` | `/v1/chat/completions` 返回 **404**；`/compatible-mode/v1/chat/completions` 返回 200 |
| 协议 | OpenAI Chat Completions（`api: openai-completions`） | DSH `dsh-llm-pi-ai` |
| 回包里的模型名 | `qwen4.0-max-preview`（服务端写法，与请求 ID 不同） | 实调 |
| 上下文 / 输出 | 1,000,000 / 393,216 token（思考与回答**共享**输出预算） | 官方页 |
| 预览窗口 | 2026-11-11 10:00 ~ 2026-11-25 24:00（UTC+8），到期下线 | 官方页 |
| 计费 | **仅 Token Plan**，不支持按量计费 | 官方页 |

## 2. 配置

凭据存进 `~/.dsh/.credentials.yaml` 的 `refs:`（**不进仓库**）：

```yaml
refs:
  QWEN_TOKEN_PLAN_KEY: 'sk-sp-...'
```

GUI（`profiles/desktop/cordis.patch.yml`）与 headless（`profiles/qwen4/cordis.patch.yml`）各加一段 provider：

```yml
- id: llm-pi-ai
  config:
    providers:
      qwen4:
        displayName: Qwen4 Token Plan
        api: openai-completions
        apiKeyEnv: QWEN_TOKEN_PLAN_KEY
        baseURL: https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1
        models:
          - id: qwen4-max-preview
            name: Qwen4-Max-Preview
            contextWindow: 1000000
            maxTokens: 65536
```

`maxTokens` 别按"只给回答留额度"来填：这一代起 `max_tokens` 与 `max_completion_tokens` 等价，
思考内容从同一份预算里扣，给小了会得到空正文。

## 3. 两个坑

**坑一：`/v1` 是 404。** 套餐端点的 OpenAI 兼容路径是 `/compatible-mode/v1`，
与公开 dashscope 的 `compatible-mode/v1` 同名但主机不同。填错时表现为 404，不是 key 无效。

**坑二：headless 下带代理环境变量会 TRANSPORT 失败。** 从 `app.asar` 解包出来的 CLI（`%TEMP%\dsh-asar`）
里 `undici@8.11.2` 缺 `lib/llhttp/llhttp-wasm.js` 与 `llhttp_simd-wasm.js`。
只要进程带 `http_proxy/https_proxy`，DSH 就会装自己的全局 dispatcher，一装就走到 `client-h1`
→ 缺模块 → 任何请求都报 `Connection error.`（`TRANSPORT`），与模型、key、网络都无关。
补法：

```powershell
npm pack undici@8.11.2   # 解包后把 lib/llhttp/* 复制进 %TEMP%\dsh-asar\dsh\node_modules\undici\lib\llhttp
```

补完同一命令从 21 s 失败变成 7 s 成功。**桌面 GUI 不受影响**（它从 app.asar 里读，文件齐）。

## 4. 生效层次

| 层 | 状态 | 依据 |
|---|---|---|
| provider 配置 + 凭据 | ✅ 已写入 | `profiles/desktop`、`profiles/qwen4`、`.credentials.yaml` |
| headless 端到端 | ✅ 实测 | `docs/evidence/qwen4-2026-11-14/`：单轮问答 7 s、工具调用（读文件取 version）12 s，`--json` 事件里 thinking 与 final 都正常 |
| 多轮回传 `reasoning_content` | ✅ 实测 | 工具调用必然产生第二轮请求，未出现 400 |
| 桌面 GUI 模型列表 | ⏳ 需重启 | cordis loader **不监听** patch 文件（loader / app-boot / config-editor 里都没有 `fs.watch`），改文件不会热载 |

GUI 侧结论：配置已经在磁盘上，模型选择器要等**下一次宿主重启**才出现；那次重启同时会把
pierce v0.4.0（适配器 + profiles）带到运行态。

## 5. 与 pierce 的关系

`profiles/qwen4/profile.json` 是 **trial** 档：未测、不进准入名单、不写入 `scores`。
已知歧义：`match: ["qwen4"]` 与 qwen 档案的 `"qwen"` 会同时命中 Qwen4 的模型名，
`matchProfile` 按"宁可不猜"回落默认档案——要自动识别，得先定"更具体者优先"的规则。
