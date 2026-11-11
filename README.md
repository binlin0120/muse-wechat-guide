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
- **事件带租约（lease）**：领取事件时桥会发一个 lease token，回复时必须带上它。租约是整套流程里最容易出事故的地方，详见第五章（日常使用）与第七章（踩坑记录）。
- **桥只负责传递**：消息和附件的理解、生成由 Muse 代理的原生工具完成。接口返回 `accepted` 只表示微信侧接受了请求，手机端实际显示以真机为准。
- **同一微信账号只跑一个桥实例**，多实例会互相抢事件。
- **绑定哪个微信由扫码那一下决定**：不需要预先准备编号或微信名，也不需要微信密码——ClawBot 是扫码登录。

本指南的部署定版：

| 项 | 定版 |
| --- | --- |
| 桥版本 | v0.4.0-rc.7 |
| 运行方式 | systemd 系统级服务常驻 |
| 对话模式 | `legacy_workers`（见第六章，**不要**切到 coordinator） |
| 领取方式 | `muse next` 批量领取（`--limit 5` 起步） |
| 回复方式 | 同一代理领取后连续回复，租约 token 不离手 |
| 值守方式 | hook 每 5 秒轮询 + worker 只敲门 + 主聊直连领取回复（第四章） |

---

## 二、前置条件

- **Linux** 主机（桥与 Muse 代理在同一台机器上运行），带**持久磁盘**（登录态和状态都在本地，磁盘不持久等于每次重登）。
- **Node.js 24+** 与 npm。
- **systemd** 正在运行，且有安装/管理服务的权限。
- **一个可用的 Muse 环境**：能执行 shell 命令的主聊天代理。只装桥的 Node 服务、没有 Muse 代理消费事件，是不会有人回消息的。
- **一个微信账号**，用于扫码登录。注意：安卓微信的 ClawBot 还在灰度，没有入口的账号暂时接不了，**iPhone 微信优先**。
- **无需开放任何入站端口**：桥默认只监听 `127.0.0.1`。
- 若运行环境访问外网需要代理，提前准备好代理环境变量（`http_proxy` 等）——桥的 poll 连不上微信接口，十有八九栽在这上面（见第七章踩坑 12）。

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
| `~/.local/state/muse-link/claims/` | 领取输出文件（找回租约 token 的地方，见第七章） |
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
- 对话模式定版为 `legacy_workers`（第六章）。不要在配置或命令行里把会话切到 coordinator 模式。

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

这是最容易漏掉、漏掉就等于没接的一步：**桥只负责把事件存下来，没有任何东西去领，微信那边就永远收不到回复。**本节是要求与验收口径；这套值守的完整实现（脚本、配置、唤醒规则、worker 指令）见第四章，照着抄即可。

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

## 四、常驻值守机制（完整复刻）

第三章第六步讲的是值守的「要求与验收口径」；这一章是它的**完整实现**：定时器、值守脚本、桥自带检测器、worker、主聊五件套的真实配置与代码。一个没接触过这套东西的 Muse，照着这一章抄，能把值守机制原样搭出来。

### 4.1 架构与分工

```
每 5 秒 ──▶ hook 运行时 ──▶ wechat-watch.sh（值守脚本）
                              │  ├─ 自查：服务在否、桥健康否、unit 丢失自愈
                              │  └─ 调桥自带检测器 hook-check.mjs 数待处理事件
                              ├─ silent ──▶ 本轮结束，什么都不发生
                              └─ wake ──▶ 拉起 worker（只敲门报数）
                                            │  执行总结报给主聊
                                            ▼
                                     主聊亲自领取并回复微信
                                     （muse next / muse reply）
```

三条设计原则：

- **检测与处理分离**：脚本只判断「有没有事」，不碰消息内容，更不领消息。
- **worker 只敲门**：被唤醒的 worker 只把事实转告主聊，不领取、不回复、不 ack。领取和回复要经手租约 token，「同一个人领、同一个人回」才不丢 token（见第七章踩坑 3）。
- **主聊直连**：真正领消息、回消息的，是主聊本人在主会话里做（legacy_workers 模式，见第六章）。

### 4.2 第一层：桥自带检测器 hook-check

桥的安装包自带检测器 `~/.local/share/muse-link/current/scripts/hook-check.mjs`。它做两件事：

