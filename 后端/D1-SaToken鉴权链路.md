# D1 · Sa-Token 鉴权链路（后端视角）

> **这份文档讲的是"后端拿到 token 之后干了什么"。**
>
> 姊妹篇 [D2-前后端调用链路.md](D2-前后端调用链路.md) 讲的是另一半——"前端怎么把 token 送出去"。两边合起来才是完整鉴权链路。
>
> **读法**：如果还没读 [D2](D2-前后端调用链路.md) 的**第零部分**（认证 ≠ 授权、三层权限、超管判定），先回去读那个。D1 全程假设你知道"权限码"是什么。
>
> 全文所有 `文件:行号` 都基于 `E:\Develop\project\zxcm-learn\`，可以直接跳过去看源码。
>
> **⭐ 本文档里带「实测」标记的数据，都是从正在运行的 Redis / 源码里当场抠出来的，不是从文档里抄的。**

> 🧭 **本文是附录 · 后端鉴权** · 姊妹篇：[D0 表地图](D0-底座表地图.md) · [D2 前后端链路](D2-前后端调用链路.md) · [D3 数据权限](D3-数据权限链路.md) · [E1 前端结构](../前端/E1-zxcm-ui工程结构.md) · 🏁 [回到总入口](../README.md)
>
> 📍 **最该读本文的时机**：讲「为什么登录能成功、token 是什么」时（[M0 · 骨架](M0-骨架（总览）.md) / [M2 审批流](施工单/M2-审批流与事件状态机.md)）。

---

# 第零部分 · 一张图看懂全链路

```
════════ 阶段一：登录，把 token 造出来 ════════

浏览器  POST /login {username, password, code, uuid}
   │
   ├─→ SysLoginController
   │        │
   │        └─→ SysLoginService.login()
   │                 ├─ 校验验证码     validateCaptcha()        (读 Redis)
   │                 ├─ 查库           loadUserByUsername()     (查 sys_user)
   │                 ├─ 校验密码       BCrypt.checkpw()
   │                 └─ buildLoginUser()                        (组装 LoginUser)
   │                          │
   │                          └─→ LoginHelper.loginByDevice(loginUser, DeviceType.PC)
   │                                   │
   │                                   ├─ StpUtil.login(loginUser.getLoginId(), "pc")
   │                                   │      │
   │                                   │      ├─ createLoginSession()
   │                                   │      │    ├─ createTokenValue()  → 造一个 JWT
   │                                   │      │    └─ saveTokenToIdMapping() / addTokenSign()
   │                                   │      └─ 触发 SaTokenListener.doLogin()
   │                                   │              └─→ UserActionListener  (写"在线用户"记录)
   │                                   │
   │                                   └─ setLoginUser(loginUser)
   │                                          └─ StpUtil.getTokenSession().set("loginUser", ...)
   │
   │        ★ 以上所有"存"的动作，最终都落到 Redis（db 5）
   │
   └─→ 返回 {token: "eyJ..."}

════════ 阶段二：带 token 请求，校验它 ════════

浏览器  GET /system/user/listWithP   头: token: Bearer eyJ...
   │
   ├─→ 拦截器 1：SaRouteInterceptor     ← 管"登录了吗"（认证）
   │
   ├─→ 拦截器 2：SaAnnotationInterceptor ← 管"有权限吗"（授权）
   │        翻出方法上的 @SaCheckPermission("system:user:list")
   │        └─→ StpUtil.checkPermission(...)
   │             └─→ SaInterfaceImpl.getPermissionList()      ← SPI，项目实现的
   │                  └─→ LoginHelper.getLoginUser()
   │                       ├─ 先查 ThreadLocal（同一次请求内的缓存）
   │                       └─ 没有就读 Redis 的 token-session
   │                            └─ 拿到 LoginUser.menuPermission
   │                                 ├─ ["*:*:*"]         → 超管，全放行
   │                                 └─ ["system:user:list", ...] → 逐个比对
   │
   └─→ 校验通过，进入 Controller
```

**一句话总结**：**登录时把"我是谁 + 我能干什么"整个塞进 Redis；此后每次请求只是把这份快照读出来比对，不再查库。**

---

# 第一部分 · 登录：token 是怎么产生的

## 1.0 前置：验证码的 `uuid` 从哪来

登录请求的 payload 长这样：

```json
{"username":"zxcmAdmin","password":"123456","code":"3",
 "uuid":"0f182307568548a2b95405c31108c754","tenantId":"00000000"}
```

那个 `uuid` 不是前端造的，是**上一次 `/captchaImage` 请求时后端发的**（`CaptchaController.java:101`）：

```java
String uuid = IdUtil.simpleUUID();          // Hutool: UUID.randomUUID() 去掉横线 → 32 位十六进制
String verifyKey = Constants.CAPTCHA_CODE_KEY + uuid;   // "captcha_codes:" + uuid   (:102)
```

完整往返：

```
① 前端打开登录页 → GET /captchaImage
② CaptchaController.getCode()
     uuid = IdUtil.simpleUUID()                              (:101)
     算答案（MATH 类型走 getCodeResult，如 "3*4" → "12"）    (:111)
     Redis.set("captcha_codes:"+uuid, 答案, TTL 2分钟)        (:112)
     返回 { uuid, img }  ← 只回图片 base64，**不回答案**      (:113-114)
③ 前端：img 画成图片给用户看；uuid 存在登录页 data 里
④ 用户照图输数字 → 拼进 payload POST /login
⑤ SysLoginService.validateCaptcha()                          (:197-209)
     用同一个 uuid 拼 key → 取答案 → 第 200 行立刻 deleteObject
     取不到 → CaptchaExpireException（已过期）
     对不上 → CaptchaException，比较用 equalsIgnoreCase（大小写不敏感）
