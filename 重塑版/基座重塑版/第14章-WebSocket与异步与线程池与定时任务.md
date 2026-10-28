# 第 14 章 · WebSocket · 异步 · 线程池 · 定时任务

> **前面 13 章讲的都是"请求进来 → 处理 → 返回"这种"前端问、后端答"的模式。本章讲四种"不按这个模式走"的东西**：
>
> ① **WebSocket**——**服务端主动推消息给前端**（不问了，直接告诉你）；② **`@Async` 异步**——**把"不该让用户等的事"丢到后台**；③ **线程池**——**异步和定时任务的"发动机"**；④ **XXL-Job 定时任务**——**让系统在"没人操作的时候"自己干活**。
>
> **本章第一次讲透**：① **WebSocket 的完整实现**（`@ServerEndpoint` + 会话管理 + **一个很聪明的"3 秒防抖"**）；② **为什么 `@Async` 必须配线程池**（不配就用 Spring 的默认线程池，**每次调用新建一个线程，高并发直接炸**）；③ **线程池四个参数怎么定、拒绝策略为什么选 `CallerRunsPolicy`**；④ **为什么本项目用 XXL-Job 而不是 Quartz**；⑤ 几个坦白点：**`thread-pool.enabled: false` 导致一半配置项没生效**、**WebSocket 的线程池不受 Spring 管**。

---

## 一、这部分是做什么的？

### 1.1 四块内容，各解决什么

| 块 | 解决什么 | 关键类 | 状态 |
|---|---|---|---|
| **WebSocket** | **服务端主动推送**（消息、待办提醒） | `WebSocketServer`、`WebSocketSessionManager` | **✔ 在用**（通知模块调用） |
| **异步 `@Async`** | **不阻塞主流程**（记日志、发短信、发邮件） | `AsyncConfig`、`@Async` 方法 | **✔ 在用**（日志、短信、邮件） |
| **线程池** | **给异步/定时任务提供线程** | `ThreadPoolConfig`、`Threads`、`ThreadPoolRejectedPolicy` | **✔ 部分在用**（见坦白点） |
| **定时任务 XXL-Job** | **定时/周期执行**（档案归还、错误消息、数据同步） | `XxlJobConfig` + `@XxlJob` | **✔ 在用**（3 个 handler） |

### 1.1.1 零基础先补概念

| 概念 | 大白话 | 本项目里的真实例子 |
|---|---|---|
| **长连接（WebSocket）** | 普通 HTTP 是"问一次答一次"；WebSocket 是"打通一条电话线"，双方随时能说话，服务端可以主动推 | 端点 `/websocket/message/{token}`（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketServer.java:35`） |
| **异步（`@Async`）** | 把"用户不用等的事"（记日志、发短信）交给另一条线程去做，主线程马上返回 | 方法上加 `@Async`，Spring 就把它交给线程池后台执行；总开关在 `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/AsyncConfig.java:35` |
| **线程池** | 一组"可复用的干活线程"。任务来了排队，线程空了就接活；四个关键参数：核心线程数、最大线程数、队列容量、拒绝策略 | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/ThreadPoolConfig.java:58-69` |
| **定时任务 / 调度中心** | 到点自动执行代码。调度中心是"闹钟总管"，业务应用是"执行者" | XXL-Job：`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/XxlJobConfig.java:65` 注册执行器，`@XxlJob("任务名")` 定义任务 |

**一句话串起来**：WebSocket 负责"推"，`@Async` 负责"不等"，线程池负责"到底谁来干"，定时任务负责"没人点时也会干"。

### 1.2 为什么需要"服务端主动推"？

**"前端问、后端答"（HTTP 请求-响应）有个天然短板：后端不能主动说话。**

**场景**：张三审批了李四提交的合同，**系统要"实时告诉李四：你的合同审批通过了"**。**用 HTTP 怎么办？**

- **方案 A（轮询）**：**前端每秒问一次"我有新消息吗？"**——**99.9% 的请求都是白问**，浪费带宽和服务器资源，且"最多延迟 1 秒"；
- **方案 B（长轮询）**：**前端发一个请求，后端"hold 住"不返回，直到有新消息或超时**——**比轮询好，但连接占用高**；
- **方案 C（WebSocket，本项目选的）**：**前端连一次，之后双向长连接，后端随时能推**——**实时、低开销**。

**本项目的场景**：**审批提醒、待办通知、系统公告**——**"实时性有要求、但消息量不大"**，**WebSocket 是正解。**

### 1.3 为什么要"异步"？

**有些事"用户根本不需要等"**：

- **记操作日志**（第 8 章）——**用户点了"新增"，日志记没记成功关他什么事？**
- **发短信/邮件**——**发出去要 1-3 秒，用户干等着看转圈？**
- **写登录日志**（第 2/8 章）——**登录成功就该立刻返回 token，日志慢慢写**。

**异步的核心价值**：**把"耗时但不影响返回结果"的操作，从"主线程"挪到"后台线程"**——**主流程快返回（用户体感好），后台慢慢做。**

**代价**（第 11 章讲过）：**异步线程里的异常"进不了全局异常处理器"**——**得自己兜住。**

### 1.4 为什么要"定时任务"？

**有些活"没人操作也要干"**：

- **档案到期归还**（`ContractJobService.java:96` `@XxlJob("archiveReturnHandler")`）——**每天早上检查"有没有档案该归还了"**；
- **错误消息处理**（`SysErrorServiceJob.java:43` `@XxlJob("errorMsgScheduleHandler")`）；
- **钉钉数据同步**（`SyncJobService.java:34` `@XxlJob("syncDing")`）——**每隔几分钟从钉钉拉最新数据**。

**定时任务 = "系统的闹钟"**——**到点自动执行某段代码。**

### 1.5 四块的关系：一条链

```
定时任务（XXL-Job 触发）
   └─ 执行时可能用 @Async 异步跑 → 消耗线程池的线程
                                        ↓
WebSocket 推送（服务端主动通知） ← 业务动作完成后触发
```

**线程池是"发动机"，异步和定时任务是"两种负载"，WebSocket 是"输出口"。** **本章把这条链讲通。**

---

## 二、站在架构师 / 开发经理角度：为什么这么设计？

### 2.1 WebSocket 为什么比"轮询"好？——用数据说话

| 方案 | 实时性 | 服务器开销 | 实现复杂度 | 本项目 |
|---|---|---|---|---|
| **轮询（每 5 秒）** | 差（最多延迟 5 秒） | **高（大量空请求）** | **低** | ✗ |
| **长轮询** | 好 | 中（连接占用） | 中 | ✗ |
| **SSE（Server-Sent Events）** | 好 | 低 | 中 | ✗ |
| **WebSocket（本项目）** | **好（真正实时）** | **低（一条长连接）** | 中 | **✔** |

**算一笔账**：**1000 个在线用户，轮询 5 秒一次 = 每秒 200 个请求**——**哪怕 99% 都返回"没有新消息"，这 200 QPS 也是纯浪费。** **WebSocket 下，这 1000 个用户只有 1000 条连接（建立时握手一次），之后推送才消耗资源。**

**WebSocket 的代价**：**长连接占用服务器连接数、要处理"断线重连"、要维护"谁连着"的会话表**（这就是 `WebSocketSessionManager` 存在的理由）。

### 2.2 为什么不用 SSE？——一句话对比

**SSE（Server-Sent Events）** 也是"服务端推"，**但它只有"单向"（服务端 → 客户端）**，而且：
- **基于 HTTP，会受"连接数限制"影响**（浏览器对同域名 HTTP/1.1 连接数有限制）；
- **只能发文本**。

**WebSocket 双向、二进制/文本都行、不受 HTTP 连接限制**。**本项目的推送需求虽然只需单向，但 WebSocket 更通用、生态更成熟**（`javax.websocket` 标准 API，Tomcat 原生支持）。

### 2.3 为什么 `@Async` 必须"显式配线程池"？

**这是个高频踩坑点。** Spring 的 `@Async` **如果不指定 Executor，会用默认的 `SimpleAsyncTaskExecutor`**——**而这个默认实现在**每次**调用时都"新建一个线程"，且不复用！**

```
@Async 用默认 Executor
   → 1000 次调用 = 新建 1000 个线程
   → 每次调用都可能 OOM（线程栈占内存）
```

**Spring 官方明确警告过这一点**，所以**正确的做法是：配置一个真正的线程池，让 `@Async` 用它**。本项目就是这么干的（`AsyncConfig.getAsyncExecutor() :48` 返回 `scheduledExecutorService`）。

**所以："用了 `@Async` 但没配线程池" = 埋了个定时炸弹。** **这是面试常考、也常被面试者忽略的点。**

### 2.4 线程池的四个参数怎么定？

**`ThreadPoolExecutor` 的核心参数**（本项目 `thread-pool` 配置，`application.yml:322-338`）：