1. 以子进程方式跑一次桥的 `health`，从返回里读出待处理（ready）事件数；
2. 数一遍后台 job 目录（`~/.local/state/muse-link-worker/jobs`）里 pending 的任务数；

然后输出一行结论：silent，或 wake 并附原因（如「微信有 N 条新事件待处理」）。

平台侧只需要一个极薄的适配脚本把它接上，保存为 `~/hooks/scripts/muse-link-wechat.sh` 并加可执行权限（可直接照抄）：

```sh
#!/bin/sh
# Detector only. A custom platform adapter must emit exactly one result.
# Set MUSE_LINK_HOOK_FORMAT=hatch only when this script is not already wrapped.
set -eu
PROBE="${MUSE_LINK_HOOK_HELPER:-$HOME/.local/share/muse-link/current/scripts/hook-check.mjs}"
if [ -f "$PROBE" ]; then
    exec node "$PROBE"
fi
detected_at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
if [ "${MUSE_LINK_HOOK_FORMAT:-plain}" = hatch ]; then
    printf 'HATCH_HOOK_RESULT:{"decision":"wake","reason":"hook_detector_missing; detected_at=%s; recheck current status before acting; inspect installation"}\n' "$detected_at"
else
    printf 'wake: hook_detector_missing; detected_at=%s; recheck current status before acting; inspect installation\n' "$detected_at"
fi
```

> ⚠️ **探针超时补丁（必打）**：hook-check.mjs 里跑 health 的那个子进程，execFile 超时原厂只有 **1500ms**，而 health 实测要 2.4–2.9 秒才返回——不改必然误报 `hook_probe_timeout`。打法：
>
> 1. 先备份原文件到工作区（如 `~/workspace/system/hook-check.mjs.bak-<日期>`）；
> 2. 把 hook-check.mjs 里的 `const options = { timeout: 1500, ... }` 改成 `timeout: 5000`；
> 3. 跑一次 `muse-link health` 复查。
>
> 它在链路里的位置就是本节这一步：值守脚本每次调检测器、检测器每次跑 health，都走这个超时。**桥重装/升级会覆盖这个文件，升级后必须重打。**与第七章踩坑 10 是同一件事，两处互为索引。

### 4.3 第二层：值守脚本 wechat-watch.sh

这是部署侧自加的包装层（上游没有），补上裸检测器缺的四种行为：①新事件唤醒 180 秒去重；②桥故障连续满 180 秒才报警、同一轮故障只报一次；③桥健康在服务层双查（systemctl + CLI health），不靠事件数猜；④systemd unit 文件丢失时从备份自动恢复并拉起服务。

前置条件：

- 系统装有 `jq`；
- 把在用的 systemd unit 备份一份到 `~/workspace/system/muse-link.service`（自愈要用）；
- 脚本里 `source "$HATCH_HOOK_RUNTIME"` 引入的 `log` / `silent` / `wake` 是 hook 运行时提供的助手函数（见 4.4 的运行时契约）；状态文件在 `~/hooks/state/wechat-watch/state.json`，脚本自己维护。

脚本全文，保存为 `~/hooks/scripts/wechat-watch.sh` 并加可执行权限（可直接照抄，逻辑即线上版本）：

