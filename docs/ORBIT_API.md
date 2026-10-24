# Orbit 鉴权中控 · 实例对接 API 文档

面向 NBEmail 内置 orbit-sdk 的对接契约。所有实例侧接口走 **HTTP + HMAC-SHA256 请求防篡改签名**，不依赖 Cookie / Bearer 会话。

- 默认监听：`http://<中控IP>:34444`
- 实例侧接口前缀：`/api/v1/`
- 管理员接口前缀：`/api/orbit/`（走管理员 Bearer 会话，见文末）
- 统一返回结构：`{ "code": 200, "message": "操作成功", "data": ..., "traceId": "..." }`

## 1. 机器指纹规则

| 项 | 说明 |
| --- | --- |
| 采集 | 主板 UUID + 主网卡 MAC |
| 算法 | `SHA256(盐 + 主板UUID + MAC)`，输出 64 位小写十六进制 |
| 持久化 | 写入 NBEmail 的 `data/machine.id`，重启优先读缓存，不再重新采集 |
| 虚拟化降级 | 读不到主板 UUID 时用 `CPUID + MAC` 生成，并在心跳里上报 `virtualized=true` |

指纹是实例的身份证，中控后台录入的必须是这串 64 位十六进制。

## 2. 密钥怎么来

1. 管理员在 Orbit 后台「实例管理」录入指纹，系统自动签发 `secret`（64 位十六进制 HMAC 密钥）。
2. 录入或「重新签发密钥」时前端会展示/复制一次，请写入 NBEmail 的配置。
3. 密钥只用于实例签名，泄漏后拿不到管理后台权限；怀疑泄漏用「重新签发密钥」轮换。

## 3. HMAC 签名规范

**所有** 实例接口（除未授权查询状态外）都必须带 4 个请求头：

| 请求头 | 含义 |
| --- | --- |
| `X-Orbit-Fingerprint` | 机器指纹（64 位小写十六进制） |
| `X-Orbit-Timestamp` | Unix 秒时间戳，允许 ±300 秒偏差 |
| `X-Orbit-Nonce` | 16 字节以上随机串，窗口内不可重复（防重放） |
| `X-Orbit-Signature` | 签名，见下 |

**签名算法：**

```text
canonical = METHOD + "\n" + PATH + "\n" + TIMESTAMP + "\n" + NONCE + "\n" + sha256Hex(body)
signature = hex( HMAC-SHA256(key = secret, message = canonical) )
```

- `METHOD` 大写（`POST` / `GET`）。
- `PATH` 是接口路径本身，例如 `/api/v1/heartbeat`（不含域名、不含查询串）。
- `body` 是原始请求体字节，空请求体也要用空串的 SHA256。
- 服务端校验失败返回 `401`，并记一条 `verify` 审计日志。

**Python 参考实现：**

```python
import hashlib, hmac, json, os, time, urllib.request

def signed_request(base, secret, fingerprint, method, path, payload):
    body = json.dumps(payload, separators=(',', ':')).encode()
    ts = str(int(time.time()))
    nonce = os.urandom(16).hex()
    canonical = "\n".join([method.upper(), path, ts, nonce,
                           hashlib.sha256(body).hexdigest()])
    sig = hmac.new(secret.encode(), canonical.encode(), hashlib.sha256).hexdigest()

    req = urllib.request.Request(base + path, data=body, method=method)
    req.add_header("Content-Type", "application/json")
    req.add_header("X-Orbit-Fingerprint", fingerprint)
    req.add_header("X-Orbit-Timestamp", ts)
    req.add_header("X-Orbit-Nonce", nonce)
    req.add_header("X-Orbit-Signature", sig)
    return req

# 用法
req = signed_request(
    "http://127.0.0.1:34444", secret, fingerprint,
    "POST", "/api/v1/heartbeat",
    {"fingerprint": fingerprint, "domainCount": 3, "todaySent": 100},
)
with urllib.request.urlopen(req) as resp:
    print(resp.read().decode())
```

## 4. 未授权状态查询（无需签名）

**`GET /api/v1/enroll/{fingerprint}`**

未授权实例全屏展示机器码时用。只查状态，不返回密钥，不需要签名头。

响应 `data`：

| 字段 | 说明 |
| --- | --- |
| `fingerprint` | 机器指纹 |
| `status` | `pending` 未登记/待授权 / `active` 正常 / `expired` 过期 / `banned` 拉黑 |
| `authorized` | 是否已授权放行 |
| `maxDomains` | 已配置的域名配额 |

```json
{ "code": 200, "data": { "fingerprint": "9f2c...", "status": "pending",
  "authorized": false, "maxDomains": 0 } }
```

## 5. 心跳上报（核心接口）

**`POST /api/v1/heartbeat`**

每 30~60 秒调一次。成功返回完整授权策略 `policy`，实例侧应缓存，断连时在宽限期内继续使用。