```yaml
thread-pool:
  enabled: false            # :324  是否启用（独立线程池 Bean）
  corePoolSize: 8           # :326  ★核心线程数
  maxPoolSize: 16           # :328  ★最大线程数
  queueCapacity: 128        # :330  ★队列容量
  keepAliveSeconds: 300     # :332  空闲回收时间
  rejectedExecutionHandler: CALLER_RUNS_POLICY   # :338  ★拒绝策略
```

**执行逻辑（背下来）**：

```
任务来了：
  ├─ 核心线程数没满（< corePoolSize=8）→ 新建核心线程执行
  ├─ 核心满了 → 塞进队列（queueCapacity=128）
  ├─ 队列也满了 → 新建"临时线程"（最多 maxPoolSize=16）
  ├─ 临时线程也满了 → 触发"拒绝策略"
  └─ 空闲：非核心线程空闲 keepAliveSeconds=300 秒后被回收
```

**"核心小、最大稍大、队列较大"是 IO 密集型任务的典型配法**——**本项目异步任务（记日志、发消息）都是 IO 密集型**（等数据库、等网络），**所以不需要开很多线程**（`core=8` 够用），**但要给队列留缓冲**（`128`）。

**⚠️ 关键认知**：**队列容量和最大线程数是"此消彼长"的**——**如果队列设成无界（Integer.MAX_VALUE），那 `maxPoolSize` 永远不会生效**（任务全塞队列了，不会新建临时线程）——**这正是 `ScheduledThreadPoolExecutor` 的情况**（§4.6 会讲）。

### 2.5 拒绝策略为什么选 `CallerRunsPolicy`？

**当"队列满 + 线程满"，怎么办？** 四种策略（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/enums/ThreadPoolRejectedPolicy.java`）：

| 策略 | 行为 | 适用 |
|---|---|---|
| `AbortPolicy`（JDK 默认） | **抛 `RejectedExecutionException`** | 想"快速失败"、不能丢任务 |
| `DiscardPolicy` | **静默丢弃** | 丢了不心疼（如心跳） |
| `DiscardOldestPolicy` | **丢队列里最老的，塞新的** | 只要最新（如实时行情） |
| **`CallerRunsPolicy`（本项目）** | **让"提交任务的线程"自己执行** | **想"不丢任务 + 自动限流"** |

**为什么 `CallerRunsPolicy` 最适合本项目？**

**它的精髓是"反压（back pressure）"**：**队列满了，就让"调用方线程"自己跑这个任务，而不是新开线程或丢弃。** **效果**：

1. **任务不丢**——**保证"记日志""发消息"这些事一定被执行**；
2. **自动限流**——**调用方（比如用户的请求线程）被"占用"去跑任务了，它就没空继续生产新任务**——**生产速度自然慢下来**，**形成负反馈**；
3. **不会 OOM**——**因为不新建线程、也不无界堆积**。

**代价**：**调用方被"拖慢"**（本来异步是想快的，结果队列一满又变同步了）——**但这是"宁可慢、不可丢、不可崩"的正确取舍。** **本项目**（日志、消息推送）**对"不丢"的要求高于"快"，所以这个策略是合适的。**

---

## 三、需要哪些技术？

| 技术点 | 是什么 | 在本项目哪里用到 | 关键类/方法 |
|---|---|---|---|
| **JSR-356 WebSocket（`javax.websocket`）** | Java 标准的 WebSocket API | 服务端推送 | `WebSocketServer.java:35` |
| **`@ServerEndpoint`** | 声明 WebSocket 端点 | 定义连接地址 | `WebSocketServer.java:35` |
| **`ServerEndpointExporter`** | 让 Spring 扫描 `@ServerEndpoint` | WebSocket 注册 | `WebSocketConfig.java:28` |
| **`Session`** | 一条 WebSocket 连接的抽象 | 发消息 | `WebSocketServer.java:38` |
| **`ConcurrentHashMap` + `CopyOnWriteArrayList`** | 并发安全的"用户→会话"表 | 在线会话管理 | `WebSocketSessionManager.java:35` |
| **`ScheduledExecutorService`** | 延时/周期执行任务 | 推送"防抖" | `WebSocketSessionManager.java:39` |
| **`@Async`** | 声明异步方法 | 记日志、发消息 | `AsyncConfig.java:35`（`@EnableAsync`） |
| **`AsyncConfigurerSupport`** | 定制 `@Async` 的执行器与异常处理 | 指定线程池 | `AsyncConfig.java:38` |
| **`ThreadPoolTaskExecutor`** | Spring 封装的线程池 | 独立线程池 Bean | `ThreadPoolConfig.java:44` |
| **`ScheduledThreadPoolExecutor`** | 支持定时/周期的线程池 | `@Async` 的实际执行器 | `ThreadPoolConfig.java:60` |
| **`RejectedExecutionHandler`** | 拒绝策略 | 队列满时的行为 | `ThreadPoolRejectedPolicy.java` |
| **`BasicThreadFactory`** | 给线程起有意义的名字 | 便于排查 | `ThreadPoolConfig.java:61` |
| **反射创建实例（`ReflectUtils.newInstance`）** | 按配置类名造对象 | 拒绝策略可配置 | `ThreadPoolConfig.java:50` |
| **`@ConditionalOnProperty`** | 条件装配 | 线程池/XXL-Job 开关 | `ThreadPoolConfig.java:43`、`XxlJobConfig.java:35` |
| **XXL-Job** | 分布式调度中心 | 定时任务 | `XxlJobConfig.java`、`@XxlJob` |

---

## 四、功能具体怎么实现（带着你读懂）

### 4.1 WebSocket 注册：`WebSocketConfig`

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketConfig.java`）

```java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {          // :28
        return new ServerEndpointExporter();
    }
}
```

**这三行的作用**：**`@ServerEndpoint` 注解的类，如果只靠标准 JSR-356，容器（Tomcat）会把它当普通类——Spring 的 Bean 注入、AOP 都不生效。** **`ServerEndpointExporter` 是 Spring 提供的"桥"**——**它会在启动时扫描所有 `@ServerEndpoint` 的 Bean，把它们注册到 WebSocket 容器里，同时保证这些 Bean 仍是"Spring 管的"**（所以 `WebSocketServer` 能 `@Component` + 注入依赖）。

**⚠️ 注意**：**`ServerEndpointExporter` 只在"内嵌容器"（内嵌 Tomcat）下需要**。**如果打成 war 部署到外部 Tomcat，这个 Bean 会冲突**（外部 Tomcat 自己管 WebSocket）——**这是"内嵌 vs 外置容器"的一个经典差异点**（本项目用内嵌 jar，所以没问题）。

### 4.2 端点定义：`WebSocketServer`

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketServer.java`）

```java
@Component                                                          // :34  ★Spring Bean
@ServerEndpoint("/websocket/message/{token}")                        // :35  ★★ 端点地址，带 token 路径参数
public class WebSocketServer {

    @OnOpen                                                         // :37  ★连接建立时
    public void onOpen(Session session, @PathParam("token") String token) {
        WebSocketSessionManager.addUserSession(token, session);      // :39  登记"这个 token 的这条会话"
    }

    @OnClose                                                        // :43  ★连接关闭时
    public void onClose(Session session, @PathParam("token") String token) {
        WebSocketSessionManager.removeUserSession(token, session);   // :45  注销
    }

    @OnError                                                        // :49
    public void onError(Session session, @PathParam("token") String token, Throwable throwable) {
        // 本项目留空（只注释掉了 println）
    }

    @OnMessage                                                      // :54
    public void onMessage(String message, Session session, @PathParam("token") String token) {
        // 收到的消息一般是 ping 消息，什么都不做
    }
}
```

**逐点读懂**：

1. **`@ServerEndpoint("/websocket/message/{token}")`（`:35`）**：**前端连的地址是 `ws://ip:8080/websocket/message/{你的token}`**。**把 token 放在 URL 路径里，是为了"连接时就知道是谁"**——**因为 WebSocket 握手时不能像普通 HTTP 那样方便地带自定义 Header**（浏览器原生 `WebSocket` API 不支持自定义 header）。**"token 走 URL 参数"是 WebSocket 鉴权的常见做法。**
2. **`@OnOpen` / `@OnClose` / `@OnError` / `@OnMessage`**：**四个生命周期回调**——**建连、断连、出错、收到消息**。**本项目"只关心连接和断开"**（因为业务是"服务端推、客户端收"，客户端几乎不发有意义的消息；`@OnMessage` 收到的只是**心跳 ping**）。
3. **`@OnMessage` 里注释说"一般是 ping 消息"**：**这是"保活"机制**——**前端定时发个 ping，确保连接不被中间网络设备（Nginx、防火墙）判定为"空闲连接"而切断。**