```bash
#!/usr/bin/env bash
# wechat-watch: wrapper around the installer probe (~/hooks/scripts/muse-link-wechat.sh).
# Adds the deployment-spec behaviors the bare probe lacks:
#  1) 180s wake dedup for new-event wakes unless the pending count grows
#  2) bridge failure alerts only after 180s of continuous failure, once per episode
#  3) daemon health checked at the service level (systemctl is-active) plus CLI health,
#     not inferred from inbox counts
#  4) self-heal: if the systemd unit file is lost, restore it from
#     ~/workspace/system/muse-link.service, daemon-reload and start the service
set -euo pipefail
source "$HATCH_HOOK_RUNTIME"

DRY=0
[ "${HATCH_HOOK_DRY_RUN:-0}" = "1" ] && DRY=1

STATE_DIR="$HOME/hooks/state/wechat-watch"
STATE="$STATE_DIR/state.json"
UNIT="/etc/systemd/system/muse-link.service"
BACKUP="$HOME/workspace/system/muse-link.service"
SERVICE="muse-link.service"
CLI="$HOME/.local/bin/muse-link"
PROBE="$HOME/hooks/scripts/muse-link-wechat.sh"
NOW="$(date +%s)"

mkdir -p "$STATE_DIR"

# ---- load state -------------------------------------------------------------
last_event_wake_at=0; last_event_count=-1; fail_since=0; alert_sent=0; last_soft_alert_at=0
if [ -f "$STATE" ]; then
  read -r last_event_wake_at last_event_count fail_since alert_sent last_soft_alert_at <<EOF || true
$(jq -r '[.last_event_wake_at // 0, .last_event_count // -1, .fail_since // 0, .alert_sent // 0, .last_soft_alert_at // 0] | @tsv' "$STATE" 2>/dev/null || echo "0 -1 0 0 0")
EOF
fi
new_fail_since="$fail_since"; new_alert_sent="$alert_sent"
new_last_event_wake_at="$last_event_wake_at"; new_last_event_count="$last_event_count"
new_last_soft_alert_at="$last_soft_alert_at"

save_state() {
  [ "$DRY" = "1" ] && return 0
  jq -n \
    --argjson ew "$new_last_event_wake_at" --argjson ec "$new_last_event_count" \
    --argjson fs "$new_fail_since" --argjson as "$new_alert_sent" \
    --argjson sa "$new_last_soft_alert_at" \
    '{last_event_wake_at:$ew, last_event_count:$ec, fail_since:$fs, alert_sent:$as, last_soft_alert_at:$sa}' \
    > "$STATE.tmp" && mv "$STATE.tmp" "$STATE"
}

# ---- 4) self-heal: unit file lost -------------------------------------------
healed=0; heal_failed=0
if [ ! -f "$UNIT" ]; then
  log "unit_missing" '{"unit":"/etc/systemd/system/muse-link.service"}'
  if [ -f "$BACKUP" ]; then
    if [ "$DRY" = "1" ]; then
      log "unit_heal_skipped_dry_run" '{}'
    elif cp "$BACKUP" "$UNIT" 2>/dev/null && systemctl daemon-reload 2>/dev/null; then
      systemctl start "$SERVICE" 2>/dev/null || true
      healed=1
      log "unit_restored_from_backup" '{}'
    else
      heal_failed=1
      log "unit_restore_failed" '{}'
    fi
  else
    heal_failed=1
    log "unit_backup_missing" '{}'
  fi
fi

# ---- 3) daemon health: service level + CLI health ----------------------------
svc_state="$(systemctl is-active "$SERVICE" 2>/dev/null || echo unknown)"
health_json="$(timeout 8 "$CLI" health 2>/dev/null || true)"
healthy="no"
if [ -n "$health_json" ] && [ "$(printf '%s' "$health_json" | jq -r 'if (.healthy == true) and ((.problems // []) | length == 0) then "yes" else "no" end' 2>/dev/null || echo no)" = "yes" ]; then
  healthy="yes"
fi
daemon_ok=0
[ "$svc_state" = "active" ] && [ "$healthy" = "yes" ] && daemon_ok=1
log "daemon_check" "{\"svc\":\"$svc_state\",\"healthy\":\"$healthy\",\"healed\":$healed,\"heal_failed\":$heal_failed}"

# ---- 2) continuous-failure tracking ------------------------------------------
alert_due=0; fail_duration=0
if [ "$daemon_ok" = "1" ]; then
  new_fail_since=0; new_alert_sent=0
else
  [ "$new_fail_since" = "0" ] && new_fail_since="$NOW"
  fail_duration=$((NOW - new_fail_since))
  if [ "$fail_duration" -ge 180 ] && [ "$new_alert_sent" = "0" ]; then
    alert_due=1; new_alert_sent=1
  fi
fi

# ---- probe: new events / pending jobs / detector alerts ----------------------
probe_out="$(MUSE_LINK_HOOK_FORMAT=hatch "$PROBE" 2>/dev/null || true)"
probe_line="$(printf '%s\n' "$probe_out" | grep '^HATCH_HOOK_RESULT:' | tail -n 1 || true)"
p_decision="silent"; p_reason=""
if [ -n "$probe_line" ]; then
  p_json="${probe_line#HATCH_HOOK_RESULT:}"
  p_decision="$(printf '%s' "$p_json" | jq -r '.decision // "silent"' 2>/dev/null || echo silent)"
  p_reason="$(printf '%s' "$p_json" | jq -r '.reason // ""' 2>/dev/null || echo "")"
fi

event_count=""
event_kind=""
if [ "$p_decision" = "wake" ]; then
  event_count="$(printf '%s' "$p_reason" | sed -n 's/.*微信有 \([0-9][0-9]*\) 条新事件.*/\1/p' | head -n 1)"
  [ -n "$event_count" ] && event_kind="wechat_events"
  if [ -z "$event_count" ]; then
    event_count="$(printf '%s' "$p_reason" | sed -n 's/.*Muse 有 \([0-9][0-9]*\) 个后台任务.*/\1/p' | head -n 1)"
    [ -n "$event_count" ] && event_kind="background_jobs"
  fi
fi

# ---- decide -------------------------------------------------------------------
if [ -n "$event_kind" ]; then
  # 1) 180s dedup unless the pending count grew
  if [ $((NOW - last_event_wake_at)) -lt 180 ] && [ "$event_count" -le "$last_event_count" ]; then
    log "event_wake_deduped" "{\"kind\":\"$event_kind\",\"count\":$event_count,\"last_count\":$last_event_count}"
    save_state
    silent "wechat events already reported within 180s and count not increased" "{\"kind\":\"$event_kind\",\"count\":$event_count}"
  fi
  new_last_event_wake_at="$NOW"; new_last_event_count="$event_count"
  save_state
  wake "$p_reason" "{\"kind\":\"$event_kind\",\"count\":$event_count,\"svc\":\"$svc_state\",\"healthy\":\"$healthy\"}"
elif [ "$alert_due" = "1" ]; then
  save_state
  wake "微信桥连续故障已 ${fail_duration} 秒：systemd 服务状态=${svc_state}，CLI health 健康=${healthy}${heal_failed:+；unit 自愈失败}。请向主号汇报桥故障，不要自行清租约；如 unit 丢失脚本会先自愈。" "{\"kind\":\"bridge_failure\",\"svc\":\"$svc_state\",\"healthy\":\"$healthy\",\"fail_duration_secs\":$fail_duration,\"healed\":$healed,\"heal_failed\":$heal_failed}"
elif [ "$p_decision" = "wake" ] && [ "$daemon_ok" = "1" ]; then
  # Detector-side alert while the daemon itself is healthy (e.g. job state unreadable):
  # pass through at most once per 180s. Bridge-failure alerts are governed above.
  if [ $((NOW - last_soft_alert_at)) -lt 180 ]; then
    save_state
    silent "detector alert deduped within 180s" '{}'
  fi
  new_last_soft_alert_at="$NOW"
  save_state
  wake "$p_reason" "{\"kind\":\"detector_alert\",\"svc\":\"$svc_state\",\"healthy\":\"$healthy\"}"
else
  save_state
  silent "no new wechat events; bridge ok or failure younger than 180s" "{\"svc\":\"$svc_state\",\"healthy\":\"$healthy\",\"fail_duration_secs\":$fail_duration}"
fi
```