```

**实测**：现场调一次 `/captchaImage` 拿到 `uuid=516f29bf151c4ffc87588a7e737df8f2`，
`redis-cli -n 5 GET captcha_codes:516f29bf...` → `"0"`，`TTL` → `119` 秒。

### 三个设计点

| 设计 | 为什么 |
|---|---|
| **答案只在服务端**，前端只给 uuid | 若连答案一起返回，JS 读出来自动填，验证码完全失效 |
| **uuid 用随机值**，不用自增/时间戳 | `/captchaImage` 是匿名接口（`application.yml:162` 白名单），可枚举的 id 能被猜到 → 拿别人的验证码答案撞库 |
| **一次性 + 2 分钟 TTL** | `:200` 在校验后**无论对错都删**，一个 uuid 只能提交一次，重试必须刷新；过期即作废 |

常量在 `Constants.java:81`（`CAPTCHA_CODE_KEY = "captcha_codes:"`）和 `:111`（`CAPTCHA_EXPIRATION = 2` 分钟）。

> `sys_config` 里 `sys.account.captchaOnOff` 为 false 时，`:97-99` 直接 return，**连 uuid 都不生成**，前端也就不带 `code`/`uuid` 了。

### ⚠️ 安全边界（实测，值得在面试里讲）

这个接口**没有任何保护**：`/captchaImage` 在 `security.excludes` 白名单里（匿名可调），
且**没加限流**——框架自带 `@RateLimiter` 注解和 `RateLimiterAspect` 切面
（支持 `LimitType.IP` → `ServletUtils.getClientIP()`），但**全项目 0 处使用**。

「黑客狂刷会不会把 Redis 刷满？」——**CPU 会先满，而且差一个数量级**。实测数据：

| 实测项 | 数值 |
|---|---|
| 单个 key 内存开销 | **169.4 字节**（key = `captcha_codes:` + 32 位 uuid，value 为 2 位数字，`PX 120000`）|
| **每请求 CPU 开销** | **7.68 ms**（`(loadCpu-idleCpu) / 请求数`，见下面的测法）|
| 单机吞吐 | 20 逻辑核 ÷ 7.68 ms = **~2,600 req/s** 上限 |
| 响应体积 / 单次耗时 | ~3.2 KB / 14–24 ms |
| 本项目 Redis 配置 | **`maxmemory = 0`（无上限）**、`maxmemory-policy = noeviction` |

**测法**：`Get-Process <pid>).TotalProcessorTime` 采样空转 5s 得基线（0.005 core），
再 48 线程打 6s（2,786 个请求、441 req/s、消耗 3.39 core）→ 差值 ÷ 请求数 = 7.68 ms。
> ⚠️ 这是在 **IDEA Debug 模式 + async-profiler（`event=wall, interval=10ms`）** 下测的，属**上界**；
> 生产 JIT 后更低。「CPU 先满」这个结论不随该系数改变。

### 为什么是 CPU 先满（机理，不是巧合）

两种资源的性质根本不同：

| | CPU | Redis 内存 |
|---|---|---|
| 消耗方式 | **串联**——每个请求必须独占 CPU 时间片才能完成 | **累积**——key 只要没过期就一直留着 |
| 上限形式 | **速率上限**（req/s）：`核数 ÷ 每请求 CPU` | **存量上限**：`速率 × TTL` |
| 扩展性 | 无状态、可水平扩容（HPA 加 Pod） | 有状态、单点/分片，扩容要动集群拓扑 |
| 打满的代价 | 慢（可扩容缓解） | 全站掉线 + 缓存击穿打到 DB（级联故障） |

因为 CPU 是**速率**约束、内存是**存量**约束，低速率就能把 CPU 顶死，而内存要高速率堆够久才满。

**代入这台机器**：20 核满打满算 2,600 req/s → 稳态堆量 = `2,600 × 120s × 169.4B ≈ 50 MB`。
也就是说**即使 CPU 100% 打满，Redis 也只涨 50 MB**。要填到 512 MB 需要 **203 个核**——差 10 倍。

| 想堆到 | 需要稳态 key 数 | 需要持续速率 | 按 7.68ms/请求需要多少核 |
|---|---|---|---|
| 64 MB | 39.6 万 | 3.3k req/s | 25 |
| 512 MB | 317 万 | 2.6 万 req/s | 203 |
| 2 GB | 1268 万 | 10.6 万 req/s | 811 |

### ⭐ 企业级部署下结论会翻转

单机时 **CPU 饱和自己就是一道事实限流器**（刷不过 2,600 req/s，Redis 就吃不到更多写）。
但企业是**多实例 + HPA**：前端 Nginx/LB 后面挂 N 个 Pod。

> **每个 Pod 有独立的 CPU 预算，但它们写的是同一个 Redis。**
> 所以水平扩容把**写入上限放大了 N 倍，Redis 的内存上限一点没变** —— 稳态堆量 = `N × 单机吞吐 × TTL × 169.4B`。

假设 100 个 Pod 全被刷满（约 26 万 req/s），120 秒稳态就是 **5.3 GB**——这才是真会炸的场景。
**这就是为什么「CPU 先满」这个答案只在单体部署下成立，在微服务下要重新算。**

另外，企业里 Redis 的后果比单机严重得多：会话和缓存常在同一实例，
Redis 一挂 = **全员掉线 + 缓存击穿 → 流量直接打到 MySQL → 级联故障**。

### 分层防线（企业实践）

| 层 | 手段 | 作用 |
|---|---|---|
| 边缘 | 云 WAF / CDN / 高防 IP | **主战场**。匿名接口的 CC 攻击在这一层就该被拦掉 |
| 网关 | Nginx `limit_req`（令牌桶）/ `limit_conn`；或 APISIX/Kong | 按 IP + 接口限流 |
| 应用 | `@RateLimiter`（本项目自带） | 兜底，防单机脚本 |
| Redis | `maxmemory` + `allkeys-lru`；**会话 Redis 与缓存 Redis 物理隔离** | 即使被刷也只淘汰缓存，不踢人下线 |
| 架构 | 换**第三方行为验证码**（极验/阿里云/腾讯云） | 服务端不再渲染图片，**CPU 热点直接消失**，风控外部化——最彻底 |

> ⚠️ **反直觉点**：应用层 `@RateLimiter` 用 Redisson `RRateLimiter`，**计数器就写在同一个 Redis 上**——
> 被刷时限流本身也在加剧 Redis 压力；且按 IP 限流遇僵尸网络时 key 数随攻击 IP 数爆炸。
> 所以**应用层限流能防单机脚本，防不了 DDoS，DDoS 必须在边缘/网关拦**。

加固只需一行：

```java
@RateLimiter(key = "captcha", time = 60, count = 20, limitType = LimitType.IP)
@GetMapping("/captchaImage")
```

---

## 1.1 起点：登录接口

`SysLoginController` 收到 `POST /login`，调到 `SysLoginService`。

主流程在 `SysLoginService.java:84-92` 和 `:94-159`，从 `:94` 那个重载往下看，依次做了这些事：

| 步骤 | 行号 | 干了什么 | 读/写哪个存储 |
|---|---|---|---|
| 1 | `:97` | 查登录错误次数 | 读 Redis `login_error:{tenantId}:{username}` |
| 2 | `:104` | `loadUserByUsername()` 查用户 | **读 MySQL `sys_user`** |
| 3 | `:106` | `BCrypt.checkpw()` 校验密码 | — |
| 4 | `:121-148` | 非管理员才校验"3 个月未登录""临时密码过期" | — |
| 5 | `:152` | `buildLoginUser(user)` 组装 `LoginUser` | — |
| 6 | `:154` | **`LoginHelper.loginByDevice(loginUser, DeviceType.PC)`** | — |
| 7 | `:157` | 记录登录日志、更新 `LOGIN_DATE` | 写 MySQL |
| 8 | `:158` | `return StpUtil.getTokenValue()` | 取回刚生成的 token |

第 6 步是分水岭——**从这里开始就进入 Sa-Token 的世界了**。

## 1.2 ⭐ `StpUtil.login()` 的参数不是 userId

这是最容易看错的一行。`LoginHelper.java:78-82`：

```java
public static void loginByDevice(LoginUser loginUser, DeviceType deviceType) {
    LOGIN_CACHE.set(loginUser);
    StpUtil.login(loginUser.getLoginId(), deviceType.getDevice());   // ← 注意参数
    setLoginUser(loginUser);
}
```

Sa-Token 源码里这个参数叫 `id`（登录标识），**不是数字类型的用户 ID**。传进去的实际是个字符串，来自 `LoginUser.java:135-137`：

```java
public String getLoginId() {
    return tenantId + LoginHelper.JOIN_CODE + userType + LoginHelper.JOIN_CODE + userId;
}
```

`JOIN_CODE` 是 `":"`（`LoginHelper.java:43`）。所以 `zxcmAdmin` 登录时传进去的是：

```
00000000:sys_user:1
└─租户─┘ └用户类型┘ └用户ID┘
```

**这就是 D2 里 JWT payload 解出 `"loginId":"00000000:sys_user:1"` 的原因**——它不是被解析出来的，是一开始就这么拼好递进去的。

### 为什么要拼三段

只传 `1` 会串号：

- **多租户**：租户 A 的用户 1 和租户 B 的用户 1 是两个人
- **多用户类型**：`sys_user`（后台）和 `app_user`（App 端）共用一张用户表（`LoginHelper.java:30-36` 的类注释就在讲这件事）

`SaInterfaceImpl.java:31-36` 拿回登录用户后也要**先看前缀判断 `userType`**，才知道该返回哪一套权限。而 `LoginHelper.getUserId()`（`:120-138`）反着来——**从右往左取最后一段**当 userId，注释写着「用户id在总是在最后」（`:128`）。

## 1.3 token 本体：一个 JWT

`SaTokenConfig.java:82-86` 换掉了 Sa-Token 默认的 token 生成策略：

```java
@Bean
public StpLogic getStpLogicJwt() {
    // Sa-Token 整合 jwt (Style模式)
    return new StpLogicJwtForStyle();
}
```

跟进 `StpLogic.login(id, loginModel)` 内部（1.29.0 字节码，可自行用 IDE 反编译核对）：

```
login(id, loginModel)
 ├─ createLoginSession(id, loginModel)
 │   ├─ createTokenValue(id, device, timeout, extraData)
 │   │   └─ SaJwtUtil.createToken(id, extraData, jwtSecretKey())   ★ 拼 JWT
 │   ├─ saveTokenToIdMapping(token, id, timeout)                   ★ 写 token→loginId
 │   └─ SaSession.addTokenSign(token, device)                      ★ 挂到账号会话
 ├─ setTokenValue(token, cookieTimeout)                            写响应（本项目走 header）
 └─ SaManager.getSaTokenListener().doLogin(...)                    ★ 触发监听器
