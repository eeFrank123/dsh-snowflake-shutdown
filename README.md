# dsh-snowflake-shutdown ❄️

任务完成后自动关机 —— [DeepSeek Harness](https://github.com/deepseek-ai) 插件（Windows，零依赖）。
本仓库是 **[virggle/dsh-shutdown-after-task](https://github.com/virggle/dsh-shutdown-after-task)**（MIT）的**桌面版修复 + 界面精简分支**。

跑长任务（批量迁移、模型生成、过夜任务）时，点一下右下角的雪花按钮，任务成功完成后电脑自动关机；倒计时期间可一键取消。

![按钮三种状态](https://cdn.jsdelivr.net/gh/eeFrank123/dsh-snowflake-shutdown@main/docs/preview-button.png)

![提示面板](https://cdn.jsdelivr.net/gh/eeFrank123/dsh-snowflake-shutdown@main/docs/preview-panel.png)

## 为什么会有这个分支

原插件在 **DSH 桌面版（Electron）上按钮根本不会出现**，刷新页面、完全重启都无效。

原因在于它只有一条 UI 投递通道 `webServer.tapIndex`：把 `<script src="/api/.../ui.js">` 插进**服务端渲染**的 `index.html`。
而桌面版从不经过服务端渲染 —— 宿主（`@deepseek-ai/dsh-desktop-host`）只通过 IPC 把**结构化注入行**交给窗口：

```js
process.send?.({ type: "ready", url, injections: ctx.webServer.collectIndexInjections() })
```

页面端解释器再按行类型应用（`global` / `script` / `script-src` / `script-preload` / `style` / `html`）。
这张表在**宿主启动时一次性收集**，`tapIndex` 这种"函数变换"完全不在链路上 —— 所以怎么重启都不会有按钮。

本分支补上了注入行通道，并顺手把界面改成更小、更克制的样子。

## 相对原版的改动

| # | 改动 | 说明 |
|---|---|---|
| 1 | **桌面端可用** | 额外注册 `webserver/index-inject` 行，推送 `{ kind: 'script-src', src: '/api/dsh-snowflake-shutdown/ui.js', placement: 'body' }`；原 `tapIndex` 保留，浏览器 / `dsh web` 场景照旧 |
| 2 | **雪花圆按钮** | 原来约 150×34 的文字长条 → **32×32 圆形**，内联 SVG 六向雪花（用 `currentColor`，颜色跟随状态），悬停放大 8%，文字提示移到 `title`；已开启仍是黄底 |
| 3 | **紧凑提示面板** | 原来固定 300px 宽（一大片黑色空白）→ `width:max-content` 按内容收缩、长文本最多 320px 换行；位置固定在**窗口右下角**，与按钮重叠时自动上移到按钮上方（先显示再测量，避让判断才准确） |
| 4 | **插件身份改名** | 包名、路由前缀 `/api/dsh-snowflake-shutdown/*`、日志 `~/.dsh/dsh-snowflake-shutdown.log`、类名 `dsh-snow-*` —— 可以和原版同时安装，不会抢同一个 HTTP 路由 |

功能行为（触发判定、成功判定、倒计时、取消、自动避让）与原版一致，未作改动。

## 安装

从 GitHub 安装：

```sh
dsh plugin --profile desktop add github:eeFrank123/dsh-snowflake-shutdown
```

从本地目录安装：

```sh
dsh plugin --profile desktop add file:E:\path\to\dsh-snowflake-shutdown
```

> 网页版 profile 请把 `--profile desktop` 换成 `--profile web`。

装完**完全退出 DeepSeek Harness（含系统托盘图标 → 右键 → 退出）再重新打开**。

> 为什么必须整进程重启：界面脚本是宿主启动时载入内存的，注入行也在启动时收集。只刷新页面（F5）读到的仍是旧脚本；把窗口关到托盘不算重启。

## 使用

1. 点右下角雪花按钮 → 变黄 = 已武装（默认永远是关闭状态，绝不自动开启）；再点一次 = 取消
2. 按钮可拖到任意位置，位置记在浏览器 localStorage 里
3. 全部 root 会话空闲（且本批次无错误）→ 右下角弹出「任务已完成 / N 秒后自动关机」
4. 倒计时里点「取消关机」→ 执行 `shutdown /a`，本次取消**且退出模式**；倒计时内发新消息 → 只撤销本次关机，模式保持开启

## 配置

在 profile 的 `cordis.patch.yml`（或 设置 → 插件配置）里：

```yaml
- id: dsh-snowflake-shutdown
  config:
    countdownSec: 60      # 倒计时秒数（10–600，默认 60）
    marginSec: 5          # 系统 shutdown /t 额外余量（0–60，默认 5）
    onlyOnSuccess: true   # 仅成功时关机（false = 只要跑完就关，含出错）
```

## 行为细节

| 项 | 说明 |
|---|---|
| 触发时机 | 模式开启后，全部 root 会话从 running 回到 idle |
| 成功判定 | 本批次没有 `agent/error`（仅 `onlyOnSuccess: true` 时判定） |
| 倒计时 | `countdownSec` 秒，右下角横幅实时显示剩余秒数，可点「取消关机」 |
| 系统指令 | 倒计时开始时执行 `shutdown /s /t <countdownSec + marginSec>` |
| 取消方式 | 横幅上的取消按钮（退出模式）／倒计时内发新消息（保持模式） |
| 状态查询 | `GET http://127.0.0.1:<port>/api/dsh-snowflake-shutdown/state` |

## 运行日志

`$DSH_HOME/dsh-snowflake-shutdown.log`（默认 `C:\Users\<你>\.dsh\dsh-snowflake-shutdown.log`），挂载、武装、触发、取消都有记录。

## 已知限制

- **仅 Windows**：执行的是 `shutdown.exe`，其他平台插件保持惰性并记日志
- 模式状态存在内存里：DSH 重启后回到默认关闭，需要重新点按钮
- root 会话 idle 但 inbox 还有排队消息时也会触发（判定依据是"全部 root 空闲"）
- **新增/修改插件后必须完整重启客户端**才会看到界面变化

## 卸载

```sh
dsh plugin --profile desktop remove dsh-snowflake-shutdown
```

## 致谢

- 原插件 [virggle/dsh-shutdown-after-task](https://github.com/virggle/dsh-shutdown-after-task)（MIT）—— 本分支的功能逻辑全部来自它
- 原 README 提到的参考项目：[yang040709/dsh-win-toast-notify](https://github.com/yang040709/dsh-win-toast-notify)、[keyiadiannao/dsh-power-button](https://github.com/keyiadiannao/dsh-power-button)（均为 MIT）
- 桌面端注入通道的定位来自 DSH 自身源码：`@deepseek-ai/dsh-desktop-host`、`@deepseek-ai/dsh-host-webserver`、`@deepseek-ai/dsh-web-frontend`

## 常见问题

**装完按钮没出现？**
必须**完全退出客户端（含系统托盘图标 → 退出）再打开**。界面脚本是宿主进程启动时载入内存的，注入表也在那一刻收集一次 —— 刷新页面（F5）拿到的还是旧内容，把窗口关到托盘不算重启。

**`dsh plugin ... add` 报 `lockfile entries failed verification` / `within the minimumReleaseAge cutoff`？**
pnpm 11 默认带一条供应链策略：发布不满 24 小时的包会被拦下（本插件 0.3.0 刚发布时也会遇到）。等一天再装，或者这次先关掉这个检查：

```sh
dsh plugin --profile desktop add dsh-snowflake-shutdown --config.minimumReleaseAge=0
```

**`dsh plugin ... add github:...` 卡住或报 `connection to github.com timed out`？**
这台机器直连 GitHub 不通，给 git 配一个本地代理即可（端口换成你自己的）：

```sh
git config --global http.https://github.com.proxy http://127.0.0.1:7897
```

配好后，插件市场里"从 GitHub 安装插件"也会一起恢复正常。

**按钮和我别的插件弹窗重叠怎么办？**
按钮会自动让开，弹窗消失后自动归位；也可以直接把按钮拖到自己喜欢的位置。

**想换回原版？**

```sh
dsh plugin --profile desktop remove dsh-snowflake-shutdown
dsh plugin --profile desktop add dsh-shutdown-after-task
```

## License

MIT —— 见 [LICENSE](LICENSE)，保留原项目的版权声明。