逐段看点：状态加载/保存跨轮记住上次唤醒时间、上次事件数、故障起始、是否已报过警（dry-run 不写状态）；探针段只取适配脚本输出的最后一行 `HATCH_HOOK_RESULT:` 解析；决策段的优先级是「新事件/新任务 → 故障报警 → 检测器软告警 → silent」；每条出口路径都会先存状态再给出唯一结论。

### 4.4 第三层：hook 注册配置

把值守脚本注册成一个定时 hook，字段如下：

| 字段 | 值 | 说明 |
| --- | --- | --- |
| id | `wechat-watch` | 稳定标识 |
| 脚本 | `~/hooks/scripts/wechat-watch.sh` | 4.3 的脚本 |
| 轮询间隔 | **5 秒** | 脚本本身很轻（一次 health + 一次数数），5 秒不压桥 |
| 脚本超时 | **30 秒** | 单轮最长执行时间，超了强杀 |
| 投递目标 | **主聊（main）** | wake 的结果只送主聊，不送任何旁聊或外部会话 |
| worker 指令 | 见 4.6 全文 | wake 时拉起的 worker 按这段指令行事 |

运行时契约：脚本每轮必须且只能以一个 `silent "原因" '{json}'`（无事，不拉起任何人）或 `wake "原因" '{json}'`（拉起一个 worker，reason 与 json payload 作为事件数据交给它）结束，这就是脚本里 `silent` / `wake` 助手函数的来历。注册后先 dry-run 一次验证输出格式，再启用。

