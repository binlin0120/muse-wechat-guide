# Muse 接入微信：muse-link（ClawBot 桥）开源接入指南

> 本指南由一份实战验证过的接入指令扩充而成：正文骨架是「给 Muse 的部署流程」（审代码 → 安装 → 登录 → 值守 → 验收），后半部分补上了长期运行后定版的做法与踩坑记录（模式选择、租约纪律、媒体限制、探针补丁）。照着走，可以从零搭起一套能长期稳定运行的「微信 ↔ Muse」桥。
>
> 本指南对应版本 **v0.4.0-rc.7**。文中所有 wxid、令牌、路径均为占位符，请替换成你自己的值。**切勿把真实令牌、wxid、二维码或本地配置提交到公开仓库。**

---

## 一、项目简介与原理

**Muse Link** 是连接 Meta Muse 与微信的桥接项目，基于腾讯官方 iLink / 微信 ClawBot 协议。桥是一个常驻的 Node.js 服务：把微信里的消息事件交给 Muse 代理处理，再把代理的回复发回微信。

工作链路：

```
微信用户 ──消息──▶ ClawBot 通道 ──▶ muse-link 桥（常驻服务）
                                        │  事件入队（带租约）
                                        ▼
                                   Muse 代理领取事件
                                   （muse next，拉取式）
                                        │  处理 / 生成回复
                                        ▼
                                   Muse 代理提交回复
                                   （muse reply / send）
                                        │
微信用户 ◀──消息── 桥 ◀───────────────┘
```

几个关键设计，理解了后面才不会踩坑：

- **拉取式，没有推送**：桥不提供 webhook 回调。代理侧的 `GET /v1/events` 本身就是「领取」动作——谁来拉、谁处理。让代理及时来领，靠的是 Muse 侧安排常驻值守（hook 或定时巡检去敲代理），不是桥主动推。
- **事件带租约（lease）**：领取事件时桥会发一个 lease token，回复时必须带上它。租约是整套流程里最容易出事故的地方，详见第五、六章。
- **桥只负责传递**：消息和附件的理解、生成由 Muse 代理的原生工具完成。接口返回 `accepted` 只表示微信侧接受了请求，手机端实际显示以真机为准。
- **同一微信账号只跑一个桥实例**，多实例会互相抢事件。
- **绑定哪个微信由扫码那一下决定**：不需要预先准备编号或微信名，也不需要微信密码——ClawBot 是扫码登录。

本指南的部署定版：

| 项 | 定版 |
| --- | --- |
| 桥版本 | v0.4.0-rc.7 |
| 运行方式 | systemd 系统级服务常驻 |
| 对话模式 | `legacy_workers`（见第五章，**不要**切到 coordinator） |
| 领取方式 | `muse next` 批量领取（`--limit 5` 起步） |
| 回复方式 | 同一代理领取后连续回复，租约 token 不离手 |

---

## 二、前置条件

- **Linux** 主机（桥与 Muse 代理在同一台机器上运行），带**持久磁盘**（登录态和状态都在本地，磁盘不持久等于每次重登）。
- **Node.js 24+** 与 npm。
- **systemd** 正在运行，且有安装/管理服务的权限。
- **一个可用的 Muse 环境**：能执行 shell 命令的主聊天代理。只装桥的 Node 服务、没有 Muse 代理消费事件，是不会有人回消息的。
- **一个微信账号**，用于扫码登录。注意：安卓微信的 ClawBot 还在灰度，没有入口的账号暂时接不了，**iPhone 微信优先**。
- **无需开放任何入站端口**：桥默认只监听 `127.0.0.1`。
- 若运行环境访问外网需要代理，提前准备好代理环境变量（`http_proxy` 等）——桥的 poll 连不上微信接口，十有八九栽在这上面（见第六章踩坑 12）。

---

## 三、部署流程

### 第一步：先审代码再装，不要盲跑脚本

装之前先做一遍安全审查，确认三件事：

1. 读仓库 `README.md` 和 `docs/QUICKSTART.md`，搞清版本与部署方式；
2. 逐行看 `deploy/bootstrap.sh`、`deploy/install.sh` 和 `src/` 源码；
3. 确认：**出站只连 `*.weixin.qq.com`；本地接口只听 `127.0.0.1` 且带令牌鉴权；登录凭证只存本地**。