**⚠️ 一个重要的安全点**：**`/websocket/message/**` 被放进了免登录白名单**（`application.yml:191`）——**因为"WebSocket 握手"在 Sa-Token 拦截器看来是个普通请求，如果不白名单就会 401**。**真正的鉴权靠"token 参数"**（token 本身就是登录凭证）。**可以说"鉴权靠 token 自证"，但这个设计"不优雅"**——**更规范的做法是在 `onOpen` 里主动校验 token 有效性（`StpUtil.getLoginIdByToken(token)`），无效就 `session.close()`**（本项目没做这一步，属于"依赖前端传对 token"）。

### 4.3 会话管理：`WebSocketSessionManager`（本章最值得读的代码）

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketSessionManager.java`）

```java
public class WebSocketSessionManager {
    // 连接的 websocket：token → 该 token 的所有会话
    private static final ConcurrentHashMap<String, CopyOnWriteArrayList<Session>> userSessions
        = new ConcurrentHashMap<>();                                    // :35  ★
    // 待推送消息任务：token → 待执行的延时任务
    private static final ConcurrentHashMap<String, ScheduledFuture<?>> userTasks
        = new ConcurrentHashMap<>();                                    // :37  ★
    // 推送线程池
    private static final ScheduledExecutorService scheduler
        = Executors.newScheduledThreadPool(20);                         // :39  ★★ 自己新建的线程池

    public static void addUserSession(String token, Session session) {   // :41
        userSessions.computeIfAbsent(token, k -> new CopyOnWriteArrayList<>()).add(session);
    }

    public static void removeUserSession(String token, Session session) {  // :45
        CopyOnWriteArrayList<Session> sessions = userSessions.get(token);
        if (sessions != null) {
            sessions.remove(session);
            if (sessions.isEmpty()) {
                userSessions.remove(token);                              // ★ 空的就删 key，防内存泄漏
            }
        }
    }

    // 异步推送方法：延迟 3 秒推送，同一个 3 秒内不多次推送
    public static void sendMessageToUser(String token, WebSocketMessage message) {   // :57
        // 移除之前的推送任务（如果存在）
        ScheduledFuture<?> previousTask = userTasks.remove(token);        // :60
        if (previousTask != null && !previousTask.isDone()) {
            previousTask.cancel(true);                                    // :62  ★★ 取消上一次待推的
        }

        // 创建一个新的推送任务
        ScheduledFuture<?> task = scheduler.schedule(() -> {              // :66
            CopyOnWriteArrayList<Session> sessions = userSessions.get(token);
            if (sessions != null) {
                for (Session session : sessions) {
                    try {
                        session.getBasicRemote().sendText(JsonUtils.toJsonString(message));   // :71  ★发出去
                    } catch (IOException e) {
                        log.error("WebSocket 消息发送失败", e);
                    }
                }
            }
        }, 3, TimeUnit.SECONDS);                                          // :77  ★★ 3 秒后执行

        userTasks.put(token, task);                                       // :80
    }
}
```

**逐行读 `sendMessageToUser`**（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketSessionManager.java:57-81`）：

| 行号 | 这一行在干什么 | 为什么这么写 |
|---|---|---|
| `:57` | 方法签名：`token` + 要推的消息 | 以 token 为键，说明"一条连接属于一个登录 token" |
| `:60` | `userTasks.remove(token)`：把该 token 的旧任务取出来并从表里删掉 | 一个 token 同时只允许有一个"待推任务"，这是防抖的存储基础 |
| `:61-63` | 旧任务还没执行完，就 `cancel(true)` 取消 | **取消的是"3 秒后要执行的那次推送"**，不是已经发出去的消息 |
| `:66` | `scheduler.schedule(() -> {...}, 3, SECONDS)`：安排 3 秒后执行 | 先不急着发，给"连续消息"留出合并窗口 |
| `:67-76` | 3 秒后按 token 取会话列表，逐个发送 | 同一 token 可能有多条连接（多标签页），要全部推 |
| `:71` | `session.getBasicRemote().sendText(JSON)` | WebSocket 只能发文本/二进制，所以对象先转 JSON（见本节下方第 5 点） |
| `:77` | 延迟 3 秒 | 防抖窗口；**期间每来一条新消息都会把计时重置** |
| `:80` | `userTasks.put(token, task)` | 记住新任务，供下一条消息来时取消 |

**一个并发细节**：`remove` 和 `put` 之间不是原子的。如果两个请求几乎同时进来，理论上可能都拿到 `null`、各自排一个任务。**防抖能挡住"大部分连续消息"，但不是严格串行**——对通知类消息够用，对"每条必须且只能一次"的消息不够。

**★ 核心亮点：`sendMessageToUser` 的"3 秒防抖"**

**这段代码解决了一个很实际的体验问题**：**如果业务在 3 秒内给同一个用户推了 5 条消息（比如"批量导入 5 条合同，每条都触发一次提醒"），用户会被弹窗轰炸。**

**防抖逻辑**（`:57-80`）：

```
第 1 条消息来 → 没有待执行任务 → 安排"3 秒后推送"，记下任务
第 2 条消息来（3 秒内）→ 有任务 → 取消它（:62）→ 重新安排"3 秒后推送"
第 3、4、5 条同理 → 每次都是"取消旧的、重排新的"
3 秒后（期间没新消息了）→ 任务执行 → 只推最后一次的消息
```

**效果**：**"连续多条的推送"被合并成"最后一次"**——**用户只被提醒一次，而不是 5 次。** **这是"防抖（debounce）"思想**（前端搜索框"输入停止后才请求"也是这个）。

**代价（要清楚）**：**中间那几条消息的内容丢了**（只保留最后一条）——**所以这个设计适合"通知类消息"**（知道"有新消息"就行），**不适合"每条都必须送达"的消息**（那种不能防抖）。

**其他几个设计点**：

1. **`userSessions` 的值是 `CopyOnWriteArrayList<Session>`（`:35`）**——**为什么是 List 不是单个 Session？** 因为**同一 token 可能有多条连接**（比如同一个浏览器开了两个标签页，或者断线重连时旧的还没清理掉）。**用 List 能"一次推给这个 token 的所有连接"。**
2. **`ConcurrentHashMap` + `CopyOnWriteArrayList`（`:35`）**——**都是并发安全的容器**。**WebSocket 的回调在"各自的连接线程"里执行**（多个线程并发 `addUserSession`/`removeUserSession`），**普通 `HashMap`/`ArrayList` 会出并发问题。** **`CopyOnWriteArrayList` 适合"读多写少"**（推送时读、连接变化时写），**正是这里的场景。**
3. **`removeUserSession` 里"空了就删 key"（`:50`）**——**防止 `userSessions` 里堆积大量"空 List"的内存泄漏。**
4. **`session.getBasicRemote().sendText(...)`（`:71`）**——**"Basic（基础）" vs "Async（异步）"的两个 Remote**：**`getBasicRemote()` 是阻塞发送**（在当前线程发完才返回）；**`getAsyncRemote()` 是非阻塞**（丢进队列立刻返回）。**本项目在"自己的调度线程"里用 `getBasicRemote()`**——**因为反正是后台线程，阻塞也无所谓，且能捕获 `IOException`。**
5. **用 `JsonUtils.toJsonString(message)`（`:71`）**——**WebSocket 只能发"文本/二进制"，对象要先转 JSON。**

### 4.4 消息体：`WebSocketMessage`

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketMessage.java`）

```java
@Data
public class WebSocketMessage {                                   // :19
    String type;      // :20 消息分类（用 templateCode，前端据此决定怎么展示）
    String content;   // :21 内容
    String title;     // :22 标题
}
```

**只有三个字段，非常"薄"**——**`type` 让前端能"分类处理"**（比如 `type=approval` 显示审批弹窗、`type=announcement` 显示公告条）。**这是"前端路由消息"的常见设计。**

**调用方**（`zxcm-notify/src/main/java/com/ruoyi/notify/handle/impl/SystemSendMsgHandle.java:59`）：

```java
//推送websocket消息
WebSocketMessage message = new WebSocketMessage();
message.setTitle(es_title);
message.setContent(es_content);
message.setType(templateCode);                                        // ★用模板编码当 type
WebSocketSessionManager.sendMessageToUser(es_receiver, message);       // ★接收人是 es_receiver（token）
```

**注意**：**`sendMessageToUser(es_receiver, ...)` 的第一个参数是 token**——**说明推送方"必须知道接收人的 token"**（§4.12 会讲这个设计的局限）。

### 4.5 异步：`AsyncConfig`（回顾第 11 章）

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/AsyncConfig.java`）