其它环境没有这套 hook 系统时，用任何定时器（cron / systemd timer）每 5 秒跑同一个脚本、把 wake 的输出转交给一个带 4.6 指令的代理，效果等价。

### 4.5 唤醒规则汇总

| 情形 | 动作 | 规则 |
| --- | --- | --- |
| 有新微信事件 | wake | 180 秒内已唤醒过且待处理数没变多 → 去重压掉；数变多立刻再唤醒 |
| 有后台任务结果 | wake | 同上 |
| 桥连续故障 | wake（报警） | 连续故障满 180 秒才报，且同一轮故障只报一次；恢复后计时清零 |
| 检测器软告警（桥本身健康） | wake（转告） | 180 秒内最多透传一次 |
| 其余 | silent | 不打扰主聊 |

### 4.6 worker 指令（原文，可直接照抄）

wake 拉起的 worker 只按这段指令行事（角色称呼按你的部署替换）：

```text
微信桥值守（wechat-watch）唤醒。按 payload 的 kind 处理：
1. kind=wechat_events 或 background_jobs：有新的微信事件/后台结果待处理。按「主聊直连模式」——你只负责提醒主号有新微信消息待处理，不要自己领取、回复或 ack 任何微信消息（主聊会亲自用 muse-link muse next/reply 处理）。在你的执行总结里写清事件数量与类型即可。
2. kind=bridge_failure：微信桥连续故障已达 payload 中的 fail_duration_secs（服务状态 svc、CLI 健康 healthy 在 payload 里）。向主号汇报故障状态与持续时间；不要自行重启服务、不要清租约、不要改桥配置（unit 丢失的自愈已由值守脚本自动完成，healed/heal_failed 字段会说明）。
3. kind=detector_alert：桥本身健康但检测器报告了异常（如后台任务状态不可读），转告主号原文原因即可，不要自行处置。
任何情况下都不要直接联系用户、不要在微信里发送任何消息。
```

### 4.7 主聊接手链路

主聊被 worker 敲门后，在主会话里亲自完成领取与回复（命令细节见第五章）：

1. `muse-link muse next --limit 5 --lease 30m --wait 30s --output <claims 目录/新文件>.json` 批量领取，把每条事件的 id 与 lease token 记下；
2. 按轻重缓急处理（顺序由主聊定，不必按到达顺序）；
3. 逐条 `muse-link muse reply <EVENT_ID> --lease-token <TOKEN> --text -` 回复；长任务先 `muse renew` 续租。

为什么必须主聊亲自来：租约 token 只在领取者手里有效，「领取和回复同一个人、一次做完」才不丢 token（第七章踩坑 3）；worker 每次都是新拉起的短命代理，天然不适合持有租约。

整体延迟：5 秒轮询 + worker 敲门 + 主聊领取处理，实测约 1 分钟内接上。这个延迟是刻意取的平衡，不要为了抢秒把轮询改得更密。

### 4.8 复刻验收清单

搭完逐项验，全过才算复刻成功：

- [ ] 手动 dry-run 值守脚本一次，输出是合法的 silent/wake 结论；
- [ ] 机主在微信发一条测试消息：主聊在约 1 分钟内被唤醒，worker 只报数、主聊完成领取回复；
- [ ] 连发 3 条消息：只被唤醒一次（去重生效），3 条都被领走；
- [ ] 停掉桥服务等 3 分钟以上：只收到一次故障报警，内容含服务状态与故障时长；
- [ ] 把 systemd unit 文件临时移走再跑一轮：脚本能自愈恢复并拉起服务；
- [ ] 桥升级/重装后：复查 hook-check.mjs 的 timeout 还是 5000（被覆盖就重打，见 4.2）。

---

## 五、日常使用

下文命令都用完整路径 `~/.local/bin/muse-link`（避免 PATH 不一致踩坑），已加进 PATH 的可简写为 `muse-link`。

### 1）领取事件：`muse next`（批量领，不要只取 1 条）

