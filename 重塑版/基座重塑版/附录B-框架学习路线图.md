# 附录 B · 框架学习路线图

> **这一篇不是「新知识」，是「用网上的资料继续深挖」的地图。**
>
> 第 0~15 章讲的都是**「本仓库这一份基座」**。但面试、跳槽、换项目时，你要面对的是**「这类技术」**——Sa-Token、MyBatis-Plus、Redis、EasyExcel、XXL-Job……**这一篇告诉你**：
> **① 每个技术该看什么（网上最好的资源）；② 按什么顺序看（先学什么后学什么）；③ 看的时候「落回本仓库哪儿」验证。**
>
> **⚠️ 铁律（第 0 章 §3.2）**：**网上 RuoYi-Vue-Plus 教程绝大多数是 5.x**（`org.dromara`/`jakarta`/JDK 17），**与本站 `4.0.1s`**（`com.ruoyi`/`javax`/JDK 11）**代码对不上**。**只学「设计思想」，代码落回本仓库**——本卷每一处 `file:line` 都是答案册里真实存在的，这是你**对照学习的锚点**。

---

## 一、这条路线解决什么问题

**你的现状**（记忆里的画像）：**有 Java 基础、无企业实战、要拿这套项目求职答辩**。所以学习目标不是「成为架构师」，而是三条：

| 目标 | 标准 | 怎么达成 |
|---|---|---|
| **读懂** | 拿到基座任意一个类，能说出它「做什么、为什么这么设计」 | 本卷 0~14 章 |
| **能改** | 能在基座上**从零加一个模块**、改一个权限、加一个日志 | 本卷第 15 章 |
| **答辩说得清** | 面试官追问「为什么用它 / 它的坑在哪」，你能**主动交代** | 本卷每章 §6 Q&A + 附录 A §五 |

**这条路线分四阶段**，**严格按顺序**——前一个没踏实，后面的看了也是白看：

```
阶段 1「读透本基座」  ← 本卷 0~15 章（现在的你，在这里）
      ↓
阶段 2「逐技术深挖」  ← 6 大技术，每个配官方文档 + 本仓库验证点
      ↓
阶段 3「补底层原理」  ← Spring / MyBatis / Redis 源码级
      ↓
阶段 4「拓后端能力」  ← 分布式、消息队列、微服务（求职加分项）
```

---

## 二、阶段 1：读透本基座（对应本卷 0~15 章）

**这一阶段的资源就是本卷本身**——不用看网上的 RuoYi 教程（版本对不上）。**读的时候配合三样**：