审完没问题就直接装，不用再逐项向机主请示确认——整个接入全程由 Muse 操作，机主只负责扫码那一下。云端缺 Node/systemd/持久盘等条件时如实说明，不要跳过审查硬闯、更不要谎称完成。

### 第二步：安装

**方式一：一键引导脚本（新环境）**

```bash
setup_file="$(mktemp)" &&
curl -fsSL --connect-timeout 10 --max-time 60 \
  https://raw.githubusercontent.com/xianjia95-oss/muse-wechat-clawbot/main/deploy/bootstrap.sh \
  --output "$setup_file" &&
bash "$setup_file" --version v0.4.0-rc.7-owner --install-only
```

用仓库根目录的 `SHA256SUMS.txt` 核对安装包校验值，`versions.json` 记录各包路径、版本与构建标识。

**方式二：手动安装**

解压安装包后在包目录运行：

```bash
bash deploy/install.sh
export PATH="$HOME/.local/bin:$PATH"
```

**install.sh 属主校验过不了时**（典型场景：以 root 运行、HOME 属于另一个用户），不要删状态、不要跳过校验硬闯。按同样的目录结构**手动等效安装**：固定一个提交版本 → `npm ci` 按锁文件装依赖 → 生成配置 → 写 systemd 单元，效果与脚本一致。

**安装后的目录布局**

| 路径 | 内容 |
| --- | --- |
| `~/.local/share/muse-link/current` | 桥程序本体（当前版本） |
| `~/.local/bin/muse-link` | CLI 入口，下文所有命令都走它 |
| `~/.config/muse-link/config.json` | 主配置（见第三步） |
| `~/.config/muse-link/env` | 服务环境变量文件（systemd 引用，含敏感项，勿入库） |
| `~/.local/state/muse-link/` | 运行状态：事件、发件箱、登录态 |
| `~/.local/state/muse-link/claims/` | 领取输出文件（找回租约 token 的地方，见第六章） |
| `~/.local/state/muse-link/inbound/` | 入站媒体文件落盘目录 |

### 第三步：配置

先初始化生成配置与本地 API 令牌：

```bash
muse-link init                 # 生成 ~/.config/muse-link/config.json 与 api_token
```

配置示例（节选，令牌为占位符）：

```json
{
  "binding": "muse",
  "state_dir": "~/.local/state/muse-link",
  "listen": { "host": "127.0.0.1", "port": 18791 },
  "api_token": "<init 生成的本地令牌，不要外泄、不要提交到仓库>",
  "allow_wxids": ["<允许对话的对方WXID>"],
  "send_timeout_ms": 12000,
  "poll_timeout_ms": 45000,
  "media_timeout_ms": 120000,
  "inbound_media_timeout_ms": 30000,
  "media_roots": ["~/workspace"],
  "require_lease_token": true,
  "operations": {
    "max_pending_events": 2000,
    "worker_claim_warn_ms": 60000,
    "worker_stale_ms": 600000,
    "event_retention_days": 7,
    "send_retention_days": 90
  }
}
```

要点：

- `allow_wxids` 是白名单：只有名单里的 wxid 发来的消息会被处理，先把机主自己的 wxid 填进去再启动。
- `api_token` 由 `init` 生成，只用于本机接口鉴权。真实值只留在本机配置里。
- `media_roots` 限定回复时可引用的媒体文件根目录，图/视频路径必须落在其中。
- `require_lease_token` 保持 `true`，不要为了省事关掉——租约是防重复消费和串话的。
- 对话模式定版为 `legacy_workers`（第五章）。不要在配置或命令行里把会话切到 coordinator 模式。

### 第四步：注册 systemd 服务并启动

把 unit 安装为系统服务（unit 模板在安装包 `deploy/` 里）：

```ini
[Unit]
Description=Muse Link WeChat bridge
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=300
StartLimitBurst=10

[Service]
Type=simple
User=<your-user>
Environment=HOME=/home/<your-user>
Environment=NODE_ENV=production
EnvironmentFile=-/home/<your-user>/.config/muse-link/env
WorkingDirectory=/home/<your-user>/.local/share/muse-link/current
ExecStart=/usr/bin/node /home/<your-user>/.local/share/muse-link/current/src/cli.mjs run
Restart=on-failure
RestartSec=2
TimeoutStopSec=210
KillMode=control-group
MemoryMax=512M
UMask=0077
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full

[Install]
WantedBy=default.target
```