```java
@EnableAsync                                                          // :35  ★开启 @Async
@Configuration
public class AsyncConfig extends AsyncConfigurerSupport {             // :38

    @Autowired
    @Qualifier("scheduledExecutorService")
    private ScheduledExecutorService scheduledExecutorService;

    @Override
    public Executor getAsyncExecutor() {                              // :48  ★★ @Async 用哪个线程池
        return scheduledExecutorService;                              //      答案：上面那个
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {   // :56
        return (throwable, method, objects) -> {
            log.error("异步任务执行异常", throwable);                    // ★只能记日志（第 11 章的坑）
            ...
            throw new ServiceException(sb.toString());                // :65  ★这句其实"没有出口"
        };
    }
}
```

**这里的"三件事"**：

1. **`@EnableAsync`（`:35`）**：**开启 `@Async` 支持**（不写这句，`@Async` 完全不生效）；
2. **`getAsyncExecutor()`（`:48`）**：**指定 `@Async` 用哪个线程池**——**这是 §2.3 讲的"必须显式配线程池"的落点**；
3. **`getAsyncUncaughtExceptionHandler()`（`:56`）**：**异步异常的"兜底"**——**只能记日志**（第 11 章已讲过它"进不了全局处理器"）。

**`@Async` 的实际使用**（全项目）：

| 位置 | 干什么 |
|---|---|
| `OperLogService.java:26`（接口方法上） | 记操作日志 |
| `SysOperLogServiceImpl.java:56`（实现方法上） | 记操作日志 |
| `SysLogininforServiceImpl.java:61`（实现方法上） | 记登录日志 |
| `SmsSendMsgHandle.java:55` | 发短信 |
| `AsyncEmailService.java:32/51` | 发邮件 |

**一个小观察**：**`recordOper` 的 `@Async` 同时标在"接口方法"（`OperLogService.java:26`）和"实现方法"（`SysOperLogServiceImpl.java:56`）上**——**这是"双保险"写法**（Spring 解析时会看最具体的方法，标在实现上才真正生效；标在接口上是"意图声明"）。**标两处不算错，但属于"冗余"。**

### 4.6 线程池：`ThreadPoolConfig`（两个 Bean，一个"活"一个"死"）

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/ThreadPoolConfig.java`）

**Bean 一：`threadPoolTaskExecutor`（有开关，本项目"没启用"）**

```java
@Bean(name = "threadPoolTaskExecutor")                                // :42
@ConditionalOnProperty(prefix = "thread-pool", name = "enabled", havingValue = "true")   // :43  ★★ 有开关
public ThreadPoolTaskExecutor threadPoolTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setMaxPoolSize(threadPoolProperties.getMaxPoolSize());           // :46
    executor.setCorePoolSize(threadPoolProperties.getCorePoolSize());         // :47
    executor.setQueueCapacity(threadPoolProperties.getQueueCapacity());       // :48
    executor.setKeepAliveSeconds(threadPoolProperties.getKeepAliveSeconds()); // :49
    RejectedExecutionHandler handler = ReflectUtils.newInstance(
        threadPoolProperties.getRejectedExecutionHandler().getClazz());       // :50  ★反射造拒绝策略
    executor.setRejectedExecutionHandler(handler);                            // :51
    return executor;
}
```

**读懂 `:50` 的"反射造对象"**：**`thread-pool.rejectedExecutionHandler` 配的是枚举名 `CALLER_RUNS_POLICY`**（`application.yml:338`）——**枚举里存的是 `Class`（`ThreadPoolRejectedPolicy.java` 里 `CALLER_RUNS_POLICY("调用方执行", ThreadPoolExecutor.CallerRunsPolicy.class)`）**，**`ReflectUtils.newInstance(clazz)` 按类名 new 出实例**。**这样"拒绝策略"就能"配置驱动"**（改配置换策略，不用改代码）——**是"配置化"的一个小实践。**

**Bean 二：`scheduledExecutorService`（无开关，永远创建）**

```java
@Bean(name = "scheduledExecutorService")                              // :58
protected ScheduledExecutorService scheduledExecutorService() {
    return new ScheduledThreadPoolExecutor(
        threadPoolProperties.getCorePoolSize(),                        // :60  ★核心线程数
        new BasicThreadFactory.Builder()
            .namingPattern("schedule-pool-%d")                          // :61  ★线程名：schedule-pool-0/1/2...
            .daemon(true)                                               //      守护线程
            .build(),
        new ThreadPoolExecutor.CallerRunsPolicy()) {                    // :62  ★拒绝策略硬编码为 CallerRuns
        @Override
        protected void afterExecute(Runnable r, Throwable t) {           // :64  ★★ 任务执行后的钩子
            super.afterExecute(r, t);
            Threads.printException(r, t);                                // :66  ★把异常打出来
        }
    };
}
```

**这个 Bean 有四个值得读懂的点**：

1. **`ScheduledThreadPoolExecutor`（`:60`）**：**它内置了一个"无界的延时队列"（`DelayedWorkQueue`）**——**所以它的"最大线程数"其实是固定的（= corePoolSize）**，**`maxPoolSize`/`queueCapacity` 对它没意义**（这就是 §2.4 讲的"队列无界则 max 失效"）。
2. **`BasicThreadFactory` 起名（`:61`）**：**线程名叫 `schedule-pool-0`、`schedule-pool-1`……**——**这是"排查问题的救命稻草"**：**出问题时 `jstack` 一看线程名，就知道是"调度线程池"的线程**，而不是一堆无意义的 `pool-1-thread-5`。**"给线程起名"是线程池最佳实践，很多人不做。**
3. **`daemon(true)`（`:61`）**：**守护线程**——**JVM 退出时，"守护线程"不会阻止退出**。**如果这里用非守护线程，可能导致"应用关不掉"**（非守护线程还在跑，JVM 等它）。
4. **`afterExecute` 钩子（`:64`）**：**任务执行完（无论成功失败）都会调它**——**这里调 `Threads.printException(r, t)`，把"被吞掉的异常"打出来。** **为什么需要？** **因为 `ScheduledExecutorService` 提交的任务如果抛异常，"提交方"可能拿不到**（`schedule()` 返回的 `ScheduledFuture` 若没人 `get()`，异常就被静默丢弃）——**`afterExecute` 是"最后的兜底，把异常写进日志"。**

**`Threads.printException`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/Threads.java:69`）** 的逻辑很讲究：

```java
public static void printException(Runnable r, Throwable t) {              // :69
    if (t == null && r instanceof Future<?>) {                            // ★t 为 null 但 r 是 Future
        try {
            Future<?> future = (Future<?>) r;
            if (future.isDone()) {
                future.get();                                             // :74  ★★ 主动 get() 触发异常抛出
            }
        } catch (CancellationException ce) {
            t = ce;
        } catch (ExecutionException ee) {
            t = ee.getCause();                                            // :79  取出真实异常
        } ...
    }
    if (t != null) {
        log.error(t.getMessage(), t);                                      // :85  ★打日志
    }
}
```

**读懂 `:74` 的"玄机"**：**`Future.get()` 会"抛出任务执行时的异常"**——**如果任务抛了异常但没人 `get()`，异常就被"困在 Future 里"**。**`printException` 主动 `get()` 一次，就是为了"把困住的异常逼出来、写进日志"。** **这是"异步任务异常静默丢失"的标准解法之一。**

### 4.7 `thread-pool.enabled: false` 意味着什么？（★ 坦白点预警）

**本项目配置**（`application.yml:322-338`）：

```yaml
thread-pool:
  enabled: false            # :324  ★★ 这一行是关键
  corePoolSize: 8
  maxPoolSize: 16
  queueCapacity: 128
  keepAliveSeconds: 300
  rejectedExecutionHandler: CALLER_RUNS_POLICY
```

**`enabled: false` 的连锁反应**：

| 配置项 | 是否生效 | 说明 |
|---|---|---|
| `corePoolSize: 8` | **✔ 生效** | **`scheduledExecutorService` 用它**（`:60`） |
| `maxPoolSize: 16` | **✗ 不生效** | 只给 `threadPoolTaskExecutor` 用，**而它没创建** |
| `queueCapacity: 128` | **✗ 不生效** | 同上 |
| `keepAliveSeconds: 300` | **✗ 不生效** | 同上 |
| `rejectedExecutionHandler` | **✗ 不生效** | `scheduledExecutorService` 硬编码了 `CallerRunsPolicy`（`:62`） |

**结论**：**`thread-pool` 里 5 个配置项，只有 `corePoolSize` 真正起作用**——**`threadPoolTaskExecutor` 这个 Bean 因为 `enabled: false` 根本没被创建。** **而 `@Async` 用的是 `scheduledExecutorService`（`AsyncConfig.java:48`）**，**它只吃 `corePoolSize`。**

**这是个很典型的"配置看起来很多、实际生效的很少"的情况**——**排查性能问题时，如果你以为"异步线程池最多 16 个线程"，那就错了**（实际是 `ScheduledThreadPoolExecutor` 的 8 个核心线程 + 无界队列）。**§4.12 会把这个作为坦白点。**