```

`SaJwtUtil.createToken` 把 `id` 放进 JWT 的 `loginId` claim，再用 `application.yml:125` 的 `jwt-secret-key` 签名。**JWT 本身不自带加密，payload 是明文 base64——随便找个在线 JWT 解码站点就能读出 `loginId`。** 这也是为什么 demo 里的 secret 必须换掉。

相关配置（`application.yml:101-127`）：

| 配置 | 值 | 作用 |
|---|---|---|
| `token-name` | `token` | **请求头名 + Cookie 名 + Redis key 前缀**（注意这三个都是一份配置） |
| `token-prefix` | `Bearer` | 头的值要写成 `Bearer <jwt>` |
| `timeout` | `186400` | token 绝对过期（约 2.16 天） |
| `activity-timeout` | `28800` | 8 小时无操作就过期 |
| `is-concurrent` | `true` | 允许同一账号多端同时在线 |
| `is-share` | `false` | 每次登录**新建**一个 token，不复用 |
| `is-read-cookie` | `false` | 不从 Cookie 读 token（前端走 header） |
| `jwt-secret-key` | `please-change-...` | ⚠️ 开源占位值，生产必须换 |

## 1.4 存进 Redis：SPI 换实现

Sa-Token 默认把会话存在**内存**里（`SaTokenDaoDefaultImpl`），重启就没，也没法多实例。

本项目用 `PlusSaTokenDao`（`ruoyi-framework/.../satoken/dao/PlusSaTokenDao.java`）换掉了它。注意这个类**没有 XML、没有 mapper**，每个方法都是三行：

```java
@Component
public class PlusSaTokenDao implements SaTokenDao {
    @Override
    public String get(String key) { return RedisUtils.getCacheObject(key); }        // 读
    @Override
    public void set(String key, String value, long timeout) { ...RedisUtils... }    // 写
    @Override
    public void delete(String key) { RedisUtils.deleteObject(key); }                // 删
    ...
}
```

完整的落地链条：

```
Sa-Token 内核 (StpLogic)
     ↓ 只认接口 SaTokenDao
PlusSaTokenDao                     ← 项目实现，@Component
     ↓
RedisUtils                         ← ruoyi-common 的工具类
     ↓ 用的是 RedissonClient（不是 RedisTemplate）
       RedisUtils.java:39  private static final RedissonClient CLIENT = SpringUtils.getBean(RedissonClient.class);
     ↓ codec