```bash
~/.local/bin/muse-link muse next \
  --limit 5 --lease 30m --wait 30s \
  --output ~/.local/state/muse-link/claims/<本次新文件名>.json
```

- `--limit`：一次多领几条。对方连发多条消息时，只取 1 条会漏收、吞消息（实测踩过，见第七章）。领到后逐条登记，再按轻重缓急决定处理顺序，不要攒着等齐。
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

## 六、两种 conversation 模式对比与避坑

桥有两种对话组织模式，**先选对，再别乱切**：

| | `legacy_workers`（本指南定版） | coordinator（负责人制） |
| --- | --- | --- |
| 工作方式 | 代理直接用 `muse next` 领取、用 `muse reply` 回复，租约 token 传递 | 先 `muse session start --session-file ...` 建立协调会话，之后 next/reply/renew/checkpoint/history 全部要带 `--session-file`；一个负责人统一回复，执行者只出草稿 |
| 适合 | 单代理（或主代理直领直回）的简单稳定部署 | 多执行者并行出草稿、负责人统稿的复杂部署 |
| 切换成本 | — | **单向**：一旦执行过 `session start`，会话会被锁进 coordinator 模式，没有命令能切回来 |

> ⚠️ **铁律：在 legacy_workers 部署里，永远不要执行 `muse session start`。**
> 实测教训：误跑一次 `session start`，桥被单向锁进 coordinator 模式，原有领取/回复流程全部失效，最后只能整桥卸载重装、重新扫码登录。任何部署脚本、修复指令里看到 `session start`，先停下来确认当前模式再动手。

---

## 七、踩坑记录（现象 → 原因 → 正确做法）

1. **误跑 `muse session start` 后桥彻底不回消息**
   原因：会话被单向锁进 coordinator 模式（见第六章）。
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
   正确做法：用顶层命令 `muse-link send --to ... --text - --request-id ...`（第五章）。

7. **刚重装/重登完，主动发送报 `no_context`**
   原因：桥还没有该对方的会话 context。
   正确做法：让对方先发一条微信过来建立 context，之后再用 `send`。

8. **想用 `send` 主动推图片，怎么都发不出去**
   原因：`send` 只有文本通道，没有媒体参数，这不是 bug。
   正确做法：图片只能在「回复」里发（`muse reply --image`，且必须有可领取的事件在手）；主动推送场景一律改发图片链接。

9. **微信里直发视频文件失败（`media_upload_failed`）**
   原因：2026-11-08 两次实测上传失败（同一个 3.0MB 视频，两次都卡在向微信 CDN 上传确认，56/76 秒三次尝试未获确认）；当时桥刚重建、环境变量刚落盘，定位为当时环境的瞬时问题。
   正确做法：2026-11-10 重新实测已成功——`reply --video` 可用（见第八章数据）。但大文件上传很慢（8.85MB 约 103 秒），文件越大越接近默认 120 秒上传超时；超过约 10MB 建议先压小，或把配置里的 `media_timeout_ms` 适当调大。特别大的视频仍可走文件中转站发链接。

10. **健康探针频繁误报 `hook_probe_timeout`**
    原因：桥自带脚本 `scripts/hook-check.mjs` 里探针的 execFile 超时只有 1500ms，机器一忙就超时误报。
    正确做法：先备份原文件，再把该文件中的 `timeout: 1500` 改为 `timeout: 5000`。**注意：桥重装/升级会覆盖这个文件**，升级后必须重新打这个补丁并跑一次 `muse-link health` 复查。（这个补丁在值守链路里的位置和完整打法见第四章 4.2。）

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

16. **媒体回复失败后重试，报 `id_conflict`（request_id already belongs to another payload）**
    原因：`muse reply` 的 request_id 是按 [event_id, mode, reply_id] 算出来的（默认 mode=final、不传 --reply-id 时每条事件固定一个 ID）；桥按 [to, text, media] 算指纹，同 ID 指纹不同就拒绝。重试时改了文字、换了文件路径或改了 mode，都会撞上。
    正确做法：续传必须原样重发——同一 event + 同一 lease token，文字一字不改、媒体同一文件同一路径、mode 不变、不加 --reply-id；桥会复用首次已生成的加密快照、只重传失败的媒体，已接受的文字段不再重发。想换内容重发时，加 `--reply-id <新ID>` 换一个新 request_id（文字段会再发一遍）。`muse media --retry` 是入站附件的重试命令，对出站发送无效。