### 4.8 线程工具类：`Threads`

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/Threads.java`）

```java
public class Threads {
    /** 睡一会儿 */
    public static void sleep(long milliseconds) {                        // :34
        try { Thread.sleep(milliseconds); }
        catch (InterruptedException e) { return; }                       // ★吞掉中断，直接返回
    }

    /** 优雅关闭线程池 */
    public static void shutdownAndAwaitTermination(ExecutorService pool) {   // :49
        if (pool != null && !pool.isShutdown()) {
            pool.shutdown();                                             // ★① 不再收新任务，跑完已有的
            try {
                if (!pool.awaitTermination(120, TimeUnit.SECONDS)) {      // ★② 等 120 秒
                    pool.shutdownNow();                                   // ★③ 还没完就"强制中断"
                    if (!pool.awaitTermination(120, TimeUnit.SECONDS)) {
                        log.info("Pool did not terminate");               // ★④ 还不完就放弃
                    }
                }
            } catch (InterruptedException ie) {
                pool.shutdownNow();
                Thread.currentThread().interrupt();                      // ★⑤ 恢复中断标志
            }
        }
    }

    /** 打印异常（前面讲过） */
    public static void printException(Runnable r, Throwable t) { ... }    // :69
}
```

**`shutdownAndAwaitTermination`（`:49`）是"优雅停机"的标准三步**（连 Oracle 官方文档的示例代码都和它一致）：

1. **`shutdown()`**：**停止接收新任务，但已提交的会跑完**（温柔）；
2. **`awaitTermination(120s)`**：**等最多 120 秒**；
3. **`shutdownNow()`**：**还没完就"发中断信号 + 丢弃队列里的任务"**（强硬）；
4. **第二步还超时就"放弃"并记日志**。

**"温柔 → 等待 → 强硬 → 放弃"是关闭线程池的正确姿势**——**直接 `shutdownNow()` 会丢任务；只 `shutdown()` 不等会泄漏线程。**

### 4.9 定时任务：XXL-Job

**本项目没用 Spring 的 `@Scheduled`，也没用 Quartz，而是接了 XXL-Job。**

**`XxlJobConfig`**（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/XxlJobConfig.java:36`）：

```java
@Configuration
@ConditionalOnProperty(prefix = "xxl.job", name = "enabled", havingValue = "true")   // :35  ★开关
public class XxlJobConfig {

    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {                       // :65
        XxlJobSpringExecutor executor = new XxlJobSpringExecutor();
        executor.setAdminAddresses(adminAddresses);                      // 调度中心地址
        executor.setAppname(appName);                                    // 执行器名（分组依据）
        executor.setPort(port);                                          // 通信端口
        executor.setAccessToken(accessToken);                            // ★鉴权 token
        executor.setLogPath(logPath);                                    // 日志目录
        executor.setLogRetentionDays(logRetentionDays);
        return executor;
    }
}
```

**配置**（`ruoyi-admin/src/main/resources/application-dev.yml:81-103`）：

```yaml
xxl:
  job:
    enabled: true                                                    # :85  ★启用
    admin-addresses: http://localhost:9100/xxl-job-admin             # :87  ★调度中心地址
    access-token: ""                                                 # :89  ★非空时启用通讯鉴权
    executor:
      appname: xxl-job-executor                                      # 执行器名
      port: 9101                                                     # 执行器端口
      ...
```

**任务定义**（业务模块里）：

```java
// zxcm-contract/src/main/java/com/ruoyi/contract/service/ContractJobService.java
@XxlJob("archiveReturnHandler")                                      // :96  ★任务名，调度中心按这个名字触发
public void archiveReturnHandler() { ... }

// zxcm-contract/src/main/java/com/ruoyi/contract/service/SysErrorServiceJob.java:43
@XxlJob("errorMsgScheduleHandler")

// zxcm-third/zxcm-syncData/src/main/java/com/ruoyi/sync/service/SyncJobService.java:34
@XxlJob("syncDing")
```

**XXL-Job 的架构（一句话）**：**分"调度中心"（`xxl-job-admin`，一个独立部署的 Web 应用）和"执行器"（你的业务应用）两部分。** **在调度中心的 Web 界面上，给每个 `@XxlJob("任务名")` 配一个 CRON 表达式，调度中心到点就"通知执行器"去跑。** **执行器（本项目）通过 `XxlJobSpringExecutor` 注册自己。**

**`ConditionalOnProperty`（`:35`）+ 注释里的"默认不启用"**：**XXL-Job 是"可选的分布式调度"**——**不需要分布式调度的部署（比如单机小项目），把 `xxl.job.enabled` 设成 `false` 就不会去连调度中心**，跑得更轻。

**`access-token: ""`（`:89`）**：**配置项自己的注释写着"生产环境务必修改"**——**空 token = 不鉴权，调度中心和执行器之间是明文通信。** **生产环境必须设一个非空 token**（这是 XXL-Job 官方安全建议）。

### 4.10 ★ 为什么用 XXL-Job，不用 `@Scheduled` 或 Quartz？

**这是本章最值得讲透的"选型题"**（§7.3 会给对比表，这里先讲逻辑）：

**`@Scheduled`（Spring 原生）的问题**：

1. **没有界面**——**任务写死在代码里，改 CRON 要重新部署**；
2. **集群会重复执行**——**你有 3 台服务器，每个 `@Scheduled` 都会在 3 台上各跑一次**（没有任何"分布式协调"），**"每天发一次日报"就发了 3 次**。

**Quartz 的问题**：

1. **需要"任务表"（`QRTZ_*` 一堆表）**——**侵入数据库 schema**；
2. **集群模式要配"数据库锁"**——**配置复杂、性能一般**；
3. **没有可视化界面**——**触发记录、重试、手动执行都要自己写**。

**XXL-Job 的好处**：

1. **有独立调度中心 + Web 界面**——**CRON、启停、手动触发、看执行日志、失败重试都在界面上点**；
2. **天然的分布式**——**调度中心"只通知一个执行器实例"去跑**（路由策略可选），**解决了"集群重复执行"**；
3. **执行器零依赖**——**只是在你的应用里注册一个 `XxlJobSpringExecutor` + 写 `@XxlJob` 方法**，不侵入数据库；
4. **"任务与应用"解耦**——**任务调度逻辑在调度中心，业务逻辑在执行器**，**职责清晰**。

**代价**：**要多部署一个 `xxl-job-admin`（一个独立应用 + 它自己的数据库）**——**对"就一两个定时任务"的小项目，是过度设计。** **所以本项目用 `enabled` 开关让它"可选"。**

### 4.11 ★ 坦白点（本章有三个）

**坦白点一：`thread-pool.enabled: false`，一半配置项形同虚设**

**事实**（已在 §4.7 展开）：`application.yml:324` 的 `enabled: false` 导致 **`threadPoolTaskExecutor` Bean 从未创建**，**`maxPoolSize`/`queueCapacity`/`keepAliveSeconds`/`rejectedExecutionHandler` 四个配置项全部不生效**；**唯一生效的是 `corePoolSize: 8`**（被 `scheduledExecutorService` 使用，`ThreadPoolConfig.java:60`）。

**后果**：**配置里写着"最大 16 线程、队列 128"，实际上完全不是这样**——**真实执行器是 `ScheduledThreadPoolExecutor(corePoolSize=8)`，队列无界**。**排查性能/线程问题时极易被这组"存在但不生效"的配置误导。**

**怎么理解/怎么说**：**这是 RuoYi-Vue-Plus 从"可选独立线程池"演进到"统一用调度线程池"留下的痕迹**——**`@Async` 现在统一指向 `scheduledExecutorService`（`AsyncConfig.java:48`），独立的 `threadPoolTaskExecutor` 就没用了，但仍留着开关和配置。** **答辩时可以说**："我们的 `@Async` 实际用的是 `scheduledExecutorService`（8 个核心线程 + 无界队列）；`thread-pool` 里那组配置因为 `enabled=false` 没生效——**如果要精细控制异步线程池，应该把 `@Async` 指向一个真正配置化的 `ThreadPoolTaskExecutor`，并打开这个开关。**"

**坦白点二：WebSocket 的线程池不受 Spring 管理，且是硬编码的**

**事实**（`WebSocketSessionManager.java:39`）：

```java
private static final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(20);
```

**问题**：

1. **`Executors.newScheduledThreadPool(20)` 是"裸线程池"**——**不是 Spring Bean，不受容器管理**：**应用关闭时不会优雅停机**（对比 `Threads.shutdownAndAwaitTermination` 那套），**线程可能"残留"**；
2. **`20` 是硬编码**——**改不了、配不了**，**而且跟项目里其他线程池的命名规范（`schedule-pool-%d`）、拒绝策略（`CallerRunsPolicy`）都不一致**（这个裸池用的是默认拒绝策略 `AbortPolicy`，**队列满了就抛异常丢消息**）；
3. **`@Async` 的 import 在这个类里但没用到**（`WebSocketSessionManager.java:19`）——**残留代码**。