注意 unit 里的环境要包含代理变量（如需代理出网）：systemd 服务默认**不继承**你 shell 里的 `http_proxy`，要在 unit 或 env 文件里显式写明，否则桥起来了 poll 也连不上微信接口（踩坑 12）。

常用运维命令：

```bash
muse-link status     # 服务与队列状态
muse-link health     # 健康检查
muse-link diagnose   # 诊断输出
muse-link metrics    # 指标
```

升级/重装会保留登录态、配置与状态目录（只要 `state_dir` 不动），但**会覆盖程序目录里的本地补丁**——第六章的探针超时补丁在升级后必须重新打并复查。

---

### 第五步：扫码登录

```bash
muse-link login                        # 生成二维码（可加 --qr-dir 指定二维码文件目录，--timeout 10m）
```

- **二维码生成后立刻把二维码图片发给机主**，不要只发文件路径——二维码几分钟就过期，机主点开路径再找文件很容易错过。
- 机主扫码后，手机上可能出现**配对数字**：机主把数字发过来后，用 `muse-link login verify --code-file <数字文件> --text -` 按流程提交。配对数字属于敏感信息，**不要写进任何公开日志**。
- 不需要、也不要向机主索要微信密码：全程只有扫码 + 可能的配对数字两样东西经过机主。

登录成功后启动桥并自查，两个指标都对上才算登录完成：

```bash
systemctl start muse-link   # 或按你的服务管理方式启动
muse-link status            # 确认 logged_in=true，且 poll 正常（last_ok_at 有值）
```

如果 poll 一直失败，先查 systemd 服务有没有拿到代理环境变量（见第二章与踩坑 12），别急着重装。

两个登录相关的硬事实，提前知道能省很多排查时间：

1. **重新登录后，主动发送需要对方先发一条消息**：桥要先收到对方的一条入站消息、建立 context，之后 `send` 才能主动发出去；否则会报 `no_context`。重装/重登后先让对方随便发一条微信，再测主动发送。
2. 登录态保存在 `state_dir` 里。备份/恢复用 `muse-link backup` / `restore`，执行时必须先停服务。

### 第六步：安排常驻值守（桥通了 ≠ 会回消息）

这是最容易漏掉、漏掉就等于没接的一步：**桥只负责把事件存下来，没有任何东西去领，微信那边就永远收不到回复。**

- 由 Muse 侧自己安排值守：子任务轮换或定时检查（hook/巡检）持续领取消息并在微信里回复，常态目标是用户发消息后**约 1 分钟内有人接**。
- 值守要扛得住桥重启和运行环境刷新：巡检发现值守不在了要能补位，光「登记过一个计划」不算数，要实际验证过自动恢复可用。
- 值守安排完，先向机主报一声，再进入验收。
- **验收标准**：机主在微信的 ClawBot 窗口发一条测试消息，值守在微信里真实回复，验收通过才算接入完成。不要只装完桥、或只领到事件，就报告完成。

### 完成回报清单

接入完成后，向机主回报三件事：

1. 登录成功没有；
2. 绑定账号的标识（**只报尾号即可**，不用全发）；
3. 桥的健康状态（`status`/`health` 的实际结果）。

保密红线：`bot_token`、二维码、聊天内容，不要发到任何公开地方。

卡住时的沟通要求：立刻说明三件事——现在什么情况、卡在哪一环、需要机主做什么。不要闷头等机主来问。

---

## 四、日常使用

下文命令都用完整路径 `~/.local/bin/muse-link`（避免 PATH 不一致踩坑），已加进 PATH 的可简写为 `muse-link`。

### 1）领取事件：`muse next`（批量领，不要只取 1 条）

```bash
~/.local/bin/muse-link muse next \
  --limit 5 --lease 30m --wait 30s \
  --output ~/.local/state/muse-link/claims/<本次新文件名>.json
```

- `--limit`：一次多领几条。对方连发多条消息时，只取 1 条会漏收、吞消息（实测踩过，见第六章）。领到后逐条登记，再按轻重缓急决定处理顺序，不要攒着等齐。
- `--lease 30m`：租约时长。任务可能跑得久就领长一点，中途可以续租（见下）。
- `--output` 的目标文件**不能预先存在**，每次领取用一个新文件名；也可以用 `--output-dir` 指定目录。
- 输出文件里包含每条事件的 `event_id` 和对应的 **lease token**——这是回复的钥匙，保管好这个文件。
- 返回 `tasks: []` 就是当前没有新事件，属于正常情况，不必反复领取。