---

## 八、媒体能力与限制（实测结论）

| 方向 | 类型 | 状态 | 说明 |
| --- | --- | --- | --- |
| 入站 | 文字 | ✅ | 基本通道 |
| 入站 | 图片 / 视频 | ✅ | 桥会下载并落盘到 `~/.local/state/muse-link/inbound/`，代理按文件路径读取 |
| 入站 | 语音 | ✅ | 微信侧转写文本随事件到达；原始语音文件可落盘（SILK/PCM 会尝试转 WAV） |
| 出站（reply） | 文字 | ✅ | `--text -` 走 stdin |
| 出站（reply） | 图片 | ✅ 实测成功 | `--image /abs/path.png`，可重复；路径须在 `media_roots` 内，且必须有可领取的事件；2026-11-10 与视频同条 reply 实测成功 |
| 出站（reply） | 语音 | ⚠️ 有条件 | `--voice` 支持 wav/silk/mp3；mp3 必须带 `--voice-duration-ms` |
| 出站（reply） | 视频 | ✅ 实测成功 | `--video /abs/path.mp4`；2026-11-10 实测：289KB 图片 + 8.85MB 视频同条 reply 一次成功（3/3 部分确认、media_count=2、event_completed） |
| 出站（reply） | 普通文件 | ❌ | 不支持直发，改发链接 |
| 主动（send） | 文字 | ✅ | 需已有 context（第三章第五步） |
| 主动（send） | 任何媒体 | ❌ | 无媒体参数，一律改发链接 |

> 上传速度实测（2026-11-10）：media_prepare 0.37 秒，media_upload 约 103 秒（8.85MB 走代理，约 86KB/s），send 4.9 秒。大文件主要耗时在上传；`media_timeout_ms` 默认 120 秒，超过约 10MB 的文件建议先压小或调大该值。2026-11-08 的两次失败经定位是当天桥刚重建、环境刚落盘的瞬时问题，不是桥本身或配置的缺陷。视频到达对方手机后可能还要等微信转码才显示（实测图片先到、视频延迟数分钟后才出现），送达以手机端确认为准。媒体发送失败后的正确续传写法见第七章踩坑 16（原样重发；改文字会撞 id_conflict）。
>
> 一句话总结：**回复能带图、也能带视频（大文件上传慢），主动只能发字，普通文件走链接。**

---

## 九、FAQ

**Q：需要给 Muse 提供微信密码或验证码吗？**
A：不需要。ClawBot 是扫码登录；机主全程只做两件事：扫码、在手机上可能出现配对数字时把数字发给 Muse 提交。任何索要微信密码的「接入方案」都是错的。

**Q：接入前要先定好绑定哪个微信、编号是什么吗？**
A：不用。绑哪个微信由扫码那一下决定，事先写不进指令；编号是事后登记的事——绑完告诉主 Muse 一声（比如「刚绑的那个是大号」），登记进配对表即可。

**Q：安卓微信找不到 ClawBot 入口怎么办？**
A：安卓端还在灰度，没入口的账号暂时接不了，优先用 iPhone 微信扫码绑定。

**Q：为什么没有 webhook / 推送回调？**
A：桥的设计就是拉取式。代理侧 `GET /v1/events`（即 `muse next`）就是领取动作。「有消息就唤醒代理」靠 Muse 侧的 hook/巡检值守实现（第三章第六步），不是等桥推。值守机制的完整实现见第四章。

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

## 十、版本与上游

- 本指南对应版本：**muse-link v0.4.0-rc.7**（同系列另有 v0.3.0、v0.4.0-rc.5，命令与模式细节以各自版本文档为准）。
- 上游仓库：https://github.com/xianjia95-oss/muse-wechat-clawbot （MIT License；安装包校验见仓库 `SHA256SUMS.txt`，版本清单见 `versions.json`）。
- 本指南是部署与运维经验整理，不包含任何真实账号、令牌或本地环境信息；部署时请把文中占位符换成你自己的值。

---

*整理日期：2026-11-10。桥版本与命令如有更新，以上游仓库与 `muse-link --help` 的实际输出为准。*