**怎么修**：**把 `scheduler` 换成注入 Spring 的 `scheduledExecutorService`**（或新建一个受管的 Bean），**统一线程命名和拒绝策略。**

**坦白点三：WebSocket 推送以"token"为键，推送方必须知道接收人的 token**

**事实**：`sendMessageToUser(token, message)`（`:57`）和 `addUserSession(token, session)`（`:41`）**都以 token 为 key**——**调用方 `SystemSendMsgHandle.java:59` 传的是 `es_receiver`（接收人的 token）。**

**问题**：

1. **"推送"需要先拿到"接收人的 token"**——**这意味着推送方（业务代码）得知道对端当前的登录 token**，**而 token 是"登录会话"的凭证，业务层通常只知道 `userId`**——**所以要么额外存一张"userId → token"的映射，要么改造成按 userId 推**；
2. **一个人多端登录（第 5 章）会有多个 token**——**按 token 推，只能推到"某一个 token 的连接"**，**除非"用 userId 遍历所有 token 推"**；
3. **token 变化（重新登录）后，旧的 `userSessions` 里那条记录会失效**——**靠 `@OnClose` 清理，但若"连接异常断开且没触发 `@OnClose`"就会残留**。

**怎么修**：**会话表的 key 改成 `userId`**（`onOpen` 时用 token 解析出 `userId`，`StpUtil.getLoginIdByToken(token)`），**这样推送按 `userId` 即可，天然支持"多端全推"，也与业务层的认知（按人推）一致。**

### 4.12 动手验证：连接、防抖、线程名、定时任务

**A. 连上 WebSocket，看"谁登记了会话"**

1. 先登录拿到 token（HTTP 请求头是 `token: Bearer <token>`，头名见 `ruoyi-admin/src/main/resources/application.yml:103`，前缀见 `:119`），在浏览器控制台执行（WebSocket 的 URL 里放**裸 token**，不要带 `Bearer ` 前缀）：

   ```javascript
   const token = '把登录拿到的 token 粘这里';
   const ws = new WebSocket('ws://localhost:8080/websocket/message/' + token);
   ws.onopen = () => console.log('ws open');
   ws.onmessage = (e) => console.log('push:', e.data);
   ws.onclose = () => console.log('ws close');
   ```

   **预期**：控制台出现 `ws open`。握手请求不被 401 拦截，是因为 `/websocket/message/**` 在免登录白名单里（`ruoyi-admin/src/main/resources/application.yml:191`）。
2. 断点：`WebSocketServer.java:39`（`onOpen` 登记）、`WebSocketSessionManager.java:42`（放进 Map）。连上后应命中一次。
3. 关闭标签页或刷新，断点 `WebSocketServer.java:45`（`onClose`）应命中，Map 里该 token 的 List 被移除；如果 List 为空，`:49-50` 会把整个 key 删掉。

**B. 验证"3 秒防抖只发最后一条"**

在 `WebSocketSessionManager.java:57`（进入方法）、`:62`（取消旧任务）、`:66`（排新任务）、`:71`（真正发送）下断点，然后**在 3 秒内触发两次给同一个接收人的通知**（可以用系统的消息通知功能；实在没有，就临时写一个测试接口调用 `WebSocketSessionManager.sendMessageToUser(token, message)`，验证完删掉）。

**预期顺序**：第一次进入 `:57` → `:66` → `:80`；第二次进入 `:57` 后走到 `:62` 取消旧任务，再 `:66`；**3 秒内不会走到 `:71`**；最后一次排的任务到期后才 `:71` 发出一条。

**C. 看线程到底叫什么名字**

1. 用 `jps -l` 找到应用的进程号（Windows 用 `jps -l`，Linux 同）。
2. 执行 `jstack <pid> | findstr "schedule-pool"`（Linux 用 `grep`）。

**预期**：能看到 `schedule-pool-0`、`schedule-pool-1`……这是 `scheduledExecutorService` 起的名字（`ThreadPoolConfig.java:61`）；如果还看到 `pool-N-thread-M`，那通常是 `WebSocketSessionManager` 里 `Executors.newScheduledThreadPool(20)` 的裸线程池（`:39`）——**两种名字一对比，就能看出"受管线程池"和"裸线程池"的区别。**
3. 断点验证开关：在 `ThreadPoolConfig.java:44`（`threadPoolTaskExecutor` 方法）下断点，当前 `thread-pool.enabled: false`（`application.yml:324`）时**永远不会命中**；`:59`（`scheduledExecutorService` 方法）启动时必命中。

**D. 定时任务**

1. 启动 `xxl-job-admin`（默认调度中心地址配在 `application-dev.yml:87`：`http://localhost:9100/xxl-job-admin`），在"执行器管理"里看到 `xxl-job-executor` 在线（`appname` 配在 `application-dev.yml:93`）。
2. 在调度中心新建任务，JobHandler 填 `archiveReturnHandler`，手动执行一次。
3. 在 `zxcm-contract/src/main/java/com/ruoyi/contract/service/ContractJobService.java:96` 下断点，**预期命中**，并且日志里能看到任务执行记录。
4. 注意 `access-token: ""`（`application-dev.yml:89`）表示当前不鉴权——**生产环境必须改**，§4.9 已提醒。

---

## 五、一个操作的完整生命流程

**场景一：张三审批了李四的合同，系统实时通知李四**

```
┌─ 1. 两人都建立了 WebSocket 连接（页面加载时） ────────────────┐
│  李四前端：new WebSocket("ws://ip:8080/websocket/message/李四的token")│
│    → 后端 WebSocketServer.onOpen(session, 李四的token)(:37)  │
│    → WebSocketSessionManager.addUserSession(token, session)(:41) │
│    → userSessions = { 李四token: [session1] }                │
└────────────────────────────────────────────────────────────┘
                    ↓  （张三点"审批通过"）
┌─ 2. 业务处理（zxcm-workflow 审批逻辑） ──────────────────────┐
│  审批通过 → 触发"消息通知" → notify 模块                    │
│    → SystemSendMsgHandle.SendMsg(..., es_receiver=李四token)  │
└────────────────────────────────────────────────────────────┘
                    ↓
┌─ 3. 推送（WebSocketSessionManager.sendMessageToUser :57） ──┐
│  ① userTasks.remove(李四token) → 查有无待推任务              │
│  ② 有 → cancel（防抖，:62）；没有 → 跳过                     │
│  ③ scheduler.schedule(() -> { 发消息 }, 3, SECONDS)(:66/:77) │
│  ④ userTasks.put(李四token, task)(:80)                      │
└────────────────────────────────────────────────────────────┘
                    ↓  （3 秒后，期间若无新消息）
┌─ 4. 调度线程执行 ──────────────────────────────────────────┐
│  sessions = userSessions.get(李四token) → [session1]         │
│  session1.getBasicRemote().sendText(JSON)(:71)              │
│    → 李四浏览器收到 {"type":"approval","title":"审批通过","content":"..."}│
└────────────────────────────────────────────────────────────┘
                    ↓
┌─ 5. 前端处理 ──────────────────────────────────────────────┐
│  根据 message.type 决定展示方式（弹窗/角标/通知栏）           │
└────────────────────────────────────────────────────────────┘
```

**场景二：每天凌晨 2 点检查"档案该归还了"**

```
┌─ 1. 应用启动 ──────────────────────────────────────────────┐
│  XxlJobConfig.xxlJobExecutor()(:65)                        │
│    → 注册执行器到调度中心（appname=xxl-job-executor, port=9101）│
│    → 扫描 @XxlJob("archiveReturnHandler")（:96）             │
└────────────────────────────────────────────────────────────┘
                    ↓  （调度中心按 CRON 到点触发）
┌─ 2. 调度中心"通知"某个执行器实例 ──────────────────────────┐
│  ★只在"一个"执行器实例上触发（避免集群重复执行）              │
└────────────────────────────────────────────────────────────┘
                    ↓
┌─ 3. 执行 @XxlJob 方法 ─────────────────────────────────────┐
│  ContractJobService.archiveReturnHandler()                  │
│    ├─ 查"该归还的档案"                                       │
│    ├─ 更新状态 / 发通知                                      │
│    ├─ 可能调 @Async 方法 → 用 scheduledExecutorService 的线程 │
│    └─ 可能调 WebSocketSessionManager 推送提醒                │
│  执行结果（成功/失败/耗时/日志）回传给调度中心                 │
└────────────────────────────────────────────────────────────┘
                    ↓
┌─ 4. 运维在 XXL-Job 界面看执行记录 ──────────────────────────┐
│  可看：执行时间、耗时、日志、成功失败 → 失败可配重试/告警       │
└────────────────────────────────────────────────────────────┘
```