JsonJacksonCodec                   ← RedisConfig.java:69  setCodec(JsonJacksonCodec.INSTANCE)
     ↓
Redis  db 5 @ 192.168.56.10
```

**为什么值里全是 `@class` 字段**：`JsonJacksonCodec` 开了默认类型信息（default typing），不写 `@class` 反序列化时不知道要还原成哪个类。你在 Redis 里看到的 `{"@class":"cn.dev33.satoken.session.SaSession", ...}` 就是它干的。

## 1.5 ⭐ 实测：一次登录在 Redis 里落下 5 个 key

虚拟机开着的时候直接查（`db 5`，密码 `root`）：

```bash
/e/Redis-x64-3.2.100/redis-cli -h 192.168.56.10 -a root -n 5 KEYS "token*"
```

> ⚠️ **两个坑**：
> 1. 前缀是 **`token:`** 不是 `satoken:`——因为 `token-name` 被设成了 `token`，而 Sa-Token 的 key 前缀用的就是这个值。（Sa-Token 默认值是 `satoken`，所以网上文章里大多是 `satoken*`。**照抄会查不到东西。**）
> 2. Windows 版 `redis-cli` 输出带 `\r`，写脚本遍历 key 时要 `tr -d '\r'` 去掉，否则拼出来的 key 永远查不到。

实测结果（2026-09-26，`zxcmAdmin` 登录一次）：

| # | key | 值（实测） | 谁写的 | 作用 |
|---|---|---|---|---|
| 1 | `token:login:token:<JWT>` | `"00000000:sys_user:1"` | `saveTokenToIdMapping` | **token → loginId** 映射。校验时靠它反查"这是谁" |
| 2 | `token:login:token-session:<JWT>` | 一整个 `SaSession`，含 `loginUser` 对象 | `LoginHelper.setLoginUser()` | **最核心的一个**——装着"我是谁 + 我能干什么"的快照 |
| 3 | `token:login:last-activity:<JWT>` | `"1790242473651"`（毫秒时间戳） | `setLastActivityToNow` | 支撑 `activity-timeout: 28800` 的"8 小时无操作过期" |
| 4 | `token:login:session:00000000:sys_user:1` | `SaSession`，含 `tokenSignList` | `addTokenSign` | **账号级**会话：这个账号当前有哪些 token 在线 |
| 5 | `online_tokens:<JWT>` | `UserOnlineDTO` | `UserActionListener.java:78` | 给后台「在线用户」页面用的展示数据 |

### key 1 和 key 2 的区分（很容易混）

- `token:login:token:<JWT>` —— **key 是 token，value 是 loginId**。回答"这个 token 属于谁"
- `token:login:token-session:<JWT>` —— **key 是 token，value 是一大坨用户数据**。回答"这个人有哪些权限"

### key 2 里面到底装了啥（实测）

```json
{"@class":"cn.dev33.satoken.session.SaSession",
 "createTime":1790242330600,
 "dataMap":{"loginUser":{"@class":"com.ruoyi.common.core.domain.model.LoginUser",
     "deptIds":[[2075148338088488961]],
     "deptNames":["测试部门"],
     "menuPermission":["*:*:*"],          ← 权限码就在这
     "rolePermission":["admin"],
     "nickName":"超级管理员",
     "roles":[...]}}}
```

**⭐ 这就是 D2 里 `/getInfo` 返回 `permissions:["*:*:*"]` 的同一个东西**——只不过 D2 讲的是"这份数据从后端传到前端"，D1 讲的是"这份数据本身存在 Redis 的哪里"。

> 📌 **超管为什么是 `*:*:*`**：`SysPermissionService.java:61-62` 有个短路——管理员压根不查 `sys_menu`，直接返回通配符。所以**想验证"权限码真的来自 sys_menu 表"，必须用一个非管理员账号**。

### key 4 为什么需要（账号级会话）

因为 `is-concurrent: true` + `is-share: false`：同一个账号允许多处登录，且每次登录都是**新 token**。那"把这个账号全部踢下线"要怎么做？——找到这个账号的会话，遍历里面的 `tokenSignList` 挨个删。这就是 key 4 存在的理由。

实测它的 `tokenSignList` 长这样：

```json
"tokenSignList":[{"device":"pc","value":"<JWT-1>"},
                 {"device":"pc","value":"<JWT-2>"}]