### 2）回复事件：`muse reply`

```bash
~/.local/bin/muse-link muse reply <EVENT_ID> --lease-token <TOKEN> --text - <<'EOF'
回复正文（从 stdin 读入，--text - 表示 stdin）
EOF
```

- 带图回复加 `--image /abs/path/to.png`（可重复多次）；图文件必须落在 `media_roots` 内。
- **回复成功即完成事件**：`reply` 本身会把事件标记为已处理。之后再调 `ack` 可能返回 `stale_lease` / `event_acknowledged`——这表示「回复已经把事件标完成了」，**不是故障**，不要重试、不要报警。
- 单独确认完成才用：`muse-link muse ack <EVENT_ID...> --lease-token <TOKEN>`。

### 3）长任务续租：`muse renew`

```bash
~/.local/bin/muse-link muse renew <EVENT_ID> --lease-token <TOKEN> --lease 30m
```

领了事件但处理要花较长时间（派子任务、等外部结果）时，**尽早续租**。租约长时间没活动，健康探针会报 `worker_lease_stalled`（约 180 秒无活动即触发告警），事件也会被当成卡死。

### 4）主动发送：顶层命令 `muse-link send`

```bash
~/.local/bin/muse-link send --to <对方WXID> --text - --request-id <全局唯一ID> <<'EOF'
主动推送的正文
EOF
```

- `send` 是 **muse-link 的顶层命令**，不是 `muse-link muse send`——后者会直接报 `unknown_command`。
- `--request-id` 必须全局唯一，用于幂等与事后查询（`muse-link request <REQUEST_ID>` 可查这条发送的状态）。
- `send` **只有文本，没有任何媒体参数**：主动推送带不了图片/文件。要推图，先把图片放到一个可访问的链接（例如自建文件中转站），正文里发链接。
- 前提是已有 context：对方从未发过消息、或刚重新登录后对方还没发过消息时，`send` 会报 `no_context`（见第三章第五步）。

### 5）状态与排障

```bash
muse-link status / health / diagnose / metrics
muse-link request <REQUEST_ID>     # 查某条主动发送的投递状态
```

---

## 五、两种 conversation 模式对比与避坑

桥有两种对话组织模式，**先选对，再别乱切**：

| | `legacy_workers`（本指南定版） | coordinator（负责人制） |
| --- | --- | --- |
| 工作方式 | 代理直接用 `muse next` 领取、用 `muse reply` 回复，租约 token 传递 | 先 `muse session start --session-file ...` 建立协调会话，之后 next/reply/renew/checkpoint/history 全部要带 `--session-file`；一个负责人统一回复，执行者只出草稿 |
| 适合 | 单代理（或主代理直领直回）的简单稳定部署 | 多执行者并行出草稿、负责人统稿的复杂部署 |
| 切换成本 | — | **单向**：一旦执行过 `session start`，会话会被锁进 coordinator 模式，没有命令能切回来 |

> ⚠️ **铁律：在 legacy_workers 部署里，永远不要执行 `muse session start`。**
> 实测教训：误跑一次 `session start`，桥被单向锁进 coordinator 模式，原有领取/回复流程全部失效，最后只能整桥卸载重装、重新扫码登录。任何部署脚本、修复指令里看到 `session start`，先停下来确认当前模式再动手。

---

## 六、踩坑记录（现象 → 原因 → 正确做法）

1. **误跑 `muse session start` 后桥彻底不回消息**
   原因：会话被单向锁进 coordinator 模式（见第五章）。
   正确做法：legacy 部署永不执行 `session start`；真锁死了只能重装 + 重新扫码，并按重装流程恢复配置与状态目录。

2. **用户连发多条消息，只收到一部分**
   原因：领取时 `--limit 1`，一次只取一条，后面的消息被漏掉或吞掉。
   正确做法：`muse next --limit 5`（或按量调大）批量领取，逐条登记、逐条处理。