**两个场景对照的收获**：

| | WebSocket 推送 | 定时任务 |
|---|---|---|
| **触发者** | **业务动作**（审批通过） | **调度中心（时间）** |
| **执行线程** | `WebSocketSessionManager` 的裸调度池 | XXL-Job 执行器线程 / `scheduledExecutorService` |
| **"只做一次"的保证** | 靠"3 秒防抖" | 靠"调度中心只通知一个实例" |
| **共同的底层** | **线程池** | **线程池** |

**★ 核心认知**：**不管消息推送还是定时任务，最终都要落到"线程池"上执行。** **所以"线程池配得对不对"决定了整个系统的稳定性**——**这也是为什么 §2.4/§2.5 讲参数和拒绝策略要讲那么细。**

---

## 六、面试官高频问题 + 答案

**Q1：为什么用 WebSocket 不用轮询？**

**A**：**实时性和开销**。**轮询是"前端反复问后端有没有新消息"**——**1000 个用户 5 秒轮询一次 = 每秒 200 个请求，其中 99% 是白问**。**WebSocket 建一条长连接，后端有消息直接推**——**真正实时、开销低**。
**代价**：**要维护"谁在线"的会话表（`WebSocketSessionManager`）、要处理断线重连、长连接占连接数**。**我们的场景（审批提醒、待办通知）实时性有要求但消息量不大，WebSocket 正合适。**

**Q2：`@Async` 需要注意什么？**

**A**：**最大的坑是"不配线程池"**。**`@Async` 默认用 `SimpleAsyncTaskExecutor`，它每次调用都新建一个线程、不复用**——**高并发直接 OOM**。**所以必须配一个真正的线程池并让 `@Async` 用它**（我们通过 `AsyncConfig.getAsyncExecutor()`（`:48`）指向 `scheduledExecutorService`）。
**另外**：**`@Async` 方法①不能是 `private`/`static`（代理不生效）、②同类内部调用不走代理（也失效）、③异常进不了全局异常处理器（第 11 章）**。

> **追问：为什么同类内部调用 `this.asyncMethod()` 会失效？**
> **一句话答**：`@Async` 靠 Spring 生成代理对象生效；`this` 是"原始对象"、不是代理，绕过了拦截逻辑，所以方法会同步执行。要从另一个 Bean 调用，或注入自己（`@Lazy`）再调。

**Q3：线程池参数怎么定？你们配的多少？**

**A**：**先看任务类型**：**CPU 密集型 → 线程数 ≈ CPU 核数 + 1**；**IO 密集型 → 可以多一些**。**我们的异步任务（记日志、发消息）都是 IO 密集型**，配 `corePoolSize=8`。
**执行逻辑**：核心 → 队列 → 临时线程 → 拒绝策略。
**坦白**：**我们 `thread-pool` 里的 `maxPoolSize=16`、`queueCapacity=128` 因为 `enabled: false` 其实没生效**——**真正跑的是 `scheduledExecutorService`（8 个核心线程 + 无界队列），只有 `corePoolSize` 起了作用。**

**Q4：拒绝策略为什么用 `CallerRunsPolicy`？**

**A**：**它是唯一"既不丢任务、又不抛异常、又不无界堆积"的策略**——**队列满了就让"提交任务的线程"自己跑**。**效果是"自动反压"**：**调用方被占用去干活了，就没空继续生产新任务，生产速度自然降下来。**
**代价是"调用方被拖慢"**——**但我们的异步任务（记日志、发消息）"不丢"比"快"重要，所以这个取舍是对的。** **对比**：`AbortPolicy` 会抛异常丢任务、`DiscardPolicy` 静默丢弃、`DiscardOldestPolicy` 丢最老的。

**Q5：WebSocket 那个"3 秒防抖"是干什么的？**

**A**：**防止"消息轰炸"**。**如果 3 秒内给同一用户推了 5 条消息（比如批量操作为每条都触发提醒），用户会被弹窗刷屏。** 我们的做法是：**每来一条新消息，就"取消上一条待推的任务、重新安排 3 秒后推"**（`WebSocketSessionManager.java:57-80`）——**连续多条被合并成"最后一次"**。
**代价**：**中间几条的内容丢了**——**所以这个设计只适合"通知类"消息（知道有新消息就行），不适合"每条都要送达"的消息。**

**Q6：为什么要给线程池的线程起名字？**

**A**：**为了排查问题**。**默认线程名是 `pool-1-thread-5` 这种，`jstack` 一看根本不知道它是哪个池的线程。** 我们给调度池起名 `schedule-pool-%d`（`ThreadPoolConfig.java:61`）——**出问题时线程 dump 里一眼就能定位。** **这是线程池最佳实践，但很多人不做。**

**Q7：为什么用 XXL-Job 不用 Spring `@Scheduled`？**

**A**：**两个致命问题**：① **`@Scheduled` 没有界面**，改 CRON 要重新部署；② **集群下会重复执行**——**3 台服务器就各跑一次**，"每天发一次日报"变成发 3 次。**Quartz 能解决重复执行，但要建一堆 `QRTZ_*` 表、集群配置复杂、也没界面。**
**XXL-Job 有独立调度中心 + Web 界面**（CRON、启停、手动触发、看日志、失败重试都有），**调度中心只通知一个执行器实例，天然解决集群重复**，**而且执行器零侵入**（写个 `@XxlJob` 方法就行）。
**代价**：**多部署一个 `xxl-job-admin`**——**所以我们对小项目场景做了 `enabled` 开关。**

**Q8：`ScheduledThreadPoolExecutor` 的 `maxPoolSize` 有用吗？**

**A**：**没用**。**它是"固定大小"的线程池**——**内置了一个无界的延时队列（`DelayedWorkQueue`）**，**任务永远先塞队列，所以永远不会"因为队列满而新建临时线程"**——**线程数恒等于 `corePoolSize`。** **这也是为什么我们 `thread-pool.maxPoolSize=16` 实际不生效的一部分原因。**

**Q9：线程池怎么优雅关闭？**

**A**：**三步**（`Threads.java:49` 的 `shutdownAndAwaitTermination`）：**① `shutdown()` 停止收新任务、等已提交的跑完；② `awaitTermination(120s)` 等最多 120 秒；③ 超时则 `shutdownNow()` 强制中断 + 丢弃队列任务，再等一次，还超时就算了（记日志）。** **直接 `shutdownNow()` 会丢任务；只 `shutdown()` 不等会泄漏线程。**
**坦白**：**`WebSocketSessionManager` 里那个线程池（`Executors.newScheduledThreadPool(20)`）是裸池、不在 Spring 管理下，所以没有这个优雅关闭流程**——**这是我们的一个改进点。**

**Q10：`afterExecute` 钩子是干什么的？**

**A**：**"任务执行后的回调"，用来兜住"被静默吞掉的异常"**。**`ScheduledExecutorService` 提交的任务如果抛异常、而没人调 `Future.get()`，异常就困在 Future 里没人知道。** 我们在 `afterExecute` 里调 `Threads.printException`（`ThreadPoolConfig.java:66`），**它主动 `future.get()` 把异常抛出来、写进日志**（`Threads.java:69-87`）。**这是"异步任务异常不丢失"的标准解法。**

### 本章答辩卡（3 组追问级问答）

**卡 1｜问：3 秒防抖会把中间消息丢掉，如果审批通知必须"每条都送达"，你怎么办？**

- **10 秒答案**：防抖只适合"知道有新消息就行"的通知。当前实现每条新消息都会 `cancel(true)` 掉上一条待推任务（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketSessionManager.java:60-63`），最后只发最后一条（`:71`）。要求每条送达就不要用防抖：把消息先落库/进 Redis 队列，再按顺序推，并做"送达确认 + 未确认重推"。
- **证据**：`WebSocketSessionManager.java:60-63`、`:71`、`:77`。

**卡 2｜问：`scheduledExecutorService` 的队列是无界的，`CallerRunsPolicy` 能挡住任务堆积吗？**

- **10 秒答案**：不能。`ScheduledThreadPoolExecutor` 用的是无界的 `DelayedWorkQueue`，任务总是先入队，队列永远不会"满"，所以 `CallerRunsPolicy`（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/ThreadPoolConfig.java:62`）几乎没有触发机会；它也不会因为队列满而新建线程，线程数恒等于 `corePoolSize`。真正的风险是**任务无限排队直到内存耗尽**，要靠"提交限流/监控队列长度"来控制。
- **证据**：`ThreadPoolConfig.java:60-62`；`ruoyi-admin/src/main/resources/application.yml:324`（`enabled: false`，另一个线程池 Bean 未创建）、`:326`（`corePoolSize`）。

