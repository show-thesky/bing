# 升级 DSH 会不会用不了 pierce / 打不开宿主

当前：DSH **0.2.1-rc.2**（桌面版），pierce **0.5.0**（junction 挂载）。npm 上已有 0.2.2-rc.3。

## 1. 结论

| 问题 | 答案 | 依据 |
|---|---|---|
| 会不会打不开 DSH | **概率很低**，但不是零 | 插件是 profile 里的一个 bundle；加载失败时 cordis 只让该条目「did not activate」，宿主照常启动——0.5.0 之前的排障里就出现过这种警告 |
| 会不会插件失效 | **有可能，且不会自动报警** | 我们用了若干非文档化接口（见 §4），小版本改名就断；我们没有 fail-loud，只会在面板证据表里显示异常 |
| 数据会不会丢 | **不会** | 会话日志、凭据、档案都在 `~\.dsh\`，升级只换 app 目录 |

真正需要防的不是「打不开」，而是**「能开、但注入静默失效」**——那种情况下模型照常回答，你不会察觉。

## 2. 升级前（3 分钟）

```powershell
# 1) 备份 profile 登记与 patch
$p="$env:USERPROFILE\.dsh\profiles\desktop"
Copy-Item "$p\package.json" "$p\package.json.pre-upgrade.bak" -Force
Copy-Item "$p\cordis.patch.yml" "$p\cordis.patch.yml.pre-upgrade.bak" -Force

# 2) 记住当前版本（升级后对比）
Get-Content "$env:USERPROFILE\.dsh\state\pierce\boot.json"

# 3) 记下当前两条注入体积
node "$env:USERPROFILE\.dsh\pierce\tools\frame-audit.mjs"
```

**期望值**：`version 0.5.0`、`core 1983 B + post 539 B = 2522 B / 30%`。

## 3. 升级后（3 条自检）

```powershell
# ① 宿主起来了吗、版本对不对
Get-Content "$env:USERPROFILE\.dsh\state\pierce\boot.json"
# ② 插件真的挂上了吗（adapter / surface / frame 三个字段）
Get-Content "$env:USERPROFILE\.dsh\state\pierce\status.json" | Select-String 'adapter|surface|frame'
# ③ 注入还在吗
node "$env:USERPROFILE\.dsh\pierce\tools\frame-audit.mjs"
```

再加一条行为检查：**发一句普通消息，看面板证据表是不是 `6 行`、`正文 core 1983 B`**。
如果面板按钮消失 → 客户端 slot 变了；如果证据表显示 0 段 → 装配层 hook 变了。

## 4. 我们依赖的非公开接口（升级时最先断的就是这些）

| 接口 | 用途 | 断了会怎样 |
|---|---|---|
| `systemPrompt.section({name, order, text})` | 注册两段正文 | 注入消失，模型照常回答（静默） |
| `ctx.on("system-prompt/assemble")` | 装配层改写政策句 | 漏点回落，审计能查出来 |
| `systemPrompt.refreshContext` 包装 + 原型补丁 | 用户消息里的 runtime-context 改写 | 子会话开始漏政策句 |
| `ctx.inject(["webServer"])` + `webServer.register` | 面板同源路由 | 面板退回 7314 端口（现在就是这样） |
| `conversation.input.activity` slot + `dsh.client.inject: ["slots"]` | 输入框旁的「破甲」按钮 | 按钮消失 |
| `package.json` 的 `dsh.bundle.patch` / `dsh.client` | 插件进加载树 | 完全不加载 |

## 5. 出问题时的两条路

**先摘插件，确认是不是它**（可逆，先自动备份）：

```powershell
double-click  C:\Users\Administrator\.dsh\state\pierce\safe-mode.cmd
```

它做三件事：备份 profile `package.json` → 把 `dsh-pierce` 从 bundles 里摘掉、junction 改名 `.off` → 重启宿主。
宿主能开 = 插件的问题；还开不了 = 宿主自身的问题，跟 pierce 无关。

**装回去**：

```powershell
double-click  C:\Users\Administrator\.dsh\state\pierce\restore-plugin.cmd
```

或者命令行：

```powershell
node $env:USERPROFILE\.dsh\state\pierce\profile-tools.mjs status     # 看现在挂没挂
node $env:USERPROFILE\.dsh\state\pierce\profile-tools.mjs disable    # 摘
node $env:USERPROFILE\.dsh\state\pierce\profile-tools.mjs enable     # 装回
```

**宿主彻底开不了**：把 `profiles\desktop\cordis.patch.yml` 改名（去掉所有用户 patch 覆盖），再启动。
这一步只关配置，不动插件代码，足以让宿主回到出厂配置起得来。

## 6. 一个现实限制

本机**没有 0.2.1-rc.2 的安装包**（只有已安装目录，没有安装器缓存）。
所以「回滚宿主版本」需要重新下载旧版，未必拿得到。
**能控制的回滚只有插件层**——这也是为什么要先做 §2 的备份。