```

### 一个值得注意的残留

实测时发现**两个** `last-activity` key，其中一个 TTL 是 `-1`（永不过期）、时间戳是几天前的。这是历史登录留下的残留，不影响使用，但说明**「登出」不会把 last-activity 一起清掉**。

---

# 第二部分 · 请求：token 是怎么被校验的

## 2.1 两道拦截器，各管一件事

`SaTokenConfig.java:52-80` 把两道拦截器都挂在了 `/**` 上：

```java
public void addInterceptors(InterceptorRegistry registry) {
    // ① 管"登录了吗"
    registry.addInterceptor(new SaRouteInterceptor((request, response, handler) -> {
        SaRouter.match("/**")
            .notMatch(securityProperties.getExcludes())     // 白名单
            .check(() -> { Long userId = LoginHelper.getUserId(); ... });
    }) { ... }).addPathPatterns("/**");

    // ② 管"有权限吗"
    registry.addInterceptor(new SaAnnotationInterceptor()).addPathPatterns("/**");
}
```

| | ① `SaRouteInterceptor` | ② `SaAnnotationInterceptor` |
|---|---|---|
| 管什么 | **认证**（登录了吗） | **授权**（有权限吗） |
| 依据 | `application.yml` 的 `security.excludes` 白名单 | 方法/类上的**注解** |
| 失败报错 | `401 认证失败，无法访问系统资源` | `403 没有访问权限，请联系管理员授权` |
| 对应 D2 的 | 0.1 节表格左列 | 0.1 节表格右列 |

## 2.2 白名单：`security.excludes`

`application.yml:158` 起是白名单，D2 里提到的免登录接口都在这：

```yaml
security:
  excludes:
    - /login
    - /register
    - /logout
    - /captchaImage
    ...
```

⚠️ 注意 `:158` 是**全局**的 `security.excludes`；文件里 `:316` 还有一个**另一个组件的** `excludes: /system/notice`，别混。这两个同名不同父，改错地方不生效——排查"为什么白名单没起作用"时先确认自己改的是哪个。

## 2.3 ⭐ 纠正一个很多人会讲错的地方：`@SaCheckPermission` 不是 AOP

网上文章（包括本项目计划文档里我写的那句提示）经常说"去找 Sa-Token 的 AOP 切面"。**这个项目里不是 AOP，是 Spring MVC 的拦截器。**

验证方法很直接——看打出来的 jar 里到底有没有 AOP 那个包：

```bash
unzip -l ruoyi-admin/target/zxcm-admin.jar | grep sa-token
```

实测结果：

```
BOOT-INF/lib/sa-token-spring-boot-starter-1.29.0.jar
BOOT-INF/lib/sa-token-servlet-1.29.0.jar
BOOT-INF/lib/sa-token-jwt-1.29.0.jar
BOOT-INF/lib/sa-token-core-1.29.0.jar
```

**没有 `sa-token-spring-aop`。** 那个包含 `SaCheckAspect` 切面的 artifact 根本没被打进来。

真正的触发路径（反编译 `SaAnnotationInterceptor` 字节码得到）：

```java
public boolean preHandle(HttpServletRequest req, HttpServletResponse resp, Object handler) {
    if (!(handler instanceof HandlerMethod)) return true;
    Method method = ((HandlerMethod) handler).getMethod();
    SaStrategy.me.checkMethodAnnotation.accept(method);   // ← 全部逻辑在这里
    return true;
}
```

再往下：

```
checkMethodAnnotation(method)
 ├─ checkElementAnnotation( method.getDeclaringClass() )   ← 先看类上的注解
 └─ checkElementAnnotation( method )                       ← 再看方法上的注解
      └─ SaManager.getSaTokenAction().validateAnnotation(element)
           └─ 按注解类型分派：
                @SaCheckLogin      → 校验登录
                @SaCheckRole       → 校验角色
                @SaCheckPermission → 校验权限码   ★ 你要找的就是这个
                @SaCheckSafe       → 二级认证
```

**所以正确的说法是**：注解校验由 `SaAnnotationInterceptor`（`SaTokenConfig.java:79` 注册）在 `preHandle` 里通过反射读取注解完成的。

> 💡 面试时如果被问"Sa-Token 的注解怎么生效的"，答"拦截器 + 反射"比答"AOP 切面"更准确——至少对本项目是这样。（Sa-Token 另有 `sa-token-spring-aop` 可选模块，但**不是必需**，本项目就没用。）

## 2.4 权限码是从哪读出来的：`SaInterfaceImpl`

`@SaCheckPermission("system:user:list")` 最终会调 `StpUtil.checkPermission("system:user:list")`，它需要一个"这个人有哪些权限"的名单——这个名单由 **SPI 接口 `StpInterface`** 提供，项目的实现是 `SaInterfaceImpl.java:28-38`：

```java
@Override
public List<String> getPermissionList(Object loginId, String loginType) {
    LoginUser loginUser = LoginHelper.getLoginUser();       // ★ 从会话里拿
    UserType userType = UserType.getUserType(loginUser.getUserType());
    if (userType == UserType.SYS_USER) {
        return new ArrayList<>(loginUser.getMenuPermission());
    } else if (userType == UserType.APP_USER) {
        // app端权限返回 自行根据业务编写
    }
    return new ArrayList<>();
}
```

`getRoleList()`（`:40-50`）同理，返回 `loginUser.getRolePermission()`。

### ⭐ 这里有个关键结论

**`SaInterfaceImpl` 全程没有查数据库。** 它只是把 `LoginUser` 对象里的两个 `Set` 掏出来。

那 `LoginUser` 从哪来？`LoginHelper.getLoginUser()`（`LoginHelper.java:94-100`）：

```java
public static LoginUser getLoginUser() {
    LoginUser loginUser = LOGIN_CACHE.get();          // ① 先查 ThreadLocal
    if (loginUser != null) return loginUser;
    return (LoginUser) StpUtil.getTokenSession().get(LOGIN_USER_KEY);   // ② 再读 Redis
}
```

两级缓存：

| 级 | 位置 | 生命周期 | 为什么要有 |
|---|---|---|---|
| ① | `LOGIN_CACHE`（ThreadLocal） | **一次请求** | 一次请求里可能十几处要拿当前用户，避免十几次 Redis 往返 |
| ② | Redis 的 `token:login:token-session:<JWT>` | **整个登录期**，跨请求跨实例 | 真正的数据源 |

第 ① 级的清理写在 `SaTokenConfig.java:73-77`：

```java
@Override
public void afterCompletion(...) {
    LoginHelper.clearCache();     // 请求结束必须清，否则线程池复用时串号
}
```

> ⚠️ **这是一个经典坑**：ThreadLocal + 线程池 = 如果不清理，下一个请求复用到同一个线程，就会读到上一个用户的身份。`afterCompletion` 就是干这个的。

## 2.5 权限码的完整来路（回答计划文档 Step 5 的问题 2）

把"登录时"和"请求时"拼起来，`system:user:list` 这个权限码的完整旅程：

```
① 数据库        sys_menu.PERMS = 'system:user:list'
                    ↓  （只在【登录时】查一次）
② 登录          SysLoginService.buildLoginUser()
                    └─ permissionService.getMenuPermission(user)          SysPermissionService.java:58
                         ├─ 是超管 → 直接返回 "*:*:*"，不查库    ← 短路，:61-62
                         └─ 不是超管 → menuService.selectMenuPermsByUserId(userId, tenantId)   :64
                    ↓
③ 内存          LoginUser.menuPermission : Set<String>
                    ↓  （LoginHelper.setLoginUser）
④ Redis         token:login:token-session:<JWT>  →  dataMap.loginUser.menuPermission
                    ↓  （每次请求）
⑤ 拦截器        SaAnnotationInterceptor → @SaCheckPermission("system:user:list")
                    └─ StpUtil.checkPermission(...)
                         └─ SaInterfaceImpl.getPermissionList()              SaInterfaceImpl.java:29
                              └─ LoginHelper.getLoginUser()
                                   ├─ ThreadLocal 命中 → 直接返回
                                   └─ 未命中 → 读 Redis 的 token-session
                    ↓
⑥ 比对          取出 menuPermission 这个 Set，看里面有没有 "system:user:list"
                （超管的 Set 里是 "*:*:*"，Sa-Token 的通配符匹配直接放行）
```

**⭐ 最值得记住的一点**：权限码**只在登录那一刻查一次库**，之后全程活在 Redis 里。

这带来一个非常实用的推论——**改了角色的权限，已登录的人不会立刻生效**，必须重新登录（或后台强制踢下线）才会刷新。这也是为什么企业后台改完权限常提示"请重新登录"。

---

# 第三部分 · 三个 SPI：Sa-Token 的可插拔设计

Sa-Token 之所以能"脱离业务独立使用"，关键在于它把三个最容易变的点做成了接口（SPI），启动时由 `SaBeanInject` 从 Spring 容器里**捞实现类**替换掉自带的默认实现。

反编译 `SaBeanInject` 可以看到它在启动时干的事：

```
SaManager.setConfig(...)           ← SaTokenConfig
SaManager.setSaTokenDao(...)       ← ★ PlusSaTokenDao
SaManager.setStpInterface(...)     ← ★ SaInterfaceImpl
SaManager.setSaTokenListener(...)  ← ★ UserActionListener
...
```

对照表：

| SPI 接口 | 项目实现类 | 位置 | 干什么 | 不实现会怎样 |
|---|---|---|---|---|
| `SaTokenDao` | `PlusSaTokenDao` | `ruoyi-framework/.../satoken/dao/` | 会话存哪（本项目=Redis） | 存内存，重启失效、多实例不共享 |
| `StpInterface` | `SaInterfaceImpl` | `ruoyi-framework/.../satoken/service/` | 权限码和角色从哪来 | 所有 `@SaCheckPermission` 都失败 |
| `SaTokenListener` | `UserActionListener` | `ruoyi-framework/.../listener/` | 登录/登出/踢人时的回调 | 「在线用户」列表是空的 |

三者都只是加个 `@Component`，**没有任何一处显式的 `new` 或 `@Bean` 注册**——Sa-Token 自己在启动时扫容器。这是典型的**SPI + 依赖注入**设计。

### `UserActionListener` 具体做了什么（`UserActionListener.java:53-83`）

`doLogin()` 里组装一个 `UserOnlineDTO`（IP、浏览器、操作系统、登录时间、tokenId、用户名、租户号）写进 Redis：

```java
RedisUtils.setCacheObject(Constants.ONLINE_TOKEN_KEY + tokenValue, dto,
                          Duration.ofSeconds(tokenConfig.getTimeout()));
```

`Constants.ONLINE_TOKEN_KEY = "online_tokens:"`（`Constants.java:96`）——**这就是实测里 key 5 的来源**。

另外还有四个回调，全都只做一件事：**删掉对应的 `online_tokens:*`**：

| 回调 | 触发时机 |
|---|---|
| `doLogout` | 主动登出 |
| `doKickout` | 被管理员踢下线 |
| `doReplaced` | 被顶下线 |
| `doDisable` / `doUntieDisable` / `doCreateSession` / `doLogoutSession` | 本项目未实现（空方法） |

---

# 第四部分 · 亲手验证

## 4.1 断点清单（IDEA Debug 模式）

按顺序打，跟着走一遍：

| 顺序 | 文件:行 | 观察什么 |
|---|---|---|
| 1 | `SysLoginService.java:152` | `user` 对象的 `userId` / `userType` / `tenantId` 三个值 |
| 2 | `LoginHelper.java:80` | 求值 `loginUser.getLoginId()` → 应该是 `00000000:sys_user:1` |
| 3 | `StpLogic` 的 `createLoginSession` | 参数 `id` 是不是同一个字符串 |
| 4 | `SaInterfaceImpl.java:30` | **登录时** `loginUser.getMenuPermission()` → `["*:*:*"]` |
| 5 | `SaInterfaceImpl.java:30` | **之后每次请求**这个断点都会再进一次——但**不会**再访问 MySQL |

第 4、5 两处对比着看，能直观感受到"登录时查一次，之后读缓存"。

> 📌 第 5 条要验证"不再查库"，可以在 IDE 的 Database 面板开着 MySQL 的查询日志，或者看控制台 SQL 输出（项目开了 `logImpl: StdOutImpl`，`application.yml:241`）。

## 4.2 Redis 命令

```bash
R="/e/Redis-x64-3.2.100/redis-cli -h 192.168.56.10 -a root -n 5"

# 看全部 token 相关 key（★ 前缀是 token 不是 satoken）
$R KEYS "token*"

# 看 token → loginId 映射
$R GET "token:login:token:<JWT>"

# 看会话里装了啥（最有用的一条，能看到 menuPermission）
$R GET "token:login:token-session:<JWT>"

# 看这个账号有几个 token 在线
$R GET "token:login:session:00000000:sys_user:1"

# 看在线用户记录
$R GET "online_tokens:<JWT>"
```

## 4.3 实验：放行一个受保护的接口（计划文档 Step 4）

**改动 → 观察 → 改回去**，这是建立鉴权直觉最快的方法。

1. 打开 `ruoyi-admin/src/main/resources/application.yml`，在 `security.excludes`（`:158` 起）里加一行 `- /system/user/list`
2. 重启后端
3. 不带 token 直接请求：
   ```bash
   curl -s "http://localhost:8080/system/user/list?pageNum=1&pageSize=1" | head -c 200
   ```
   **预期**：能拿到数据（而不是 `{"code":401,...}`）
4. **把那一行删掉**，重启，再请求一次 → 确认变回 401

> ⚠️ 第 4 步别忘。忘了就等于给系统开了个后门，而且以后会因为"为什么我这个接口不用登录"困惑很久。

---

# 第五部分 · 面试问答

**Q：登录请求里的 `uuid` 是什么？**
A：是上一次 `/captchaImage` 时后端生成的随机标识（Hutool `IdUtil.simpleUUID()`，32 位无横线十六进制）。作用是把「这张验证码图」和「Redis 里那条答案」关联起来：前端只拿到 `uuid` + 图片，**答案只存服务端** `captcha_codes:<uuid>`，2 分钟过期，校验时读完即删（一次性，防重放）。详见 1.0。

**Q：验证码为什么存 Redis 而不是 session？**
A：前后端分离 + 可能多实例部署，session 绑单实例就失效。这也是 **RuoYi（`HttpSession` 版）→ RuoYi-Vue-Plus（Redis 版）的演进点**，可以拿来回答「为什么传统 session 在分布式下不行」。

**Q（加分）：`/captchaImage` 会被刷挂吗？**
A：要分两层看。**Redis 层**：每个 key 实测 169 字节，而 2 分钟 TTL 把稳态堆量锁死在「速率 × 120 秒」，单机刷不到几十 MB。**接口层**：它是匿名接口（`security.excludes` 白名单）且**没加限流**，但本身 CPU 密集（渲染图片 + base64），服务端吞吐先饱和——所以先倒的是 CPU/带宽，不是 Redis。**真正的风险在配置**：本项目 Redis 实测 `maxmemory=0` + `noeviction`，没有兜底，真堆起来是虚拟机 OOM 杀掉 Redis → 所有 `token:` 丢失 → 全站掉线。加固只需给接口加一行 `@RateLimiter(limitType = LimitType.IP)`——框架自带注解和切面，只是全项目 0 处使用。

**Q：token 是怎么产生的？**
A：登录校验通过后，`LoginUser.getLoginId()` 把「租户号 + 用户类型 + 用户 ID」用 `:` 拼成登录标识，交给 `StpUtil.login()`。项目通过 `StpLogicJwtForStyle` 把 token 换成 JWT，payload 里带 `loginId`，用 `jwt-secret-key` 签名。

**Q：token 存在哪？为什么要存 Redis？**
A：Sa-Token 默认存内存，本项目通过实现 `SaTokenDao` 接口（`PlusSaTokenDao`）换成 Redis。因为存内存有三个问题：重启即失效、多实例部署时 A 实例发的 token 到 B 实例不认、无法做强制下线。存 Redis 后这三个都解决了。

**Q：一次登录在 Redis 里写了几个 key？分别干什么？**
A：5 个（key 清单见 1.5）。核心是 `token:login:token:<JWT>`（token→loginId 映射）和 `token:login:token-session:<JWT>`（用户身份+权限快照）。另有账号级会话、活跃时间戳、在线用户展示记录。

**Q：`@SaCheckPermission` 是怎么生效的？**
A：本项目是 **Spring MVC 拦截器** `SaAnnotationInterceptor` 在 `preHandle` 里用反射读方法上的注解，再交给 `SaManager.getSaTokenAction().validateAnnotation()` 分派。**不是 AOP 切面**——`sa-token-spring-aop` 这个 artifact 根本没被打进 jar（可用 `unzip -l` 查 `BOOT-INF/lib` 验证）。

**Q：权限码每次请求都查库吗？**
A：不查。**只在登录时查一次**，结果存在 Redis 的 token-session 里。每次请求由 `SaInterfaceImpl` 从会话对象读。推论：改权限后已登录用户不会立即生效，需要重新登录或强制下线。

**Q：Sa-Token 相比 Spring Security 的优势？**
A：Sa-Token 主打**无状态设计 + 可插拔 DAO**。三个 SPI（`SaTokenDao` / `StpInterface` / `SaTokenListener`）都是接口，加个 `@Component` 就能替换实现，业务代码零侵入。Spring Security 功能更全但配置复杂、学习曲线陡。本项目用 Sa-Token 把会话落 Redis，从而支持多实例部署和强制下线。

**Q：超管为什么不用查权限表？**
A：`SysPermissionService.getMenuPermission()` 里有短路——`user.isAdmin()` 为真时直接返回 `"*:*:*"`，压根不查 `sys_menu`。注意这个判定读的是 `sys_user.SYSTEM_ADMIN` / `TENANT_ADMIN` **字段**，不是角色表（D2 §0.3 有详解）。

**Q：ThreadLocal 缓存用户信息有什么风险？**
A：线程池复用会导致上一个请求的身份泄漏给下一个请求。本项目在 `SaTokenConfig` 的拦截器 `afterCompletion` 里调 `LoginHelper.clearCache()` 清理，保证请求结束必清。

---

# 第六部分 · 🎤 一分钟讲法（对着面试官讲）

被问「Sa-Token 的鉴权链路讲一下」，照这段讲，**一分钟内讲完**：

> 「一句话：**登录时把「我是谁 + 我能干什么」整个塞进 Redis，之后每次请求只读这份快照，不再查库**。
>
> 展开说：登录校验通过后，`LoginUser.getLoginId()` 把「租户号 + 用户类型 + 用户 ID」拼成 `00000000:sys_user:1` 交给 `StpUtil.login()`，项目用 `StpLogicJwtForStyle` 把它变成 JWT。会话通过**三个 SPI** 落到 Redis（db5）——一次登录写 5 个 key，最核心的是 `token:login:token-session:<JWT>`，里面装着 `LoginUser`，包含 `menuPermission` 权限码集合。
>
> 请求进来时**两道拦截器**：`SaRouteInterceptor` 管「登录了吗」（认证，失败 401），`SaAnnotationInterceptor` 管「有权限吗」（授权，失败 403）。后者用反射读 `@SaCheckPermission`，再通过 SPI `StpInterface`（项目实现是 `SaInterfaceImpl`）拿到权限码集合来比对——**这个集合是从会话对象里读的，全程不查数据库**。
>
> 代价：权限码只在登录那一刻查一次库，所以**改了角色权限，已登录的人必须重新登录才生效**——这就是企业后台改完权限常提示「请重新登录」的原因。」

如果只给 30 秒，就讲这一句：

> 💡 **一句话说人话**：**Sa-Token 把「你是谁、你有什么权限」在登录时查一次、存进 Redis；之后每个请求只是从 Redis 里把这份快照读出来比对，不再查库。省了性能，代价是改了权限得重新登录。**

**⚠️ 这段里有两个高频送命题，答错很扣分**：

1. 「`@SaCheckPermission` 是 AOP 实现的吧？」→ **不是**。本项目是 Spring MVC 的 `SaAnnotationInterceptor` 在 `preHandle` 里反射读注解；`sa-token-spring-aop` 这个包**根本没打进 jar**（`unzip -l ... | grep sa-token` 可验证，见 2.3）。
2. 「去 Redis 里 `KEYS "satoken*"` 查会话」→ **前缀是 `token:` 不是 `satoken:`**，因为 `sa-token.token-name` 被设成了 `token`，Sa-Token 的 key 前缀用的就是这个值（见 1.5）。照抄网上文章会**一条都查不到**。

---

# 附 A · 本篇涉及的关键文件

| 文件 | 职责 |
|---|---|
| `ruoyi-admin/.../controller/system/SysLoginController.java` | 登录/登出接口入口 |
| `ruoyi-base/ruoyi-system/.../service/SysLoginService.java` | 登录主流程（验证码、密码、组装 LoginUser） |
| `ruoyi-base/ruoyi-system/.../service/SysPermissionService.java` | 查权限码/角色，含超管短路 |
| `ruoyi-base/ruoyi-common/.../core/domain/model/LoginUser.java` | 会话里装的用户对象，`getLoginId()` 在这 |
| `ruoyi-base/ruoyi-common/.../helper/LoginHelper.java` | 业务代码拿当前用户的唯一入口，两级缓存 |
| `ruoyi-base/ruoyi-common/.../constant/Constants.java` | `ONLINE_TOKEN_KEY = "online_tokens:"`（:96） |
| `ruoyi-base/ruoyi-framework/.../config/SaTokenConfig.java` | 两道拦截器注册 + JWT 模式 |
| `ruoyi-base/ruoyi-framework/.../config/RedisConfig.java` | Redisson + JsonJacksonCodec |
| `ruoyi-base/ruoyi-framework/.../config/JacksonConfig.java` | Long 超范围序列化成 String（防 JS 精度丢失） |
| `ruoyi-base/ruoyi-framework/.../satoken/dao/PlusSaTokenDao.java` | **SPI 1**：会话落 Redis |
| `ruoyi-base/ruoyi-framework/.../satoken/service/SaInterfaceImpl.java` | **SPI 2**：权限码/角色提供者 |
| `ruoyi-base/ruoyi-framework/.../listener/UserActionListener.java` | **SPI 3**：登录/登出回调，写在线用户 |
| `ruoyi-common/.../utils/redis/RedisUtils.java` | 底层 Redisson 封装（:39 注入 CLIENT） |
| `ruoyi-admin/src/main/resources/application.yml` | `sa-token:`（:101）、`security.excludes:`（:158） |

# 附 B · Redis key 速查（db 5）

| key 模式 | value | 生命周期 |
|---|---|---|
| `token:login:token:<JWT>` | loginId 字符串 | `timeout`（186400s） |
| `token:login:token-session:<JWT>` | `SaSession`（含 `loginUser`） | `timeout` |
| `token:login:last-activity:<JWT>` | 毫秒时间戳 | 随活跃续期；**登出不清** |
| `token:login:session:<loginId>` | `SaSession`（含 `tokenSignList`） | `timeout` |
| `online_tokens:<JWT>` | `UserOnlineDTO` | `timeout` |
| `captcha_codes:<uuid>` | 验证码答案 | **TTL 2 分钟**，校验后立即删（一次性）；key 46 字符，实测 ~169 字节/条 |
| `login_error:<tenantId>:<username>` | 错误次数 | 锁定时间（分钟级） |

# 附 C · 两处对计划文档的订正

计划文档 `docs/superpowers/plans/2026-09-17-m0-skeleton.md` 的 Task 15 里有两处写错了，**以本文为准**：

1. **Step 2 的 Redis 命令**：原文写 `KEYS "satoken*"`，实际前缀是 **`token:`**。原因见 1.5 的说明（`token-name` 被设成了 `token`）。
2. **Step 5 的第 3 问提示**：原文提示"找 Sa-Token 的 AOP 切面"，实际本项目**没有 AOP 切面**，是 `SaAnnotationInterceptor` 拦截器。验证方法与原因见 2.3。

---

## ⚡ 踩坑速查（现象 → 根因 → 怎么确认）

| 现象 / 报错 | 根因 | 一键确认 |
|---|---|---|
| `401 认证失败，无法访问系统资源` | 没带 token / token 过期 / **请求头名写错** | 看请求头是不是 `token: Bearer eyJ...`（**不是** `Authorization`，见铁律 1 · D2） |
| `403 没有访问权限，请联系管理员授权` | 第 2 道拦截器拦的：权限码不在集合里 | 换成**非超管账号**再试；超管的集合是 `["*:*:*"]`，**什么都不查、什么都通过** |
| Redis 里 `KEYS "satoken*"` 一条都查不到 | 前缀被 `sa-token.token-name: token` 改成了 `token:` | 改成 `KEYS "token*"`（见 1.5） |
| 后台改了角色权限，用户那边没变化 | 权限码是**登录快照**，存在 token-session 里 | 让用户**重新登录**（或后台强制踢下线）再看（见 2.5） |
| **所有** `@SaCheckPermission` 都 403 | `StpInterface` 的实现没被容器扫到 → 权限列表恒为空 | 确认 `SaInterfaceImpl` 上有 `@Component`、且返回的 `loginUser.getMenuPermission()` 非空（`SaInterfaceImpl.java:29`） |
| 重启后端后所有人被登出、多实例互相不认 | `SaTokenDao` 没换成 Redis，会话还存内存 | 确认容器里有 `PlusSaTokenDao`（`@Component`），且连的是同一台 Redis（db 5） |
| 后台「在线用户」页面是空的 | `SaTokenListener` 的实现没生效，`online_tokens:*` 没写进去 | `redis-cli -n 5 KEYS "online_tokens:*"`；检查 `UserActionListener` 的 `@Component`（`Constants.java:96`） |
| 脚本里遍历 Redis key 拼出来的 key 永远查不到 | Windows 版 `redis-cli` 输出带 `\r` | 管道接 `tr -d '\r'` 去掉回车（见 1.5 的坑 2） |
| Logout 之后 `last-activity` key 还在 | 登出只清账号 session，不清活跃时间戳 | 正常现象，不影响使用（见 1.5 末尾） |

---

## 📎 相关文档

| 文档 | 讲什么 |
|---|---|
| 🏁 [回到总入口](../README.md) | zxcm-learn 通关手册（唯一入口） |
| [D0 · 底座表地图](D0-底座表地图.md) | 26 张框架底座表的说明书 |
| [D2 · 前后端调用链路](D2-前后端调用链路.md) | 权限三层 + 前端工程概念 + 请求全链路 |
| [D3 · 数据权限链路](D3-数据权限链路.md) | `@DataPermission` → 拦截器改 SQL → SpEL 拼条件 |
| [E1 · 前端工程结构](../前端/E1-zxcm-ui工程结构.md) | 前端目录、菜单→页面、页面→接口 |
| [前端代码走读](../前端/E3-前端代码走读.md) | 跟着真实代码走一遍（登录 / 菜单 / 列表 / 新增） |