3. **回复时报 `agent not found` / 租约无效**
   原因：领取和回复不是同一个代理在一次任务里连续做完的——领取方结束后，lease token 随之丢失，回复方手里没有有效 token。
   正确做法：**同一个人领、同一个人回、一次做完**。token 真丢了，去 `~/.local/state/muse-link/claims/` 找当次领取的输出文件，token 就在里面，租期（领取时设定的时长，如 30 分钟）内有效。

4. **`ack` 返回 `stale_lease` / `event_acknowledged`，以为回复失败**
   原因：`reply` 成功时已经顺带把事件标了完成，事后再 `ack` 自然拿到「已确认」的回执。
   正确做法：把这两个返回值当成功信号，不重试、不当故障上报。

5. **长任务做到一半，探针报 `worker_lease_stalled`**
   原因：领了事件长时间没续租、没回复，超过约 180 秒无活动被判卡住。
   正确做法：派长任务前先 `muse renew` 把租约续够；处理期间保持租约活动。

6. **`muse-link muse send` 报 `unknown_command`**
   原因：主动发送不在 `muse` 子命令下。
   正确做法：用顶层命令 `muse-link send --to ... --text - --request-id ...`（第四章）。

7. **刚重装/重登完，主动发送报 `no_context`**
   原因：桥还没有该对方的会话 context。
   正确做法：让对方先发一条微信过来建立 context，之后再用 `send`。

8. **想用 `send` 主动推图片，怎么都发不出去**
   原因：`send` 只有文本通道，没有媒体参数，这不是 bug。
   正确做法：图片只能在「回复」里发（`muse reply --image`，且必须有可领取的事件在手）；主动推送场景一律改发图片链接。

9. **微信里直发视频文件失败（`media_upload_failed`）**
   原因：实测当前通道下视频文件直发不可靠（`reply` 虽有 `--video` 参数，多次实测上传失败）。
   正确做法：视频先传到文件中转站/对象存储，微信里只发下载链接。这也是本部署的现行交付路线。

10. **健康探针频繁误报 `hook_probe_timeout`**
    原因：桥自带脚本 `scripts/hook-check.mjs` 里探针的 execFile 超时只有 1500ms，机器一忙就超时误报。
    正确做法：先备份原文件，再把该文件中的 `timeout: 1500` 改为 `timeout: 5000`。**注意：桥重装/升级会覆盖这个文件**，升级后必须重新打这个补丁并跑一次 `muse-link health` 复查。

11. **巡检被唤醒却领了个空（`worker_claim_gap` 之类）**
    原因：事件可能已被领走，或是幽灵唤醒。
    正确做法：以 `muse next` 的实际返回为准，`tasks: []` 就是没新活，安静结束，不要反复空领。

12. **桥起来了、登录也成功了，但 poll 一直失败**
    原因：systemd 服务没有拿到代理环境变量（`http_proxy` 等），桥连不上微信接口。原指令作者明确提过：有账号就栽在这一条上。
    正确做法：先查服务实际环境（unit 的 `Environment`/`EnvironmentFile` 里有没有代理变量），补上后重启服务，再看 `status` 里 `last_ok_at` 是否刷新。不要一上来就重装重登。

13. **install.sh 因为属主校验装不上**
    原因：以 root 运行、或 HOME 属于另一个用户，脚本的属主校验过不了。
    正确做法：不要删状态、不要跳过校验硬闯。按原目录结构手动等效安装：固定提交版本、`npm ci` 装锁定依赖、生成配置、写 systemd 单元（第三章第二步）。

14. **二维码发过去，机主扫的时候已过期**
    原因：二维码只有几分钟有效，只发文件路径让机主自己去找，时间全耗在路上。
    正确做法：生成后立刻把二维码**图片本身**发给机主；过期了就重新生成，不要让机主反复试旧码。

15. **桥和登录都正常，微信发消息却没人回**
    原因：没安排常驻值守——事件躺在桥里没人领（第三章第六步）。
    正确做法：部署完成的定义包含值守在岗。安排子任务轮换或定时巡检持续领取回复，并用真机测试消息验收；桥重启/环境刷新后巡检要能把值守补回来。

---

## 七、媒体能力与限制（实测结论）