1. **本卷正文**（第 0~15 章）。
2. **答案册源码**（`E:\Develop\project\zhaoxin\zhaoxin-contract\`）——**看到 `file:line` 就打开对照**。
3. **`ARCHITECTURE.md`**（项目根）——总纲，请求主链路 `:73-82`、依赖方向 `:46`。

**推荐阅读顺序**（和本卷章节号一致，但可按重点调）：

| 顺序 | 读什么 | 目的 |
|---|---|---|
| 1 | 第 0 章 | 建立「基座 vs 业务」「版本红线」「依赖方向」三块地基 |
| 2 | 第 1~3 章 | 把**权限（认证+鉴权）**读透——**本卷两大重点之一** |
| 3 | **第 4 章** ★ | 数据权限——**本卷最该反复读的一章** |
| 4 | **第 5 章** ★ | 多点登录——**本卷另一大重点** |
| 5 | 第 6~14 章 | 支撑能力，按需读（面试高频的 7/8/11/12 章先读） |
| 6 | **第 15 章** ★★ | 综合实战——**读完 0~14 章一定要动手走一遍** |
| 7 | 附录 A/B | 速查 + 路线（就是本文） |

> **检验「阶段 1 读透了没有」的自测**：**关上文档，能不能对着源码把「一次带鉴权的请求」从前端点到数据库讲一遍？** 能讲清（第 0 章 §5 那张图），就过关了。

---

## 三、阶段 2：逐技术深挖（6 大技术）

> **每个技术配三样**：**① 官方/最佳资源**；**② 看的时候重点看什么**；**③ 落回本仓库哪儿验证**。
> **原则**：**先看本卷对应章节建立「直觉」，再看官方文档补「全貌」，最后回本仓库「验证」。**

### 3.1 Sa-Token（★ 本卷重点，权限与登录）

| 项 | 内容 |
|---|---|
| **官方文档** | <https://sa-token.cc/>（**注意左上角切版本**，本项目用 **1.29.0**） |
| **最该看的章节** | 「登录认证」「注解鉴权」「**同端互斥登录**」「**多账号体系**」「JWT 集成」 |
| **一句话定位** | 轻量级 Java 权限框架——用最少的 API 做「登录 + 权限 + 多端」 |
| **落回本仓库** | 登录：`SysLoginService`、`LoginHelper.loginByDevice`（第 2 章）；拦截器：`SaTokenConfig.java:55/79`（第 3 章）；同端互斥：`application.yml:109/111`（第 5 章）；JWT：`StpLogicJwtForStyle`（第 2 章） |
| **本仓库的「变异」** | 它把 Sa-Token 的**存储层换成了 Redis**（`PlusSaTokenDao`，第 2 章）——**这是本项目最有价值的「二次开发」案例**，务必读透 |
| **面试怎么用** | 被问「为什么不用 Spring Security」→ 第 0 章 §7.2；被问「多端登录怎么做」→ 第 5 章 |

### 3.2 MyBatis-Plus（★ 本卷重点，ORM 与数据权限）

| 项 | 内容 |
|---|---|
| **官方文档** | <https://baomidou.com/>（**注意版本 3.5.2**） |
| **最该看的章节** | 「条件构造器 `Wrapper`」「**插件（分页 / 乐观锁 / 逻辑删除 / 多租户 / 数据权限）**」「自动填充」「**代码生成器**」 |
| **一句话定位** | MyBatis 的增强——**CRUD 免写、条件构造器、插件链** |
| **落回本仓库** | 插件链怎么配：`MybatisPlusConfig.java:44-57`（第 7 章）；数据权限：`PlusDataPermissionHandler.java`（第 4 章）；`BaseMapperPlus` 的 VO 方法（第 7 章）；`@TableLogic`/`@TableField(fill)`（第 7 章、第 15 章） |
| **本仓库的「定制」** | **数据权限被本项目做成了「双轨」**（标准 `DataScopeType` + 定制 `ContractPermission`），且**支持自定义维度**（第 4 章）——**官方文档里没有，这是本项目的独创** |
| **附带知识点** | **JSqlParser**（MyBatis-Plus 数据权限的底层，用来解析/改写 SQL）：<https://github.com/JSQLParser/JSqlParser> —— 看它的 README 就懂「为什么能无痕改 SQL」 |
| **面试怎么用** | 被问「数据权限怎么做的」→ 第 4 章；被问「逻辑删除的坑」→ 第 7 章 §4 + 附录 A #12/#13 |

### 3.3 Redis / Redisson（缓存、分布式锁、限流）

| 项 | 内容 |
|---|---|
| **Redis 官方** | <https://redis.io/docs/> |
| **必看的「八股」** | **缓存穿透 / 击穿 / 雪崩**（三大缓存问题）、**持久化 RDB/AOF**、**过期策略与内存淘汰**、**单线程模型为什么快** |
| **中文最佳** | 「小林 coding · Redis」或 [JavaGuide · Redis](https://javaguide.cn/database/redis/redis-questions-01.html) —— **面试八股最全** |
| **Redisson 官方** | <https://github.com/redisson/redisson/wiki> —— **看「分布式锁」「分布式集合」「缓存」** |
| **落回本仓库** | Redisson 客户端创建：`RedisConfig.java`（第 12 章）；`RedisUtils` 总闸（第 12 章）；防重/发号用 Redis（第 9 章、第 3 章） |
| **本仓库的「坦白点」** | **Spring Cache 配了没用**（第 12 章 + 附录 A #33）、**字典缓存无 TTL**（#34）——**这两条正好是「缓存三大问题」的活教材** |
| **面试怎么用** | 被问「缓存穿透怎么解决」→ 用本仓库 `SysConfigServiceImpl.java:94`（不缓存空值）当反例讲；被问「分布式锁」→ Redisson 的 `RLock`（本项目间接用了 Redisson 的组件，但**没用显式分布式锁**，要如实说） |

### 3.4 EasyExcel（★ 导入导出）

| 项 | 内容 |
|---|---|
| **官方文档** | <https://easyexcel.opensource.alibaba.com/>（**版本 3.0.5**） |
| **最该看的章节** | 「读 Excel」「写 Excel」「**自定义 Converter**」「**SAX 流式读**（为什么不怕大文件 OOM）」 |
| **一句话定位** | 阿里出品——**基于 SAX、逐行读、内存友好**的 Excel 库（vs POI 的一次性全加载） |
| **落回本仓库** | 封装：`ExcelUtil.java`（第 13 章）；字典翻译 Converter：`ExcelDictConvert.java`（第 13 章、第 15 章 §4.6）；注解：`@ExcelProperty`/`@ExcelDictFormat` |
| **本仓库的「巧思」** | **`@ExcelDictFormat` + `ExcelDictConvert`**——把「字典值↔标签」的翻译**做成了注解驱动**（第 13 章）——**这是官方没有、本项目加的能力** |
| **面试怎么用** | 被问「大 Excel 导入怎么防 OOM」→ 讲 SAX 流式读 + `DefaultExcelListener`；被问「导出为什么要字典翻译」→ 第 15 章 §4.6 |

### 3.5 XXL-Job（★ 分布式定时任务）

| 项 | 内容 |
|---|---|
| **官方文档** | <https://www.xxl-job.cn/>、代码 <https://github.com/xuxueli/xxl-job> |
| **最该看的章节** | 「调度中心 + 执行器架构」「**路由策略**」「**分片广播**」「失败重试」「GLUE 模式」 |
| **一句话定位** | 有 Web 界面、**天然分布式（集群不重复执行）**、低侵入的定时任务平台 |
| **落回本仓库** | 配置：`XxlJobConfig.java`（第 14 章）；用法：`@XxlJob("任务名")`——本项目 3 处（`ContractJobService.java:96` 等，第 14 章） |
| **关键对比** | **Quartz**（重、要 `QRTZ_*` 表、集群靠 DB 锁）、**`@Scheduled`**（单机、集群会重复跑）、**XXL-Job**（轻、有界面、天然集群）——**第 14 章 §7.3 有完整对比表** |
| **面试怎么用** | 被问「集群下定时任务会不会重复执行」→ 这是**送分题**：`@Scheduled` 会，XXL-Job/Quartz 不会。**能顺带说出「分片广播怎么让大数据量任务并行」就是加分** |

### 3.6 WebSocket（★ 服务端主动推送）

| 项 | 内容 |
|---|---|
| **MDN（前端视角）** | <https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket> —— **看「浏览器怎么连、怎么收消息」**，理解**为什么本项目 token 要走 URL 参数**（浏览器原生 WebSocket 不支持自定义 header） |
| **JSR-356（Java 视角）** | 搜「Java WebSocket API `@ServerEndpoint` 教程」——`@OnOpen`/`@OnMessage`/`@OnClose`/`@OnError` 四件套 |
| **Spring 官方** | Spring 也封装了 WebSocket（`spring-websocket`）——**看清「原生 JSR-356 vs Spring 封装」的区别**（本项目用**原生**） |
| **落回本仓库** | `WebSocketServer.java`、`WebSocketSessionManager.java`（第 14 章） |
| **本仓库的「精妙设计」** | **`sendMessageToUser` 的「3 秒防抖」**（`WebSocketSessionManager.java:57-80`）——**连续多条合并成最后一次**（第 14 章）——**这个可以当亮点讲** |
| **对比** | **轮询 / 长轮询 / SSE / WebSocket** 四方案对比——第 14 章 §7.1 |
| **面试怎么用** | 被问「服务端怎么主动推消息」→ 先讲四方案对比，再说本项目选 WebSocket 的理由 + 3 秒防抖设计 |

### 3.7 配角技术（快速过一遍即可）

| 技术 | 资源 | 本仓库落点 | 讲在哪章 |
|---|---|---|---|
| **Knife4j（Swagger 增强）** | <https://doc.xiaominfo.com/> | `@Api`/`@ApiOperation` 遍布 Controller；文档页 `/doc.html` | 第 11 章 |
| **Undertow** | <https://undertow.io/> | `application.yml:35-47`（io=8/worker=888） | 第 0 章 |
| **Hutool（工具库）** | <https://hutool.cn/> | `BeanUtil`/`JSONUtil`/`StrUtil` 遍布（第 15 章 §4.9 用了 `BeanUtil.toBean`） | 各处 |
| **Lombok** | <https://projectlombok.org/> | `@Data`/`@RequiredArgsConstructor`/`@EqualsAndHashCode`——**所有 Entity/BO/VO** | 第 15 章 §4.9 |
| **Druid（连接池）** | <https://github.com/alibaba/druid> | `pom.xml:21`；监控页 `/druid` | 第 7 章 |
| **dynamic-datasource（多数据源）** | <https://www.kancloud.cn/tracy5546/dynamic-datasource/2264611> | `pom.xml:36` | 第 0 章 |
| **Jakarta Bean Validation** | <https://beanvalidation.org/> | `@NotBlank`/`@NotNull` + 分组（第 15 章 §4.5） | 第 15 章 |

---

## 四、阶段 3：补底层原理（从「会用」到「懂为什么」）

**这一阶段不再看「框架怎么用」，而是看「框架底下是什么」。** 求职中层岗位的**分水岭**就在这里。

| 主题 | 资源 | 为什么重要 |
|---|---|---|
| **Spring IoC / AOP / 事务** | 官方文档 <https://spring.io/projects/spring-boot> + [JavaGuide · Spring](https://javaguide.cn/system-design/framework/spring/spring-knowledge-and-questions-summary.html) | **基座的「自动装配」「AOP 切面（`@Log`/`@RepeatSubmit`）」「`@Transactional`」全靠它**。不懂 AOP，就讲不清「操作日志怎么自动记的」 |
| **Spring Boot 自动装配原理** | 搜「Spring Boot 自动配置原理 `spring.factories`/`AutoConfiguration`」 | **第 0 章 §4.3 讲的「排除自动配置」就是这**——懂原理才敢说「为什么排除 Security」 |
| **MyBatis 执行流程 / 拦截器** | 搜「MyBatis 四大组件 插件原理 `Interceptor`」 | **数据权限插件、分页插件、租户插件都是 MyBatis 拦截器**（第 4/7 章）——懂原理才讲得清「无痕改 SQL」 |
| **Java 并发 / 线程池** | [JavaGuide · 并发](https://javaguide.cn/java/concurrent/java-concurrent-questions-01.html) + 搜「美团 线程池最佳实践」 | **第 14 章的「核心→队列→最大→拒绝」流程、`CallerRunsPolicy`、`ScheduledThreadPoolExecutor`** 全靠它。**面试必问** |
| **JVM（内存模型 / 垃圾回收）** | 搜「JVM 八股 小林 coding / JavaGuide」 | 通用八股，与本项目不直接相关但**必考** |
| **MySQL（索引 / 事务 / 锁 / 执行计划）** | [JavaGuide · MySQL](https://javaguide.cn/database/mysql/mysql-questions-01.html) | 通用八股；**本项目你讲「数据权限改了 SQL、要加索引」时会用到**（第 15 章 §4.1） |
| **网络（HTTP / TCP / HTTPS）** | 搜「小林 coding 网络 / HTTP 八股」 | 通用八股；**第 10 章（请求包装）、第 13 章（上传）会用到** |

> **学习诀窍**：**每学一个原理，就问「本仓库哪儿用到了它」**——比如学完「MyBatis 拦截器」，回第 4 章看 `PlusDataPermissionHandler` 怎么 `implements Interceptor`。**「原理 + 本项目实例」一起记，答辩时最有杀伤力。**

---

## 五、阶段 4：拓后端能力（求职加分项）

**这些本项目**（单体应用）**没用到**，但**面试常问、跳槽常要**。**了解即可，能说出概念和应用场景就行**：

| 主题 | 资源 | 一句话 |
|---|---|---|
| **Spring Cloud（微服务全家桶）** | 搜「Spring Cloud Alibaba 教程」（Nacos/Gateway/Sentinel/OpenFeign） | 从「单体」到「微服务」——**本项目是单体**（第 0 章），**能说清「什么规模该拆微服务」就是加分** |
| **消息队列（RabbitMQ / Kafka / RocketMQ）** | 搜「消息队列 八股」 | **第 14 章 §7.2 的对比表**：`@Async` 进程内、丢了会丢；**要「不丢、可重试、跨服务」就得上 MQ** |
| **分布式锁** | Redisson 官方 `RLock` + 搜「分布式锁 三大实现（Redis/ZooKeeper/DB）」 | **第 12 章用了 Redisson**，但**本项目没用显式分布式锁**——面试若问「并发扣减库存」怎么答 |
| **分布式事务** | 搜「Seata / 最终一致性 / TCC」 | 微服务拆分后必然面对的问题 |
| **Docker / K8s** | 搜「Docker 入门 / K8s 核心概念」 | 部署相关；**本项目第 7 章的「雪花 ID 网卡方案在容器化下不稳」就是 K8s 场景下的坑**（附录 A #14） |
| **ES（Elasticsearch）** | 搜「Elasticsearch 倒排索引 八股」 | 全文检索；**合同系统天然需要「合同全文搜索」**——若被问「怎么搜合同内容」，能提到 ES 就是亮点 |
| **设计模式** | 搜「常用设计模式 在 Spring 中的应用」 | **本项目到处是**：策略模式（第 8 章回调）、模板方法（`BaseMapperPlus`）、单例（`TenantProperties.getInstance()`）——**第 0 章 §2.2 讲的「脚手架改造」也是「模板方法」思想** |

---

## 六、一张「30 天冲刺」建议表（答辩前）

> 假设你**每天能投入 3~4 小时**。**不是「学新东西」，是「把本卷读透 + 把八股串起来」**。

| 天 | 干什么 | 产出 |
|---|---|---|
| 1 | 读第 0 章 + `ARCHITECTURE.md` | 能画「基座分层 + 依赖方向」图 |
| 2~4 | 读第 1~3 章 + 对着源码走一遍登录链路 | 能在白板上讲「一次登录发生什么」 |
| **5~6** | **精读第 4 章（数据权限）** + 打开 `PlusDataPermissionHandler` 对照 | 能讲「无痕改 SQL」的全过程 |
| 7~8 | 读第 5 章（多点登录） | 能讲「同端互斥」和「在线用户」 |
| 9~10 | 读第 6~7 章（租户 + ORM） | 能讲「MP 插件链」和「逻辑删除的坑」 |
| 11~12 | 读第 8~9 章（日志 + 防重限流） | 能讲「AOP 怎么自动记日志」 |
| 13~14 | 读第 10~11 章（脱敏 XSS + 异常） | 能讲「统一返回 + 全局异常」 |
| 15~16 | 读第 12~13 章（缓存 + 文件 Excel） | 能讲「缓存三大问题」+「大 Excel 防 OOM」 |
| 17~18 | 读第 14 章（WS + 异步 + 线程池 + XXL-Job） | 能讲「线程池参数」和「定时任务集群」 |
| **19~22** | **动手走第 15 章**（真在本地起项目、加一个模块、跑 12 项自测） | **一个能写进简历的「我加过的模块」** |
| 23~25 | 过附录 A §五（46 处坦白点），标 ★ 的想清「怎么改」 | 能**主动交代**项目缺口 |
| 26~28 | 阶段 3 原理：Spring AOP / MyBatis 拦截器 / 线程池 | 八股串起来 |
| 29~30 | 全卷 §6 面试 Q&A 通读 + 自问自答 | 面试话术成型 |

> **关键**：**第 19~22 天的「动手」不能省**。**光读不写，第 15 章的 11 个接入点你记不住**；**真加过一个模块，面试时才有「具体细节」可讲。**

---

## 七、怎么用这条路线答辩

**这条路线最终是为答辩服务的。** 把「学的东西」转成「讲的料」，记住三句话：

1. **讲「设计取舍」，不讲「技术名词堆砌」**
   ❌「我们用 Sa-Token 做鉴权，用 MyBatis-Plus 做 ORM，用 Redis 做缓存……」
   ✅「**我们没从零写，而是基于 RuoYi-Vue-Plus 改造**——因为它把通用能力（鉴权/权限/字典/导入导出）都做完了，我们把时间花在合同智能比对这些差异化业务上。**代价是要适配它的既有设计，这是有意识的工程取舍**。」（第 0 章 §2.2）

2. **主动亮「坦白点」，化被动为主动**
   被问「这套系统有什么不足」——**别硬撑**，直接说：「**有几处我知道没做到位**：比如**脱敏组件配了但被一行 `return true` 关掉了**、**Spring Cache 配了但业务没用**、**线程池 `enabled:false` 导致一半配置没生效**……**这些我都记着，如果要改我知道从哪下手**。」（附录 A §五）——**这比「我们系统很完美」可信一百倍**。

3. **用「真实行号」证明「我真读过源码」**
   讲任何一点，都能落到「**这在 `XXX.java:123`**」。**这是你区别于「背八股的人」的最大武器**——本卷每章的 `file:line` 都是为此准备的。

---

## 延伸阅读（本篇引用的核心资源汇总）

> **按「官方优先」排**——官方文档永远比二手教程准（尤其版本）。

**框架官方文档**
- **Sa-Token 官方**：<https://sa-token.cc/>
- **MyBatis-Plus 官方**：<https://baomidou.com/>
- **Redisson 官方 Wiki**：<https://github.com/redisson/redisson/wiki>
- **Redis 官方**：<https://redis.io/docs/>
- **EasyExcel 官方**：<https://easyexcel.opensource.alibaba.com/>
- **XXL-Job 官方**：<https://www.xxl-job.cn/>
- **Spring Boot 官方**：<https://spring.io/projects/spring-boot>
- **Knife4j 官方**：<https://doc.xiaominfo.com/>
- **Hutool 官方**：<https://hutool.cn/>
- **MDN · WebSocket**：<https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket>

**本基座的上游项目**
- **RuoYi-Vue-Plus 仓库**：<https://github.com/dromara/RuoYi-Vue-Plus>（**注意文档是 5.x**）
- **RuoYi-Vue-Plus 官方文档**：<https://plus-doc.dromara.org/>
- **DeepWiki 源码解析**：<https://deepwiki.com/dromara/RuoYi-Vue-Plus>

**中文八股 / 系统学习（阶段 3~4）**
- **JavaGuide（综合最强）**：<https://javaguide.cn/>
- **小林 coding（图解网络/系统/MySQL/Redis）**：搜「小林 coding」
- **美团技术博客**：<https://tech.meituan.com/>（**线程池、缓存等工程实践经典**）
- **JSqlParser（数据权限底层）**：<https://github.com/JSQLParser/JSqlParser>

---

## 本篇小结 & 全卷收尾

- **四阶段路线**：**读透本基座（本卷）→ 逐技术深挖（6 大技术）→ 补底层原理（Spring/MyBatis/并发）→ 拓后端能力（微服务/MQ/ES）**。**严格按顺序**。
- **6 大技术**（各配官方文档 + 本仓库验证点）：**Sa-Token / MyBatis-Plus / Redis-Redisson / EasyExcel / XXL-Job / WebSocket**。
- **一条铁律**：**网上看思想、代码落回本仓库**（版本红线 `4.0.1s` vs `5.x`）。
- **学习诀窍**：**每学一个原理，就问「本仓库哪儿用到了」**——原理 + 实例一起记，答辩最有杀伤力。
- **30 天冲刺**：**第 19~22 天动手走第 15 章不能省**——真加过一个模块，才有「具体细节」可讲。
- **答辩三句话**：讲**设计取舍**、主动亮**坦白点**、用**真实行号**证明读过源码。

---

### 全卷到此结束

**第 0~15 章 + 附录 A/B，构成了「读懂 → 能改 → 答辩说得清」的完整闭环**：

- **第 0~5 章**（权限与登录，本卷重点）→ 你懂了「企业后台系统的安全底座」。
- **第 6~14 章**（支撑能力）→ 你懂了「一个后台系统除了业务，还需要什么」。
- **第 15 章**（综合实战）→ 你把上面全部串起来，**真正会写**。
- **附录 A**（横切索引）→ 你的**开发速查手册 + 46 处坦白点清单**。
- **附录 B**（本篇）→ 你的**继续深挖的地图**。

> **最后一句**：**这套文档的价值不在「读完」，在「用起来」**——**读第 15 章时，真的在本地起项目、加一个模块、跑一遍自测清单**。**那一刻，你就从「看开源项目的人」变成了「用开源项目、能改开源项目的人」。** 这，正是求职答辩要的答案。
