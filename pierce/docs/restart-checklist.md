# 重启检查清单（2026-11-14）

重启一次同时生效两件事：**pierce v0.4.0 机制**（适配器 + 档案 + 三级正文）与 **Qwen4 模型选择器**。
两者都在磁盘上，但 cordis loader 不监听配置文件（loader / app-boot / config-editor 里都没有
`fs.watch`），所以必须换宿主进程。

## 怎么做

双击 `C:\Users\Administrator\.dsh\state\pierce\restart.cmd`
（内容：`taskkill /IM "DeepSeek Harness.exe" /F` → 等 2 s → 重新启动）。
或托盘退出后手动启动。**重启会中断当前对话窗口**，会话本身不丢，重启后可继续。

## 重启后三行自检

```powershell
# 1) 版本与进程：应是 0.4.0，且 pid 与刚启动的一致
Get-Content C:\Users\Administrator\.dsh\state\pierce\boot.json

# 2) 机制在：应有 adapter / surface / frame 三个字段
(Get-Content C:\Users\Administrator\.dsh\state\pierce\status.json -Raw | ConvertFrom-Json) |
  Select-Object version, adapter, surface, @{n='frame';e={$_.frame.profile.id}}

# 3) 注入仍在：两段体积应与磁盘一致
node C:\Users\Administrator\.dsh\pierce\tools\frame-audit.mjs | Select-Object -First 6
```

**期望值**：`version 0.4.0`；`adapter dsh`、`surface web|desktop`、`frame.profile.id default`（或命中的档案 id）；
`core.md 1983 B + post.md 539 B = 2522 B`。

## 重启后两分钟能确认的两件事

1. 模型选择器里出现 **Qwen4-Max-Preview**，随便发一句，回包正常（若报 404 说明 baseURL 被改回 `/v1`）。
2. 面板（输入框工具条「破甲」）里证据表是 **6 行**（v0.4 面板），不是 4 行（v0.3 面板）。

## 若 qwen4 报 401/404

- 401 = 凭据没读到：确认 `~/.dsh/.credentials.yaml` 的 `refs.QWEN_TOKEN_PLAN_KEY` 存在（重启后宿主
  会重新读凭据库）。
- 404 = baseURL 错：必须是 `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`，
  不是 `/v1`、也不是 `dashscope.aliyuncs.com`。