请求 `data` 字段：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `fingerprint` | string | 是 | 机器指纹（与请求头一致） |
| `hostname` | string | 否 | 主机名 |
| `ip` | string | 否 | 实例自报公网 IP |
| `version` | string | 否 | NBEmail 版本 |
| `domainCount` | int | 是 | 当前域名数量 |
| `todaySent` | int | 是 | 今日已发信计数 |
| `virtualized` | bool | 否 | 是否虚拟化降级 |
| `cpuId` / `primaryMac` | string | 否 | 降级指纹时的原始 CPUID / MAC |

响应 `data`（policy）：

| 字段 | 说明 |
| --- | --- |
| `status` | 授权状态 |
| `authorized` | 是否放行 |
| `reason` | 未授权时的中文原因 |
| `maxDomains` / `domainCount` / `domainRemaining` | 域名配额 / 当前 / 剩余（0 表示不限制时剩 0） |
| `dailySendQuota` / `todaySent` / `sendRemaining` | 日发信额度 / 已用 / 剩余（额度 0 表示不限） |
| `sendConcurrency` | 并发投递通道上限 |
| `expireTime` | 授权到期时间，空为永久 |
| `feature.domainEdit` | 域名新增/修改/删除开关 |
| `feature.dkimSign` | DKIM 签名开关 |
| `feature.outboundSend` | 出站发信总开关 |
| `feature.bounceRetry` | 退信重试开关 |
| `offlineAfter` | 心跳超时判离线秒数 |
| `graceSeconds` | 断连宽限期秒数（本地缓存继续生效的时长） |
| `policyRevision` | 策略版本号，变更后变化，可据此判断是否重新落库 |

## 6. 高危操作实时校验

**`POST /api/v1/verify`**

新增域名、批量投递等动作触发，不依赖本地缓存，每次回源强校验。被拒绝会记 `verify` 审计日志。

请求 `data`：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `fingerprint` | string | 机器指纹 |
| `action` | string | 见下表 |
| `amount` | int | 本次影响数量，`outbound_send` 时是本次投递封数 |
| `domainCount` | int | 实例当前域名数（域名动作时用于配额判断） |
| `todaySent` | int | 实例今日已发信数（发信动作时用于额度判断） |

`action` 取值：

| 值 | 校验内容 |
| --- | --- |
| `domain_add` | `domainEdit` 开关 + 域名配额未满 |
| `domain_edit` / `domain_delete` | `domainEdit` 开关 |
| `outbound_send` | `outboundSend` 开关 + 当日发信额度未耗尽 |
| `dkim_sign` | `dkimSign` 开关 |
| `bounce_retry` | `bounceRetry` 开关 |
| `login` | 实例状态（过期实例只允许 login 类只读动作） |

响应 `data`：

```json
{
  "allowed": true,
  "reason": "",
  "policy": { ... 同心跳返回的 policy ... }
}
```

- `allowed=true` → 放行
- `allowed=false` → 拒绝，`reason` 为中文原因（如「域名配额已满（上限 3）」「出站发信已关闭」）

## 7. 状态机与拦截时机

| 状态 | 登录后台 | 域名管理 | 出站投递 |
| --- | --- | --- | --- |
| `pending` 未授权 | ❌ | ❌ | ❌ |
| `active` 正常 | ✅ | 按开关/配额 | 按开关/额度 |
| `expired` 已过期 | ✅（只读） | ❌ | ❌ |
| `banned` 拉黑 | ❌ | ❌ | ❌ |

宽限期：实例断连 Orbit 后，本地缓存策略在 `graceSeconds` 内继续生效；超时后建议按「拒绝」处理并锁登录、锁域名、锁发信。

## 8. 管理员接口（仅作参考，一般由后台页面调用）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/api/auth/captcha` | 登录验证码（返回 `uuid` + `code`） |
| `POST` | `/api/auth/login` | 管理员登录 |
| `GET` | `/api/auth/me` | 当前管理员 |
| `POST` | `/api/auth/logout` | 退出 |
| `PUT` | `/api/auth/password` | 修改密码 |
| `GET` | `/api/setup/status` | 是否需初始化管理员 |
| `POST` | `/api/setup/admin` | 创建首个管理员 |
| `GET` | `/api/orbit/overview` | 总览统计 |
| `GET` | `/api/orbit/instances` | 实例分页列表 |
| `POST` | `/api/orbit/instances` | 录入实例（签发密钥） |
| `GET` | `/api/orbit/instances/{id}` | 实例详情（含完整密钥） |
| `PUT` | `/api/orbit/instances/{id}` | 修改配额/开关/到期时间 |
| `PUT` | `/api/orbit/instances/{id}/status` | 授权 / 拉黑 / 解禁 |
| `PUT` | `/api/orbit/instances/{id}/fingerprint` | 重置指纹 |
| `PUT` | `/api/orbit/instances/{id}/secret` | 重新签发密钥 |
| `GET` | `/api/orbit/audit-logs` | 审计日志分页 |

管理员接口用 `Authorization: Bearer <token>`，token 来自登录接口。