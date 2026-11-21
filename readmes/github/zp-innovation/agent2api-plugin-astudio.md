# agent2api-plugin-astudio

AStudio（讯飞星辰 Agent Studio）作为 [agent2api](https://github.com/zp-innovation/agent2api) / magpie provider：**微信扫码登录 + Responses 协议对话 + 模型目录 + 积分余额 + 每日登录签到**。

> Provider id: `astudio`　对话: `https://astronstudio-api-volces-prod.xf-yun.com/model-gateway/v1`　账号面: `https://agent.xfyun.cn/xingchen-studio/`
> 仓库: https://github.com/zp-innovation/agent2api-plugin-astudio　当前版本: `0.2.0`

---

## 为什么需要这个插件

AStudio 把能力拆在两个面上，而这两个面用的不是同一份凭据：

| 用途 | 凭据 | 能读到什么 |
| --- | --- | --- |
| 对话（`/model-gateway/v1`，Responses 协议） | `modelBearerToken`（Bearer） | 只有推理 |
| 积分 / 会员 / 签到 / 账号可用模型（`/xingchen-studio/*`） | SSO Cookie 会话（`ssoSessionId` + `account_id`） | 这些面**不认 Bearer** |
| 换 `modelBearerToken` | 同一份 SSO Cookie 会话（`POST auth/bearer-token`） | — |

所以「一个 API Key 就够」这条路走不通：粘贴进来的 Bearer 能对话，但余额与签到永远读不到。官方桌面端自己也是这么走的 —— 扫码/手机号拿到会话，再用会话换 Bearer。

**扫码登录的关键在于二维码是一张公网 HTTPS 图片**，不是本机回环页：

1. 插件生成 8 位 `fd`，`GET sso.xfyun.cn/SSOService/wx/scanlogin/qrcode?fd=` 拿到二维码**图片 URL**；
2. 宿主把这张图片画在内嵌窗口里（`allowedHosts` 为空数组 ⇒ HTTPS 不限域）；
3. 插件自己 `GET .../uids?fd=` 轮询，扫到就把 `ssoSessionId`/`uid` 换成会话，再换 `modelBearerToken`。

全程不需要开本机端口、不需要调起浏览器、不需要过极验，因此**跨机部署（网关在远端、人在本地）天然可用**。

> 实现是照着协议重写的，没有复制 new-api 的 `pkg/astudio/scanlogin.go`（该仓库无许可证）。

## 安装

面板 → 插件 → 添加脚本插件：

```text
github:zp-innovation/agent2api-plugin-astudio
```

或用本地目录（写**网关主机**上的绝对路径）：

```text
file:/data/projects/agent2api/data/script-plugins-src/agent2api-plugin-astudio
```

装到别的机器上时，`file:` 后面必须是**网关进程能看见**的绝对路径 —— 网关跑在容器里的话，
要写容器内路径（宿主的 `data/` 目录就是容器的 `/data`）。这台部署（gk，192.168.10.10）的实际做法：

```bash
scp -r AStudio gkserver:/home/gk/astudio-upload
ssh gkserver "sudo cp -a /home/gk/astudio-upload /data/projects/agent2api/data/script-plugins/incoming/AStudio"
curl -sS -X POST http://127.0.0.1:3065/api/plugins/scripts/install \
  -H 'Authorization: Bearer <任意有效网关 API Key>' -H 'Content-Type: application/json' \
  -d '{"source":"file:/data/script-plugins/incoming/AStudio","trusted":true}'
```

`trusted:true` 是硬性的（`operations.rs:62`：`安装脚本插件前必须确认信任来源；脚本并非操作系统沙箱`）。
管理接口认网关 API Key 而不只是面板口令（`http.rs:1288`），所以脚本化安装走 Key 就行。

**provider id 就是 `astudio`，面板显示「AStudio · zp」**。这个 id 原本被官方 wasm 包
`agent2api-wasm-astudio` 占着（脚本插件不能申报「已经有执行体申报过的 id」：
`plugin_host/script/runtime.rs:290-297` 的 `reserved` 含所有 Supported 包的 `declared_ids`），
所以早先只能叫 `astudio-zp`。那个包不支持扫码登录，卸掉之后 `astudio` 就空了 ——
它不在 `core::providers::PROVIDERS`、不在 `BUILTIN_SUBSCRIPTION_IDS`，旧原生包
`agent2api-providers` 是 Legacy 状态、按 `ownership.rs:59-61` 直接跳过。于是本插件正当地接管
`astudio`：库里那几个 `provider=astudio` 的老账号继续归这张卡，面板只剩一张卡。

> 老账号是 wasm 包时代落盘的 **api 型**凭据（只有 `key`，没有 `refresh`/`expires`），
> 宿主的「401 自动续期」只认 oauth 型（`host/allowance.mjs:29-38`）。所以插件会在第一次
> `usage` 读数时顺手升级一次：先用会话换一个新凭据（同时验证会话还活着），换到了才把 secret
> 补成 oauth 型（`lib/legacy.mjs`）。会话已经死掉的账号保持原样，不会被改成「每次请求都要
> 续期、续期必失败」的坏状态。
## 登录方式

面板里的顺序就是方法序号（`host/login-policy.mjs:9` 取数组下标）：

| # | 方式 | 类型 | 需要什么 | 说明 |
| --- | --- | --- | --- | --- |
| 0 | 微信扫码登录 | `oauth` | 微信 + **已绑定星辰账号** | 推荐。登录后可读余额、可签到 |
| 1 | 粘贴模型凭据 | `api` | `modelBearerToken` | 救急用：能对话，但**没有会话** ⇒ 余额/签到读不到 |

### 扫码这条的交互

二维码 60 秒有效、2 秒轮询一次（过期后降频到 5 秒，接住「过期前一秒扫上」的确认）。窗口下方那行提示会随状态变化（靠 `authorize` 返回对象上的 `instructions` getter 实时刷新）：

| 状态 | 提示 |
| --- | --- |
| 刚取到码（第 1 次轮询之前） | 二维码已就绪，请在微信中扫码 |
| 还没人扫 | 等待微信扫码… |

> 第一行只会出现在窗口的**静态说明**里：宿主 poll 的提示取 `authorization.instructions`，
> 但要求它与 start 时烘进流程的文案**不同**才会显示（`login.mjs:112-124`），而 start 时拿到的
> 正是「二维码已就绪…」。所以第一次轮询就把 hint 换成「等待微信扫码…」，否则面板前 1.5 秒会显示
> 宿主的默认提示「等待插件完成登录」。

| 超过 60 秒 | 二维码已过期：请取消这次登录，再点一次「微信扫码登录」 |
| 扫了，但 `infoList` 空 | 微信已扫码，但这个微信还没有绑定星辰账号：请先在星辰网页端完成绑定，再扫一次 |
| 已确认 | 微信已确认，正在换取模型凭据… |

「换一张二维码」就是宿主再点一次登录：插件会生成新的 `fd`，旧流程随之作废。

实测（2026-11-20，无 Cookie、无 clientType）：

```jsonc
// GET /SSOService/wx/scanlogin/qrcode?fd=k7Qm2Zx9
{"flag":true,"code":0,"desc":"获取成功","data":"https://mp.weixin.qq.com/cgi-bin/showqrcode?ticket=gQFU8DwAA…"}
// GET /SSOService/wx/scanlogin/uids?fd=k7Qm2Zx9   （还没人扫）
{"flag":false,"code":-1,"desc":"暂无数据","data":null}
```

二维码图片落在 `mp.weixin.qq.com`，正是「公网 HTTPS 图片」这条路成立的原因。

## 账号记录长什么样

`authorize` 返回的 secret 是 **oauth 型**，不是死 Key：

```json
{
  "type": "oauth",
  "access": "<modelBearerToken>",
  "refresh": "astudio-sso",
  "expires": 1763700000000,
  "name": "星辰用户",
  "uid": "10086",
  "accountId": "10086",
  "source": "scan",
  "session": { "accountId": "10086", "uid": "10086", "ssoSessionId": "…", "token": "" },
  "metadata": { "provider": "astudio", "source": "scan" }
}
```

为什么必须是 oauth 型：

- `host/accounts.mjs:65-89`：`if (auth?.type !== "oauth") return auth` —— api 型账号宿主**根本不会**调 `hooks.auth.refresh`；
- `host/allowance.mjs:29-38`：只在 401 + oauth + 有 refresh + `expires` 未过时保留账号（把 `expires` 置 1），下一请求才会触发续期。

因此插件把 `expires` 设成本地 30 分钟，到期或上游 401 时用 **Cookie 会话**重铸一次 `modelBearerToken` —— 会话还在就永远不用重新扫码。会话本身被拒（401/403 或 `code=80000` / desc 含「重新登录」）才算登录过期。

## 面板显示

**余额**（两行，都是 `aside: true`）：

- `通用积分` —— `GET points/balance` 的 `totalBalance`；`activityBalance > 0` 时附带 `runsOutAt`（那部分积分的到期时刻）。
- `星火积分` —— `sparkTotalBalance`。

`aside` 是刻意的：积分是**余额型**读数，恰好一池为 0 时不能触发宿主的配额拦截（`host/allowance.mjs:5-27` 只看非 aside 窗口的 `remainingPercent/balance === 0`）。

**会员**：`membership/me` 的 `planName` 作计划名、`endTime` 作到期时间、`uid` 作账号标识。

读不到时不会染红整列：带 `plan` 的返回会被 `host/reports.mjs:29` 降级成 warnings 里的一条提示。

**签到**：`checkin` 先读一次积分 → `POST tenant-app/v2/init-app`（官方每日登录就是这一步）→ 等 2.5 秒再读一次，用**实际到账的差额**报告结果：

| outcome | 什么时候 |
| --- | --- |
| `claimed` | 观测到新增积分，`credit` 形如「通用积分 +30」 |
| `done` | 已执行，但两次读数没有差额（今日已领或尚未到账） |
| `inactive` | `membership/plan-config` 的 `dailyLoginEnabled` 为 false |
| `ineligible` | 只有 Bearer、没有会话（粘贴方式登录的账号） |
| `failed` | 上游拒绝或读数不完整 |

## 模型目录

两份来源合并（`lib/catalog.mjs`）：

1. **账号目录** `GET bot/models/configs` —— 这个账号能用哪些模型，带 `api_key`/`is_default`；
2. **平台目录** `GET /api/v1/model-manager/models`（Bearer） —— 元数据：`context_window`、`max_output_tokens`、`supported_reasoning_levels`、`input_modalities`、`badge`。

账号目录在前，平台目录按 `slug` / `display_name` 去重后追加；两份都拿不到时用官方兜底清单，并打上 `magpie.fellBack` 标记，让宿主保留已有模型而不是清空目录：

| id | 名称 | 上下文 | 输出 |
| --- | --- | --- | --- |
| `astronclaw-auto` | Auto（默认） | 262,144 | 32,768 |
| `spark-x2.6` | Spark-X2.6 | 262,144 | 32,768 |
| `spark-x2.6-4b` | Spark-X2.6-Lite | 1,048,576 | 110,000 |
| `spark-x2.5` | Spark-X2.5 | 262,144 | 32,768 |
| `xopglm52` | GLM-5.2 | 1,048,576 | 32,768 |
| `xopdeepseekv4pro0813` | DeepSeek-V4-Pro | 1,048,576 | 32,768 |
| `xopdsv4flash0731in` | DeepSeek-V4-Flash | 1,048,576 | 32,768 |

## 已知边界

1. **二维码只有 60 秒**，上游定的，不能延长；过期只能重开一次登录（宿主自己在 10 分钟时判「登录等待超时」）。
2. **没扫码时 `uids` 一定返回业务拒绝，而且文案会变**：实测两种 —— `code=20003`「请求地址错误」、`code=-1`「暂无数据」。这都是正常态而不是故障，插件把已知的这两种（以及**任何**「还没扫到人」的业务拒绝）翻译成「等待微信扫码…」，绝不把上游文案抛给用户；取码那一步的业务拒绝才是真失败。
3. **扫了但没绑定**：`infoList` 为空表示这个微信没有绑定星辰账号，提示去网页端绑定，而不是判失败。
4. **内嵌窗口能显示二维码**，是因为宿主 `host/login-policy.mjs:95-135` 对非 `wps-lingxi`/`joycode` 的 provider 返回 `allowedHosts: []`，而壳 `login.rs` 的 `DynamicNavigation` 把空数组解释为「HTTPS 不限域」。老的 `run_embedded` 静态白名单（`WORKBUDDY_ALLOWED_HOSTS` 那条路径）不参与动态登录。
5. **`browserMode` 写的是 `plugin` 而不是 `embedded`** —— 面板校验只接受 `choice|external|plugin|none`；真正强制内嵌窗口的是 `embeddedOnly: true`。
6. **不声明思考档位（variants）**：`supported_reasoning_levels` 里的 `effort` 值还没在上游验证过，随便给可能让对话请求直接 400，所以只把 `reasoning: true` 摆出来。
7. **粘贴方式没有会话**，所以余额 / 签到不可用，面板会明说「只有模型凭据（没有网页会话）」，不假装读到了数字。
8. **`studioVersion` 参与门控**：客户端版本 3.4.4 的目录只有 7 条，3.6.0 起 8 条。请求头必须带 `clientType`/`studioVersion`/`User-Agent: AStudio/3.6.0`，否则目录会被裁。
9. 扫码取码 / 轮询两条 SSO 请求**不带 Cookie、不带 clientType/studioVersion**，只有 `Accept: application/json` 与 `User-Agent`；账号面请求才带 Cookie 与身份头。
10. **`github:` 安装走的是官方 CDN 中转，不是直连 GitHub**：宿主 `script/website.rs:35-47` 把 `api.github.com/repos/*/commits` 与 `codeload.github.com/*/tar.gz` 换成 `agent2api.dev/api/plugins/community/github-commits|github-archive`。那个中转会限流，实测返回 `429 {"error":"社区插件来源暂时限流，请稍后再试"}`（此时直连 GitHub 是 200）。碰上就隔几分钟重试，或改用下面的 `file:` 路径安装 —— 与仓库内容无关。

## 开发

```bash
node --test "test/*.test.mjs"     # 48 条：协议常量、SSO 状态机、目录合并、登录/续期/余额/签到全链路、老账号升级
python scripts/gen-icon.py        # 重画图标（需要 Pillow），同时写 assets/icon.png 与 icon.mjs
```

源码结构：

| 文件 | 职责 |
| --- | --- |
| `index.mjs` | 插件工厂（**唯一导出**，宿主会把每个导出函数当候选调用，多导出会炸整包） |
| `lib/endpoints.mjs` | 上游地址与常量（版本号、TTL、轮询间隔、兜底模型名） |
| `lib/http.mjs` | 信封解析、Cookie 头、身份头、超时、错误分类（`failed`/`expired`） |
| `lib/scan.mjs` | SSO 取码 / 轮询、提示文案状态机 |
| `lib/login.mjs` | 两种登录方式的 `authorize` |
| `lib/session.mjs` | 会话与凭据（`settle` / `accountSecret` / `renew`） |
| `lib/legacy.mjs` | 把 wasm 时代落盘的 api 型 secret 升级成 oauth 型（先验会话，换到新凭据才写盘） |
| `lib/catalog.mjs` | 两份目录的合并与兜底 |
| `lib/provider.mjs` | 运行时模型清单与 `config` 写入 |
| `lib/usage.mjs` / `lib/checkin.mjs` | 余额与签到 |
