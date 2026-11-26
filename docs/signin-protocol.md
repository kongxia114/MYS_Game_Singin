# 米游社 / 云游戏签到协议速查

本文记录当前脚本依赖的接口、签名和返回码，方便官方更新后快速排查。
最后验证时间：2026-11-26。

## 1. 接口域名

| 域名 | 用途 | 说明 |
|---|---|---|
| `api-takumi.miyoushe.com` | 米游社 Web API（当前） | 优先使用，登录态在 `.miyoushe.com` |
| `api-takumi.mihoyo.com` | 米游社 Web API（旧） | 登录态在 `.mihoyo.com`，作为回退 |
| `api-cloudgame.mihoyo.com` | 云原神 | `x-rpc-combo_token` 认证 |
| `cg-hkrpg-api.mihoyo.com` | 云崩铁 | `x-rpc-combo_token` 认证 |

`www.miyoushe.com` 只是前端页面，页面里的 `document.cookie` 只能看到 `.miyoushe.com` 的非 HttpOnly 字段。

## 2. 登录 Cookie

| 字段 | 必需 | 说明 |
|---|---|---|
| `cookie_token_v2` | 是 | HttpOnly，真正的登录凭证 |
| `ltoken_v2` | 建议 | HttpOnly，长期登录凭证 |
| `ltuid_v2` / `ltuid` | 是 | 账号 ID |
| `ltmid_v2` / `account_mid_v2` | 建议 | 账号 mid |
| `account_id_v2` / `account_id` | 可选 | 与 ltuid 相同 |

注意：

- `document.cookie` **看不到** `cookie_token_v2` / `ltoken_v2`，因为它们是 HttpOnly。
- `.miyoushe.com` 和 `.mihoyo.com` 是两套 Cookie，用哪套取决于请求的接口域名。
- 完整 Cookie 的获取方式：`npm run login`（见 `scripts/get-cookie.mjs`）。

## 3. 请求头

```text
Cookie: cookie_token_v2=...; ltoken_v2=...; ltuid_v2=...; ltmid_v2=...
DS: <t>,<r>,<md5>
x-rpc-app_version: 2.117.0
x-rpc-client_type: 5
x-rpc-device_id: <UUID>
User-Agent: ... miHoYoBBS/2.117.0
```

`DS` 计算：

```text
t = 当前 Unix 秒
r = 6 位随机字符串
DS = t + "," + r + "," + md5("salt=" + SALT + "&t=" + t + "&r=" + r)
```

- Web 端 Salt：`4751cb5557bfad5e249e58462c39fb0a`
- 客户端版本：`2.117.0`
- 两者会随米游社客户端更新而变化，可用 `MIHOYO_APP_VERSION`、`MIHOYO_WEB_SALT` 覆盖。

## 4. 签到流程

### 4.1 查询角色

```http
GET https://api-takumi.miyoushe.com/binding/api/getUserGameRolesByCookie?game_biz=hk4e_cn
```

`game_biz`：

| 游戏 | game_biz |
|---|---|
| 原神 | `hk4e_cn` |
| 星穹铁道 | `hkrpg_cn` |

返回 `data.list[]`，每项包含 `game_uid`、`region`、`nickname` 等。

### 4.2 查询签到状态

```http
GET https://api-takumi.miyoushe.com/event/luna/info?lang=zh-cn&act_id=<act_id>&region=<region>&uid=<game_uid>
```

原神需要额外带：

```text
x-rpc-signgame: hk4e
Origin: https://act.mihoyo.com
Referer: https://act.mihoyo.com/
```

返回字段：

- `is_sign`：今天是否已签到
- `total_sign_day`：连续签到天数
- `first_bind`：是否首次绑定，需要先手动签到一次

### 4.3 执行签到

```http
POST https://api-takumi.miyoushe.com/event/luna/sign
Content-Type: application/json;charset=utf-8

{
  "act_id": "e202311201442471",
  "region": "cn_gf01",
  "uid": "304784153"
}
```

活动 ID：

| 游戏 | act_id | 默认 region |
|---|---|---|
| 原神 | `e202311201442471` | `cn_gf01` |
| 星穹铁道 | `e202304121516551` | `prod_gf_cn` |

原神带 `x-rpc-signgame: hk4e`；星铁不需要该头，活动 ID 已经能区分。

## 5. 返回码

| retcode | 含义 | 处理 |
|---|---|---|
| `0` | 成功 | `data.success === 0` 表示签到成功 |
| `-5003` | 今天已签到 | 视为成功（already） |
| `-100` | 登录状态失效 | Cookie 不完整/过期/域名不对 |
| `-500001` | 参数缺失或风控 | 检查 `x-rpc-signgame`、版本、Salt |
| `0` + `data.success === 1` | 触发人机验证 | 返回 `gt` / `challenge`，需要验证码 |

## 6. 云游戏签到

云游戏使用 `x-rpc-combo_token`，不是米游社 Cookie。

```http
GET https://api-cloudgame.mihoyo.com/hk4e_cg_cn/wallet/wallet/get
x-rpc-combo_token: <token>
```

云崩铁：

```http
GET https://cg-hkrpg-api.mihoyo.com/hkrpg_cn/cg/wallet/wallet/get
x-rpc-combo_token: <token>
```

`retcode === 0` 即签到成功，`data.free_time.send_freetime` 是本次获得的分钟数。
`-100` 表示 Token 失效或账号有限制。

## 7. 最小流程

```text
for each cookie:
  roles = getRoles(cookie, game_biz)
  for each role:
    info = getSignInfo(cookie, act_id, role.region, role.game_uid)
    if info.is_sign: already
    else if info.first_bind: skip
    else: sign(cookie, act_id, role.region, role.game_uid)
```

## 8. 安全提醒

- 完整 Cookie 等于账号登录态，不要提交到 git、不要发到公开群。
- 优先使用 Cookie，不要把账号密码交给任何第三方代挂平台。
- Cookie 泄露后，退出米游社登录即可让旧会话失效。