**卡 3｜问：WebSocket 把 token 放在 URL 里，这个鉴权设计有什么问题？**

- **10 秒答案**：端点本身在免登录白名单里（`ruoyi-admin/src/main/resources/application.yml:191`），`onOpen` 只把 token 当键存下来（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketServer.java:37-40`），**没有校验 token 是否有效、是否过期**。URL 还容易进访问日志/代理日志。改进：`onOpen` 里用 `StpUtil.getLoginIdByToken(token)` 校验，失败就 `session.close()`。
- **证据**：`application.yml:191`、`WebSocketServer.java:35-40`、`WebSocketSessionManager.java:41-43`。

---

## 七、选型对比

### 7.1 服务端推送方案

| 方案 | 实时性 | 开销 | 双向 | 浏览器兼容 | 本项目 |
|---|---|---|---|---|---|
| **轮询** | 差 | **高** | ✗ | **最好** | ✗ |
| 长轮询 | 好 | 中 | ✗ | 好 | ✗ |
| SSE | 好 | 低 | ✗（单向） | 好（IE 不支持） | ✗ |
| **WebSocket（本项目）** | **最好** | **低** | **✔** | 好（现代浏览器全支持） | **✔** |

**结论**：**只要"服务端需要主动推"，WebSocket 就是默认答案**（实时、双向、开销低）。**纯"服务端→客户端"的单向广播，SSE 更轻**；**兼容老浏览器（IE）才退而求其次用轮询/长轮询。**

### 7.2 异步方案

| 方案 | 可靠性 | 复杂度 | 跨服务 | 本项目 |
|---|---|---|---|---|
| **`@Async` + 线程池（本项目）** | **中（进程内，重启会丢）** | **低** | ✗ | **✔** |
| Spring `ApplicationEvent` | 中 | 低 | ✗ | ✗ |
| **消息队列（RabbitMQ/Kafka/RocketMQ）** | **高（持久化、可重试）** | **高** | **✔** | ✗ |

**结论**：**`@Async` 适合"进程内、丢了也能接受"的任务**（记日志、发通知）——**一旦应用重启，队列里没跑完的任务就没了。** **要"不丢、可重试、跨服务"，就得上 MQ。** **本项目用 `@Async`，说明这些异步任务"允许丢"**（日志丢了不致命，消息可以再触发）。

### 7.3 定时任务方案

| 方案 | 可视化 | 集群不重复 | 侵入性 | 适用 | 本项目 |
|---|---|---|---|---|---|
| **Spring `@Scheduled`** | ✗ | **✗（每台都跑）** | 无 | 单机简单任务 | ✗ |
| **Quartz** | ✗（要自己写） | ✔（靠 DB 锁，复杂） | **高（`QRTZ_*` 表）** | 复杂调度、日历式 | ✗ |
| **XXL-Job（本项目）** | **✔（独立控制台）** | **✔（调度中心路由）** | **低** | 分布式定时任务 | **✔** |
| Elastic-Job / PowerJob | ✔ | ✔ | 中 | 分布式 | ✗ |

**结论**：**"有 Web 界面 + 天然分布式 + 低侵入"是 XXL-Job 的核心竞争力**，**国内中台系统几乎默认它。** **Quartz 更"重"、更适合"复杂日历调度"；`@Scheduled` 只适合"单机 + 简单"。** **本项目的三个任务（档案归还、错误消息、数据同步）都是"分布式 + 要能手动触发/看日志"的场景，XXL-Job 合适。**

---

## 延伸阅读（网上优秀资料）

- **JSR-356 / `javax.websocket` 官方教程**：搜「Java WebSocket API @ServerEndpoint 教程」—— **`@OnOpen`/`@OnMessage`/`@OnClose`/`@OnError` 的完整用法**（本项目 `WebSocketServer` 的理论基础）。
- **MDN · WebSocket API**：<https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket> —— **前端怎么连、怎么收消息**（理解"为什么 token 要走 URL 参数"：浏览器原生 WebSocket 不支持自定义 header）。
- **Spring · `@Async` 官方文档**：Spring 官方搜「@Async / Task Execution and Scheduling」—— **务必看关于"默认 `SimpleAsyncTaskExecutor` 警告"那段**（§2.3）。
- **Java 并发 · `ThreadPoolExecutor` 与参数**：搜「Java 线程池 核心参数 执行流程 面试」—— **"核心→队列→最大→拒绝"的流程、四种拒绝策略**必背。
- **美团技术博客 · 线程池**：搜「美团 线程池 最佳实践」—— **生产环境线程池怎么配、怎么监控**（国内最经典的线程池文章）。
- **XXL-Job 官方文档**：<https://www.xxl-job.cn/> —— **调度中心 + 执行器架构、`@XxlJob`、路由策略、失败重试**（本项目定时任务的完整指南）。
- **XXL-Job · GLUE 与分片广播**：官方文档对应章节—— **"分片广播"是"大数据量任务并行处理"的关键特性**（本项目未用，值得了解）。
- **`ScheduledThreadPoolExecutor` 源码分析**：搜「ScheduledThreadPoolExecutor DelayedWorkQueue 原理」—— **理解为什么它的 `maxPoolSize` 无效**（§6 Q8）。

---

## 本章小结 & 下一步

- **WebSocket**：`WebSocketConfig`（`ServerEndpointExporter :28`）+ `WebSocketServer`（`@ServerEndpoint("/websocket/message/{token}") :35`，`@OnOpen` 登记会话 `:39`）+ `WebSocketSessionManager`（**`ConcurrentHashMap` + `CopyOnWriteArrayList` `:35`**）。
- **★ 精妙设计**：**`sendMessageToUser` 的"3 秒防抖"（`:57-80`）**——**取消上一条待推任务、重排新的，连续多条合并成最后一次**（通知类消息防轰炸）。
- **异步**：`@EnableAsync :35` + `getAsyncExecutor() :48`（**指定线程池**，避免默认 `SimpleAsyncTaskExecutor` 的"每次新建线程"）+ `getAsyncUncaughtExceptionHandler() :56`（**异常只能记日志**，第 11 章的坑）。
- **线程池**：`threadPoolTaskExecutor`（有条件 `:43`，本项目**未启用**）+ `scheduledExecutorService`（`:58`，`ScheduledThreadPoolExecutor` + `BasicThreadFactory` 起名 `:61` + 守护线程 + `CallerRunsPolicy :62` + `afterExecute` 兜异常 `:64`）。
- **★ 拒绝策略 `CallerRunsPolicy`**：**不丢任务 + 自动反压**（调用方被占用去跑任务，生产速度自然降）；**代价是调用方被拖慢**。
- **`Threads`**：`shutdownAndAwaitTermination`（`:49` **优雅停机三步：shutdown → awaitTermination(120s) → shutdownNow**）+ `printException`（`:69` **主动 `future.get()` 把困住的异常逼出来**）。
- **定时任务 = XXL-Job**：`XxlJobConfig`（`@ConditionalOnProperty("xxl.job.enabled") :35`，`xxlJobExecutor :65`）+ `@XxlJob("任务名")`（本项目 3 个：`ContractJobService.java:96` / `SysErrorServiceJob.java:43` / `SyncJobService.java:34`）。**选它是因为"有界面 + 集群不重复 + 低侵入"，代价是多部署一个 `xxl-job-admin`。**
- **坦白点**：① **`thread-pool.enabled: false`（`application.yml:324`）→ `maxPoolSize`/`queueCapacity`/`keepAliveSeconds`/`rejectedExecutionHandler` 全部不生效**，只有 `corePoolSize` 起作用；② **WebSocket 用裸线程池 `Executors.newScheduledThreadPool(20)`（`WebSocketSessionManager.java:39`）**——不受 Spring 管理、无优雅停机、默认 `AbortPolicy` 会丢消息；③ **WebSocket 以 token 为键推消息**——**推送方必须知道接收人 token、多端只推到一个 token**，改成 `userId` 更合理。
- **可迁移的思维**：**"防抖"**（合并连续事件）、**"给线程起名"**（排查友好）、**"反压优于丢弃"**（`CallerRunsPolicy`）、**"异步异常要主动逼出来"**（`afterExecute` + `future.get()`）、**"集群任务必须有分布式协调"**（XXL-Job vs `@Scheduled`）。

**下一章（第 15 章 · 基于基座写一个新模块）**：前面 14 章把基座的每一块拆开讲透了，**第 15 章把它们合起来用**——**假设你要在基座上从零加一个"公告管理"模块，完整走一遍**：建表、写 Entity/Mapper/Service/Controller、挂权限码、配数据权限、加日志注解、写前端页面、接菜单。**这是"综合实战"，也是检验你"真的读懂基座了没有"的试金石。**