| 方向 | 类型 | 状态 | 说明 |
| --- | --- | --- | --- |
| 入站 | 文字 | ✅ | 基本通道 |
| 入站 | 图片 / 视频 | ✅ | 桥会下载并落盘到 `~/.local/state/muse-link/inbound/`，代理按文件路径读取 |
| 入站 | 语音 | ✅ | 微信侧转写文本随事件到达；原始语音文件可落盘（SILK/PCM 会尝试转 WAV） |
| 出站（reply） | 文字 | ✅ | `--text -` 走 stdin |
| 出站（reply） | 图片 | ✅ | `--image /abs/path.png`，可重复；路径须在 `media_roots` 内，且必须有可领取的事件 |
| 出站（reply） | 语音 | ⚠️ 有条件 | `--voice` 支持 wav/silk/mp3；mp3 必须带 `--voice-duration-ms` |
| 出站（reply） | 视频 | ❌ 实测失败 | 有 `--video` 参数但实测 `media_upload_failed`；改走「传中转站 + 发链接」 |
| 出站（reply） | 普通文件 | ❌ | 不支持直发，改发链接 |
| 主动（send） | 文字 | ✅ | 需已有 context（第三章第五步） |
| 主动（send） | 任何媒体 | ❌ | 无媒体参数，一律改发链接 |

一句话总结：**回复能带图，主动只能发字，视频和文件都走链接。**

---

## 八、FAQ

**Q：需要给 Muse 提供微信密码或验证码吗？**
A：不需要。ClawBot 是扫码登录；机主全程只做两件事：扫码、在手机上可能出现配对数字时把数字发给 Muse 提交。任何索要微信密码的「接入方案」都是错的。

**Q：接入前要先定好绑定哪个微信、编号是什么吗？**
A：不用。绑哪个微信由扫码那一下决定，事先写不进指令；编号是事后登记的事——绑完告诉主 Muse 一声（比如「刚绑的那个是大号」），登记进配对表即可。

**Q：安卓微信找不到 ClawBot 入口怎么办？**
A：安卓端还在灰度，没入口的账号暂时接不了，优先用 iPhone 微信扫码绑定。

**Q：为什么没有 webhook / 推送回调？**
A：桥的设计就是拉取式。代理侧 `GET /v1/events`（即 `muse next`）就是领取动作。「有消息就唤醒代理」靠 Muse 侧的 hook/巡检值守实现（第三章第六步），不是等桥推。

**Q：同一个微信账号能不能同时跑两个桥？**
A：不能。多实例会抢同一批事件，行为未定义。上游与本部署都只有一个实例。

**Q：升级或重装后要重新扫码吗？**
A：只要 `state_dir` 保留，登录态、配置、历史都在，一般不用重新扫码；但程序目录会被覆盖，本地补丁（踩坑 10 的探针超时）要在升级后重打。如果是误入 coordinator 模式那种整桥重装，按新装流程走，重新扫码。

**Q：`api_token` 丢了/泄露了怎么办？**
A：它只用于本机接口鉴权。停服务后重新 `muse-link init` 生成并同步更新服务环境文件即可。真实 token 永远不要写进公开仓库、文档或截图。

**Q：`allow_wxids` 为空会怎样？**
A：按白名单逻辑处理，先把机主自己的 wxid 填好再启动服务，避免桥对陌生消息作出反应。

**Q：回复里的图片路径有什么要求？**
A：必须是绝对路径、落在配置的 `media_roots` 内，且回复时手里要有那条事件的有效租约。

**Q：`accepted` 是不是对方已经看到了？**
A：不是。`accepted` 只表示微信接口接受了请求；实际送达与显示以对方手机为准。重要发送可用 `muse-link request <REQUEST_ID>` 查状态。

**Q：桥停机维护怎么做？**
A：`maintain`、`backup`、`restore` 这类命令都要求**先停服务**再执行，恢复完成后还要走 `restore approve` 复核，别在运行中直接动状态目录。

---

## 九、版本与上游

- 本指南对应版本：**muse-link v0.4.0-rc.7**（同系列另有 v0.3.0、v0.4.0-rc.5，命令与模式细节以各自版本文档为准）。
- 上游仓库：https://github.com/xianjia95-oss/muse-wechat-clawbot （MIT License；安装包校验见仓库 `SHA256SUMS.txt`，版本清单见 `versions.json`）。
- 本指南是部署与运维经验整理，不包含任何真实账号、令牌或本地环境信息；部署时请把文中占位符换成你自己的值。

---

*整理日期：2026-11-10。桥版本与命令如有更新，以上游仓库与 `muse-link --help` 的实际输出为准。*
