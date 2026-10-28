# 第 12 章 · 缓存 · Redis · 字典 · 系统配置

> **前面几章已经反复出现 Redis**——第 2 章的登录态、第 5 章的在线用户、第 9 章的防重标记与限流，**它们都是"Redis 的具体用法"**。**本章把这些用法背后的"基础设施"讲清楚**：Redis 客户端是怎么创建的、`RedisUtils` 这层封装解决了什么、**为什么项目自己封了一套缓存工具而不用 Spring 的 `@Cacheable`**。
>
> **本章第一次讲透**：① **`RedisConfig`** 里 Redisson 客户端的创建（单机 vs 集群、`@Primary`、`destroyMethod`）；② **`RedisUtils`** 这个"全项目 Redis 总闸"的封装思路（**又一个 `SpringUtils.getBean` 静态注入**）；③ **Spring Cache（`@Cacheable`）在本项目的真实状态**——**基础设施配好了，但业务代码里的缓存注解全被注释掉了**（★坦白点）；④ **字典与系统参数**为什么是最典型的缓存场景，以及它们的 **cache-aside（旁路缓存）** 写法。

---

## 一、这部分是做什么的？

### 1.1 一句话：缓存就是"把查过的结果先存着"

**数据库是系统最慢、最贵的一环。** 一个"读多写少"的数据（比如"字典"、"系统参数"），**如果每次请求都去查库，数据库会被无意义的重复查询压垮**。

**缓存的思想**：**第一次查库，把结果存到 Redis；后面再来，直接从 Redis 拿，不碰数据库。**

```
第一次请求：Redis 没有 → 查数据库 → 把结果写 Redis → 返回
后续请求：  Redis 有  → 直接返回（比查库快几十倍）
```

**本项目里"缓存"分三块**：

| 块 | 干什么 | 关键代码 | 状态 |
|---|---|---|---|
| **Redis 基础设施** | 创建客户端、提供统一工具 | `RedisConfig` + `RedisUtils` | **✔ 全项目在用** |
| **Spring Cache** | 用 `@Cacheable` 注解式缓存 | `@EnableCaching` + `RedissonSpringCacheManager` | **✔ 配好了，但业务注解全被注释掉**（★坦白点） |
| **业务缓存** | 字典、系统参数的缓存 | `SysDictTypeServiceImpl` / `SysConfigServiceImpl` | **✔ 手动 `RedisUtils` 实现在用** |

### 1.1.1 零基础先补概念

> **下面四个词会贯穿整章，先用大白话过一遍**——不理解它们，后面的代码只会像"咒语"。

| 概念 | 大白话 | 本项目里的真实例子 |
|---|---|---|
| **缓存（Cache）** | 把"查过的结果"临时存在一个更快的地方，下次先来这里拿，拿不到再查数据库 | 字典数据放进 Redis 后，`selectDictDataByType` 先读它（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysDictTypeServiceImpl.java:101`） |
| **Cache-Aside（旁路缓存）** | 业务代码自己控制缓存：**读**时"先看缓存 → 没有就查库 → 查完写回缓存"；**写**时"先写库 → 再改/删缓存" | 读：`SysConfigServiceImpl.java:83-95`；写：`:136-163` |
| **TTL（Time To Live，存活时间）** | 给一个缓存键设"保质期"，时间一到 Redis 自动删掉它；不设 TTL 就是永久 | 无 TTL 的写法：`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java:103`；带 TTL 的写法：`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java:137` |
| **缓存 Key（键）** | 缓存里每条数据的"门牌号"。常用"前缀 + 业务键"命名，前缀方便批量清理 | 参数缓存键 `sys_config:{configKey}`（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysConfigServiceImpl.java:249-251`），前缀常量在 `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/constant/Constants.java:141` |

**一句话串起来**：字典和参数是"读多写少"的数据 → 用 Cache-Aside 放进 Redis → Key 加前缀便于分类 → TTL 决定它会不会自动过期。

### 1.2 为什么"字典"和"系统参数"最需要缓存？

**字典（`sys_dict_data`）** 是什么：**一堆"码值"的映射**——比如 `sys_normal_disable` 这个字典，定义了 `0=正常`、`1=停用`。**前端几乎所有下拉框、状态标签都要用它**。

**系统参数（`sys_config`）** 是什么：**一堆"可配置的开关/字符串"**——比如"验证码开关""用户初始密码"。**它们被代码在运行时读取，用于改变行为。**

**这两个数据的共同特征**：

1. **读极多、写极少**——字典/参数**几个月才改一次**，但**每个页面每次加载都在读**；
2. **数据量小**——全部加起来也就几百条，**放 Redis 毫无压力**；
3. **全站共享、与用户无关**——**所有人都读同一份**（这正是"能缓存"的前提，见 §2.4）。

**所以在"能缓存"这件事上，字典和参数是"教科书级"的候选。**

### 1.3 ★ 一个必须先厘清的问题：本项目其实有"两套"缓存机制

**很多教程会告诉你"用 Spring Cache 的 `@Cacheable` 就行"**。**本项目确实配了 Spring Cache**（`RedisConfig` 上有 `@EnableCaching :47`，还配置了 `RedissonSpringCacheManager :134`）——**但是！**

```java
// ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysUserServiceImpl.java
// @CacheEvict(value=USER_CACHE)          // :406  ← 注释掉了
// @CacheEvict(value=USER_CACHE)          // :421
// @CacheEvict(value=USER_CACHE)          // :448
// @CacheEvict(value=USER_CACHE)          // :461
// @CacheEvict(value=USER_CACHE)          // :473
// @CacheEvict(value=USER_CACHE)          // :605
// @CacheEvict(value=USER_CACHE)          // :624
```

**全项目的 `@Cacheable` / `@CacheEvict` 除了注释掉的，零使用。** 也就是说：**"注解式缓存"这条路，本项目配好了却没走。** 真正在用的是**"手动 `RedisUtils` 缓存"**这条路。

**这带来一个重要认知**：**理解本项目的缓存，重点不在 `@Cacheable`，而在 `RedisUtils` + "手动 cache-aside 模式"。** 而**为什么放弃 `@Cacheable`**，恰恰是最值得想清楚的设计问题（§2.2）。

---

## 二、站在架构师 / 开发经理角度：为什么这么设计？

### 2.1 为什么用 Redisson，而不是 Spring 自带的 `RedisTemplate`？

**`RedisTemplate`** 是 Spring Data Redis 的客户端，**很基础、很通用**。**Redisson** 是一个"**基于 Redis 的 Java 分布式工具箱**"，**把很多分布式场景封装成了对象**。

**本项目选 Redisson 的三个理由**：

1. **分布式锁**——`RLock`，`tryLock` 一行搞定（本项目有 `lock4j` 分布式锁，底座就是它）；
2. **分布式限流**——`RRateLimiter` 令牌桶（第 9 章用的就是它，`RedisUtils.rateLimiter :50`）；
3. **分布式集合 / 原子数 / 发布订阅**——`RMap` / `RSet` / `RAtomicLong` / `RTopic`，**本项目 `RedisUtils` 封装的全是这些**。

**对比**：**用 `RedisTemplate` 你得自己拼 Lua 脚本、自己处理锁续期**；**Redisson 直接给你 `RLock.lock()`**。**代价是依赖变重**（Redisson 是"大而全"的库）。

**本项目里两者并存**：**业务用 Redisson（`RedisUtils`），Redis 监控页面用 `RedisTemplate`**（`CacheController.java:42` 注入的是 `RedisTemplate<String,String>`）。

### 2.2 为什么放弃 `@Cacheable`，改用手动 `RedisUtils`？

**`@Cacheable` 看起来很美**：

```java
@Cacheable(cacheNames = "sys_dict", key = "#dictType")
public List<SysDictData> selectDictDataByType(String dictType) { ... }
```

**一行注解，自动"先查缓存、没有就查库、再写缓存"。** 那为什么本项目**把它注释掉了**？

**原因有几个（结合本项目特征推断）**：

1. **多租户隔离问题（最关键）**——本项目是多租户系统（第 6 章）。**`@Cacheable` 的 key 默认只由"方法参数"拼成**，**同一个 `dictType` 在不同租户下数据可能不同**——**如果 key 不带 `tenantId`，A 租户会读到 B 租户的缓存**。**要处理这个，得自定义 key 生成器、把 `tenantId` 拼进去**，**复杂度陡增**。
2. **缓存粒度与失效控制**——`@Cacheable` 的"失效"要你**在每个写方法上标 `@CacheEvict`**，**一旦漏标就出现"改了数据缓存还是旧值"**。**手动写反而看得清**（`insertConfig` 里手动 `setCacheObject :139`、`delete` 里手动 `deleteObject :177`）。
3. **"能读懂"优先**——**手动 cache-aside 代码直白**：读的时候"先查 Redis 再查库"（`selectConfigByKey :83-95`），**一眼就知道缓存逻辑在哪**；**注解式是"隐式"的，出问题不好排查**。

**结论**：**`@Cacheable` 适合"简单、key 明确、单租户"的场景；本项目的多租户 + 缓存失效控制需求，让它选择了"手动但可控"的路。** 而 `@EnableCaching` 还留着，**说明"路没堵死，将来要用随时能用"**。

### 2.3 缓存的三道"经典防线"（面试必问）

讲缓存**绕不开三个概念**，本项目用它们来解释"手动实现"要注意什么：

| 问题 | 是什么 | 本项目怎么应对 |
|---|---|---|
| **缓存穿透** | 查一个**根本不存在的** key，缓存和库都没有，**每次都打到库** | **查不到时返回 `StringUtils.EMPTY`（`SysConfigServiceImpl.java:94`）而非写入 null**——**严格说本项目的处理是"不缓存空值"，穿透风险存在（坦白点）** |
| **缓存击穿** | 某个**热点 key 过期瞬间**，大量并发同时打到库 | 本项目**未做特殊处理**（字典/参数数据量小、更新极少，过期策略未见"互斥重建"） |
| **缓存雪崩** | **大量 key 同时过期**，请求集体打到库 | 本项目**字典/参数缓存大多无明显 TTL**（`setCacheObject(key, value)` **无过期时间**，`RedisUtils.java:103`）——**好处是不会集体过期，坏处是数据会"永久驻留"直到被更新（见 §4.8 坦白点）** |

**关键认知**：**"缓存"不是"加个 `@Cacheable` 就完事"**——**穿透/击穿/雪崩是三种不同的失效模式，要针对性地设计**。**本项目因为"数据量小、更新极少"，选择了最朴素的做法：不加 TTL、更新时主动覆盖。** **这是"场景决定方案"的典型**——**换到一个"秒杀商品"的场景，这套朴素做法立刻就不够用了。**

### 2.4 什么样的数据"能缓存"？

**这是比"怎么缓存"更根本的问题。** **判定标准（自问三句）**：

1. **读多写少吗？** —— 写比读还频繁，缓存只会带来"永远不一致"；
2. **全站共享吗？** —— 数据**因人而异**（比如"我的订单"）**不适合共享缓存**，key 必须带用户维度，否则串数据；
3. **能容忍短暂不一致吗？** —— 缓存天然是"最终一致"，**强一致的数据（账户余额）不能缓存**（或要用别的方案）。

**字典和参数三条全中**（读多写少、全站共享、不一致后果轻微）——**所以它们是最适合缓存的。** **反过来看第 2 章的 `LoginUser`**：**它带 `userId`，是"因人而异"的，所以缓存的 key 必须带 `userId`**（`token:login:token:{token}`）——**同样是缓存，key 维度的差异决定了它能不能共享。**

---

## 三、需要哪些技术？

| 技术点 | 是什么 | 在本项目哪里用到 | 关键类/方法 |
|---|---|---|---|
| **Redisson** | 基于 Redis 的分布式工具箱 | 客户端创建、锁、限流、集合 | `RedisConfig.redisson() :61` |
| **`@Primary`** | 多个同类型 Bean 时指定"默认那个" | 保证注入的 `RedissonClient` 是这一个 | `RedisConfig.java:59` |
| **`destroyMethod = "shutdown"`** | Bean 销毁时调 `shutdown()` 释放连接 | 应用关闭时优雅断开 Redis | `RedisConfig.java:60` |
| **`JsonJacksonCodec`** | Redisson 的 JSON 序列化编解码器 | 让 Redis 里存的是可读 JSON | `RedisConfig.java:69` |
| **`RedissonSpringCacheManager`** | 把 Redisson 接到 Spring Cache | `@Cacheable` 的底层（虽未启用） | `RedisConfig.java:134` |
| **`@EnableCaching`** | 开启 Spring 的注解缓存 | Spring Cache 开关 | `RedisConfig.java:47` |
| **`CachingConfigurerSupport`** | 定制 Spring Cache 行为的基类 | 配置 CacheManager | `RedisConfig.java:48` |
| **静态工具类 + `SpringUtils.getBean`** | 在静态方法里拿容器 Bean | `RedisUtils` 的核心写法 | `RedisUtils.java:39` |
| **`@ConfigurationProperties`** | 绑定 `redisson.*` 配置 | 线程数、连接池、缓存组 | `RedissonProperties` |
| **`ApplicationRunner`** | 应用启动后执行一次 | 启动时预热字典/参数缓存 | `SystemApplicationRunner.java:33` |
| **cache-aside 模式** | "先查缓存、miss 再查库、回写缓存" | 字典/参数读取的标准写法 | `SysConfigServiceImpl.java:83` |
| **`RedisUtils.keys(pattern)`** | 按前缀批量找 key | 清空某一类缓存 | `RedisUtils.java:430` |

---

## 四、功能具体怎么实现（带着你读懂）

### 4.1 创建 Redisson 客户端：`RedisConfig`

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/RedisConfig.java`）

```java
@Slf4j
@Configuration
@EnableCaching                                                   // :47  ★开启 Spring 注解缓存
public class RedisConfig extends CachingConfigurerSupport {      // :48

    @Primary                                                     // :59  ★多个同类型 Bean 时优先这个
    @Bean(destroyMethod = "shutdown")                            // :60  ★Bean 销毁时调 shutdown()
    public RedissonClient redisson() {
        String prefix = REDIS_PROTOCOL_PREFIX;                   // "redis://"
        if (redisProperties.isSsl()) {
            prefix = REDISS_PROTOCOL_PREFIX;                     // "rediss://"
        }
        Config config = new Config();
        config.setThreads(redissonProperties.getThreads())
            .setNettyThreads(redissonProperties.getNettyThreads())
            .setCodec(JsonJacksonCodec.INSTANCE)                 // :69  ★用 JSON 编解码
            .setTransportMode(redissonProperties.getTransportMode());

        // :72 单机模式
        RedissonProperties.SingleServerConfig singleServerConfig = redissonProperties.getSingleServerConfig();
        if (ObjectUtil.isNotNull(singleServerConfig)) {
            config.useSingleServer()
                .setAddress(prefix + redisProperties.getHost() + ":" + redisProperties.getPort())   // :76
                .setDatabase(redisProperties.getDatabase())      // :78  ★选库
                .setPassword(...)
                ...  // 连接池、超时等
        }
        // :90 集群模式（配了 clusterServersConfig 就走这里）
        ...
        RedissonClient redissonClient = Redisson.create(config);  // :117  ★创建客户端
        log.info("初始化 redis 配置");
        return redissonClient;
    }
```

**逐点读懂**：

- **`@Primary`（`:59`）**：**如果项目里有多个 `RedissonClient` Bean，"不指定名字时注入哪个"由 `@Primary` 决定。** 本项目只有这一个，**但标上 `@Primary` 是"声明式保险"**。
- **`destroyMethod = "shutdown"`（`:60`）**：**应用关闭时，Spring 会自动调 `RedissonClient.shutdown()`**——**优雅关闭连接池，避免连接泄漏**。**这是"资源类 Bean 该有的收尾"。**
- **`JsonJacksonCodec.INSTANCE`（`:69`）**：**Redisson 默认用二进制编解码，存进 Redis 的是乱码。** 换成 JSON 后，**你在 Redis 里 `GET` 一个 key，看到的是可读 JSON**——**排查问题时非常关键**（第 5 章讲 `online_tokens` 时，能在 Redis 里直接看懂内容，就是靠这个）。
- **单机 / 集群二选一**：**`useSingleServer()` / `useClusterServers()`**。**配置文件里只配其中一个**（`application-dev.yml` 配单机 `:184-192`：`host/port/database/password`）。
- **`RedisConfig` 不负责"业务缓存"**：**它只负责"创建客户端"和"接上 Spring Cache"**。**具体怎么缓存是 `RedisUtils` 和业务 Service 的事。**

### 4.2 `RedissonSpringCacheManager`：把 Redisson 接给 Spring Cache

```java
    /**
     * 整合spring-cache
     */
    @Bean
    public CacheManager cacheManager(RedissonClient redissonClient) {           // :126
        List<RedissonProperties.CacheGroup> cacheGroup = redissonProperties.getCacheGroup();
        Map<String, CacheConfig> config = new HashMap<>();
        for (RedissonProperties.CacheGroup group : cacheGroup) {
            CacheConfig cacheConfig = new CacheConfig(group.getTtl(), group.getMaxIdleTime());  // :130
            cacheConfig.setMaxSize(group.getMaxSize());                          // :131
            config.put(group.getGroupId(), cacheConfig);
        }
        return new RedissonSpringCacheManager(redissonClient, config, JsonJacksonCodec.INSTANCE);  // :134
    }
```

**这段的作用**：**让 Spring Cache（`@Cacheable`）用一个"存在 Redis 里、并且能配 TTL/最大容量"的 CacheManager。**

**关键：`cacheGroup` 是从配置读的**（`application.yml:341-353`）：

```yaml
redisson:
  cacheGroup:
    # 用例: @Cacheable(cacheNames="groupId", key="#XXX") 方可使用缓存组配置
    - groupId: redissonCacheMap     # :344  ← 缓存组名（对应 @Cacheable 的 cacheNames）
      ttl: 60000                    # :346  组过期时间（毫秒）= 60 秒
      maxIdleTime: 60000            # :348  最大空闲时间
      maxSize: 0                    # :350  最大容量（0 = 不限）
    - groupId: testCache
      ttl: 1000                     # :352  1 秒
      maxIdleTime: 500
```

**读懂"缓存组"**：**Spring Cache 里 `@Cacheable(cacheNames="xxx")` 的 `xxx` 就是一个"缓存组"**。**这个组可以从配置里统一设 `ttl`（过期时间）、`maxSize`（最大条数）**。

**⚠️ 注意配置里的注释**：**"用例: `@Cacheable(cacheNames="groupId", key="#XXX")`"**——**它的意思是"如果你要用 `@Cacheable`，`cacheNames` 得写这里定义的 `groupId`"**。**但本项目实际没这么用**（因为注解都注释掉了）。**这组配置目前处于"配了没用"的状态**（§4.8 坦白点）。

### 4.3 `RedisUtils`：全项目的 Redis 总闸

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java`）

**它是"工具类 + 静态注入"写法的又一例**：

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)                 // :35  ★私有构造，纯静态工具类
public class RedisUtils {

    private static final RedissonClient CLIENT = SpringUtils.getBean(RedissonClient.class);  // :39  ★静态注入

    public static RedissonClient getClient() { return CLIENT; }  // :63  ★暴露客户端（给需要底层 API 的地方）
    ...
}
```

**读懂 `:39`**：**为什么能"静态字段 = 从容器取 Bean"？** 因为 `RedisUtils` 在 `ruoyi-common`，**它不能依赖 `ruoyi-framework` 的 `RedisConfig`**（依赖方向：common ← framework）。**所以用 `SpringUtils.getBean` 在"类加载时"从容器里取**——**这样 common 里的任何静态方法都能用 Redis，而不需要注入。**

**`RedisUtils` 提供的能力（按数据类型分组）**：

| 类别 | 方法（行号） | 底层 Redisson 对象 |
|---|---|---|
| **普通对象** | `setCacheObject` `:103`、`getCacheObject` `:185`、`deleteObject` `:206`、`expire` `:163`、`getTimeToLive` `:196`、`hasKey` `:440` | `RBucket` |
| **批量删除** | `deleteObject(Collection)` `:215` | `RBatch`（批量，减少网络往返） |
| **List** | `setCacheList` `:230`、`getCacheList` `:254` | `RList` |
| **Set** | `setCacheSet` `:266`、`getCacheSet` `:290` | `RSet` |
| **Map（含 Hash 字段）** | `setCacheMap` `:301`、`getCacheMap` `:327`、`setCacheMapValue` `:339`、`getCacheMapValue` `:351` | `RMap` |
| **原子数** | `setAtomicValue` `:386`、`getAtomicValue` `:397`、`incrAtomicValue` `:408`、`decrAtomicValue` `:419` | `RAtomicLong` |
| **发布订阅** | `publish` `:74`/`:80`、`subscribe` `:92` | `RTopic` |
| **按前缀找 key** | `keys(pattern)` `:430` | `RKeys` |
| **限流** | `rateLimiter` `:50` | `RRateLimiter` |

**三个值得注意的细节**：

1. **`setCacheObject(key, value)`（`:103`）没有 TTL**——**写进去就是永久的**（除非显式 `expire` 或 `setCacheObject(key, value, Duration)`）。**字典/参数就是用这个"永久缓存"**（§4.5）。
2. **`setCacheObject(key, value, isSaveTtl)`（`:115`）**：**"保留原 TTL"的更新**——`try setAndKeepTTL`（Redis 6+），**失败则降级为"读出剩余 TTL → 重新 set → 重新 expire"**（`:120-124`）。**这是为了兼容 Redis 5.x 的写法**（注释 `:113` 明说了 "Redis 6.X 以上使用 setAndKeepTTL 兼容 5.X 方案"）。
3. **`deleteObject(Collection)`（`:215`）用 `RBatch`**：**批量删除走"批处理"，一次网络往返删一堆**，比 for 循环 `delete` 快得多。**清空一整类缓存（比如"所有字典"）就靠它。**

### 4.4 系统参数的缓存：cache-aside 模式

（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysConfigServiceImpl.java`）

**读：先查缓存，miss 才查库，查到了回写缓存**

```java
@Override
public String selectConfigByKey(String configKey) {                          // :83
    String configValue = RedisUtils.getCacheObject(getCacheKey(configKey));  // :84  ★① 先查缓存
    if (StringUtils.isNotEmpty(configValue)) {
        return configValue;                                                  // :86  ★② 命中，直接返回
    }
    SysConfig retConfig = baseMapper.selectOne(new LambdaQueryWrapper<SysConfig>()
        .eq(SysConfig::getConfigKey, configKey));                            // :88  ★③ 未命中，查库
    if (ObjectUtil.isNotNull(retConfig)) {
        RedisUtils.setCacheObject(getCacheKey(configKey), retConfig.getConfigValue());  // :91  ★④ 回写缓存
        return retConfig.getConfigValue();
    }
    return StringUtils.EMPTY;                                                // :94  ★⑤ 库里也没有，返回空串
}
```

**逐行读这段"读缓存"代码**（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysConfigServiceImpl.java:83-95`）：

| 行号 | 这一行在干什么 | 为什么这么写 |
|---|---|---|
| `:83` | 方法签名：按 `configKey` 取参数值 | 调用方只关心"给我键，我拿值"，不关心值来自缓存还是库 |
| `:84` | `RedisUtils.getCacheObject(getCacheKey(configKey))` | 第一站永远是 Redis；`getCacheKey` 在 `:249-251` 拼上 `sys_config:` 前缀 |
| `:85-87` | 值非空就直接 `return` | 命中缓存，整个请求不碰数据库——这就是缓存提速的核心 |
| `:88-89` | `baseMapper.selectOne(...eq(configKey...))` | 未命中才查库；`eq` 保证只按这个键查一条 |
| `:90-93` | 查到后先 `setCacheObject` 回写，再返回 | 下一次同样的请求就能命中；回写放在返回前，保证"读过就留下" |
| `:94` | 库里也没有，返回 `StringUtils.EMPTY` | 注意：**空串不写缓存**，所以不存在的键每次都会查库（§2.3 的穿透风险） |

**一个容易忽略的细节**：`:85` 用的是 `StringUtils.isNotEmpty`，所以**如果某个参数的值本来就是空串，也会被当成"没命中"**、每次去查库。参数值允许为空时，这一点要特别小心。

**这就是标准的 cache-aside（旁路缓存）**：**"读的时候顺手填缓存"**。**注意第 ⑤ 步：库里也没有时返回 `EMPTY`，不写缓存**——**所以"不存在的 key"每次都会打到库（缓存穿透，§2.3）。**

**写：更新数据库的同时"覆盖/删除"缓存**

```java
public int insertConfig(SysConfig config) {                                  // :136
    int row = baseMapper.insert(config);                                     // :137  ★先写库
    if (row > 0) {
        RedisUtils.setCacheObject(getCacheKey(config.getConfigKey()), config.getConfigValue());  // :139  ★成功才更新缓存
    }
    return row;
}

public int updateConfig(SysConfig config) {                                  // :151
    int row = 0;
    if (config.getConfigId() != null) {
        row = baseMapper.updateById(config);                                 // :154  ★按主键更新
    } else {
        row = baseMapper.update(config, new LambdaQueryWrapper<SysConfig>()
            .eq(SysConfig::getConfigKey, config.getConfigKey()));            // :156-157  ★没有主键时按 key 更新
    }
    if (row > 0) {
        RedisUtils.setCacheObject(getCacheKey(config.getConfigKey()), config.getConfigValue());  // :160
    }
    return row;
}

public void deleteConfigByIds(Long[] configIds) {                            // :171
    for (Long configId : configIds) {
        SysConfig config = selectConfigById(configId);
        if (StringUtils.equals(UserConstants.YES, config.getConfigType())) {
            throw new ServiceException("内置参数不能删除");                   // :175  ★内置参数保护
        }
        RedisUtils.deleteObject(getCacheKey(config.getConfigKey()));         // :177  ★删缓存（不是覆盖）
    }
    baseMapper.deleteBatchIds(Arrays.asList(configIds));                     // :179
}
```

**读懂"为什么修改用 `set`、删除用 `delete`"**：

- **修改**：**"新值"是有的，直接覆盖缓存**（比删掉更高效——**不用等下次读再回填**）；
- **删除**：**"新值"不存在了，只能把缓存删掉**——**下次读自然 miss、发现库也没了、返回空**。

**缓存 key 的拼法**：

```java
private String getCacheKey(String configKey) {                               // :249
    return Constants.SYS_CONFIG_KEY + configKey;                             // :250
}
```

**`Constants.SYS_CONFIG_KEY = "sys_config:"`（`Constants.java:141`）**——**所以"验证码开关"的缓存 key 是 `sys_config:sys.account.captchaOnOff`。**

**"前缀 + 业务键"是缓存 key 的经典命名法**：**前缀用来"分类"**（便于批量操作），**业务键保证唯一**。

**这个前缀的实战价值**，看"清空所有参数缓存"：

```java
public void clearConfigCache() {                                             // :197
    Collection<String> keys = RedisUtils.keys(Constants.SYS_CONFIG_KEY + "*");  // :198  ★按前缀找全部
    RedisUtils.deleteObject(keys);                                           // :199  ★批量删
}
```

**一句 `sys_config:*` 就能把所有参数缓存一网打尽。** 如果 key 没有统一前缀，**你根本不知道哪些 key 是参数缓存**——**这就是"前缀命名"的意义。**

### 4.5 字典的缓存：一模一样的套路

（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysDictTypeServiceImpl.java`）

```java
@Override
public List<SysDictData> selectDictDataByType(String dictType) {              // :100
    List<SysDictData> dictDatas = RedisUtils.getCacheObject(getCacheKey(dictType));  // :101  ★先查缓存
    if (CollUtil.isNotEmpty(dictDatas)) {
        return dictDatas;                                                    // :103  命中直接返回
    }
    dictDatas = dictDataMapper.selectDictDataByType(dictType);                // :105  查库
    if (CollUtil.isNotEmpty(dictDatas)) {
        RedisUtils.setCacheObject(getCacheKey(dictType), dictDatas);          // :107  回写缓存
        return dictDatas;
    }
    return null;
}
```

**和参数缓存是同一个模子**——**"先查 Redis → miss 查库 → 回写"**。**区别只在：参数缓存的是"一个字符串"，字典缓存的是"一个 List"。**

**清空字典缓存**：

```java
public void clearDictCache() {                                               // :174
    Collection<String> keys = RedisUtils.keys(Constants.SYS_DICT_KEY + "*");  // :175  SYS_DICT_KEY="sys_dict:"
    RedisUtils.deleteObject(keys);                                           // :176
}
```

**注意一个细节**：`SysDictTypeServiceImpl.deleteDictTypeByIds`（`:141`）**删除字典类型时，会检查"是否还有字典数据引用它"**（`:144-147`），**有则抛异常阻止删除**——**这是"数据完整性校验"（外键约束的软件实现）**，和缓存无关，但**在同一段代码里，值得注意。**

### 4.6 启动预热：`SystemApplicationRunner`

**为什么需要"预热"？** 因为**"缓存冷启动"**：**应用刚启动时 Redis 是空的，第一个用户访问会经历"查库 + 写缓存"的慢过程**。**如果某个接口在启动瞬间被高并发打进来，可能瞬间打垮数据库。**

**预热 = 应用启动时，主动把所有字典/参数加载进 Redis**，让**"第一个用户"直接命中缓存**。

（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/runner/SystemApplicationRunner.java`）

```java
@Slf4j
@Component
public class SystemApplicationRunner implements ApplicationRunner {          // :33  ★实现 ApplicationRunner

    @Override
    public void run(ApplicationArguments args) throws Exception {            // :40  ★启动后自动执行一次
        if (ruoyiConfig.isCacheLazy()) {                                     // :41  ★"懒加载"开关
            return;                                                          //     配了懒加载就跳过预热
        }
        configService.loadingConfigCache();                                  // :44  ★加载全部参数进缓存
        log.info("加载参数缓存数据成功");
        dictTypeService.loadingDictCache();                                  // :46  ★加载全部字典进缓存
        log.info("加载字典缓存数据成功");
    }
}
```

**`ApplicationRunner`** 是 Spring Boot 的钩子：**应用启动完成后自动执行 `run()`**——**是"启动初始化"的标准位置。**

**`loadingConfigCache`（`SysConfigServiceImpl.java:186`）** 做的是：**把 `sys_config` 全表查出来，逐条 `setCacheObject`**：

```java
public void loadingConfigCache() {                                           // :186
    List<SysConfig> configsList = selectConfigList(new SysConfig());          // :187  查全部
    for (SysConfig config : configsList) {
        RedisUtils.setCacheObject(getCacheKey(config.getConfigKey()), config.getConfigValue());  // :189
    }
}
```

**`loadingDictCache`（`SysDictTypeServiceImpl.java:157`）** 稍微讲究一点：**查出全部字典数据 → 按 `dictType` 分组 → 每组按 `dictSort` 排序 → 存进去**：

```java
public void loadingDictCache() {                                             // :157
    List<SysDictData> dictDataList = dictDataMapper.selectList(
        new LambdaQueryWrapper<SysDictData>().eq(SysDictData::getStatus, UserConstants.DICT_NORMAL));  // :158-159  只加载"正常"状态
    Map<String, List<SysDictData>> dictDataMap = dictDataList.stream()
        .collect(Collectors.groupingBy(SysDictData::getDictType));            // :160  ★按字典类型分组
    dictDataMap.forEach((k, v) -> {
        String dictKey = getCacheKey(k);
        List<SysDictData> dictList = v.stream()
            .sorted(Comparator.comparing(SysDictData::getDictSort))           // :164  ★按排序号排好
            .collect(Collectors.toList());
        RedisUtils.setCacheObject(dictKey, dictList);                         // :166
    });
}
```

**读懂两个要点**：

1. **只加载 `status = 正常` 的字典**（`:159`）——**禁用的字典项不预热**（用不着）；
2. **存进缓存前先按 `dictSort` 排序**（`:164`）——**因为前端下拉框要按这个顺序显示**。**把"排序"这一步放在"写缓存时"做一次**，**比"每次读的时候都排"高效**——**这是"把计算放到写路径"的缓存优化思路**。

**`cacheLazy` 开关**（`application.yml:14` `cacheLazy: false`）：**`false` = 启动就预热**（默认）；**`true` = 懒加载，不预热**（等第一次访问再填缓存）。**开发时可能想用懒加载**（省启动时间）。

### 4.7 缓存管理入口：手动重置

**除了启动预热，还有"手动刷新缓存"的入口**：

| 入口 | 代码位置 | 作用 |
|---|---|---|
| **参数缓存重置** | `SysConfigController.java:150` → `resetConfigCache()` | **清空 + 重新加载全部参数** |
| **字典缓存重置** | `SysDictTypeController.java:125` → `resetDictCache()` | **清空 + 重新加载全部字典** |

**`reset` 的实现**（`SysConfigServiceImpl.java:206`）：

```java
public void resetConfigCache() {                                             // :206
    clearConfigCache();                                                      // :207  ★先清空
    loadingConfigCache();                                                    // :208  ★再重新加载
}
```

**"先清空再加载"= "强制从库里拉最新"**——**给运维一个"缓存脏了就手动刷"的按钮**。**这是"最终一致"的兜底手段：平时靠"改数据时同步缓存"保证一致，万一出现诡异的不一致，点一下重置。**

### 4.8 ★ 坦白点（本章有三个）

**坦白点一：Spring Cache 配好了却完全没用**

**事实**：`RedisConfig` 有 `@EnableCaching :47`、有 `RedissonSpringCacheManager :134`、`application.yml:341-353` 配了两个 `cacheGroup`（`redissonCacheMap` / `testCache`）——**但全项目 `@Cacheable` 零使用，`@CacheEvict` 全部被注释掉**（`SysUserServiceImpl.java:406/421/448/461/473/605/624`）。

**后果**：**这整套 Spring Cache 基础设施目前是"死配置"**——`cacheManager` Bean 建了没人用、`redisson.spring` 的 `cacheGroup` 配了没生效。**真正在工作的只有 `RedisUtils` 手动缓存那条路。**

**怎么理解 / 怎么说**：**这是"技术选型变更留下的痕迹"**——**项目早期可能想用 `@Cacheable`，后来改成手动实现（大概率因为多租户 key 问题，见 §2.2），但基础设施没删干净。** **答辩时可以说**："我们的缓存实际是手写的 cache-aside，Spring Cache 的架子还在但业务没用——**如果清理，可以把 `@EnableCaching` 和 `cacheGroup` 配置一起删掉，减少误导**。"

**坦白点二：字典/参数缓存大多"没有 TTL"（永久驻留 + 穿透风险）**

**事实**：`setCacheObject(key, value)`（`RedisUtils.java:103`）**不带过期时间**，字典和参数的缓存**全走这个**（`SysConfigServiceImpl.java:91/139/160/189`、`SysDictTypeServiceImpl.java:107/166`）。**只有登录态、防重标记这些才主动设 TTL。**

**双面后果**：

- **好的一面**：**不会雪崩**（没有"大量 key 同时过期"），**也不会因为过期而反复查库**；
- **坏的一面**：① **数据一旦缓存就是"永久"**——**如果绕过"改数据 → 更新缓存"这条路径直接改了库（比如手动执行 SQL、或另一个服务改的），缓存永远不刷新**；② **`selectConfigByKey` 对"不存在的 key"不缓存（`:94`），每次都打到库**——**典型的缓存穿透**（§2.3）。

**怎么修**：**给字典/参数这类"长缓存"设一个"较长的 TTL（如 30 分钟）+ 一定的随机抖动"**——**既能自动纠偏"绕过路径改库"的情况，随机抖动又能避免集体过期**。

**坦白点三：`RedisUtils` 的"静态注入"方式，对单元测试不友好**

**事实**：`RedisUtils.java:39` 用 `SpringUtils.getBean(RedissonClient.class)` **在类加载时取 Bean**。

**后果**：**要写单元测试，必须先起 Spring 容器（或 mock 静态字段）**——**比"构造器注入"难测**。而且**如果 `RedisUtils` 在 Spring 容器"还没就绪"时被静态引用，`SpringUtils.getBean` 会拿到 null**（虽然本项目没触发）。

**怎么理解**：**这是"工具类静态化"的固有代价**——**换来的是"哪儿都能用的便利"**（不用在每个类里注入）。**本项目在 common 模块大量用这个模式**（`MessageUtils` 第 11 章也是、`SpringUtils` 本身也是），**属于"团队既定风格"，不算错误，但要意识到它的测试成本。**

### 4.9 动手验证：三分钟看见缓存

**前置条件**：应用用 dev 配置启动，Redis 在 `localhost:6379` 的 5 号库（`ruoyi-admin/src/main/resources/application-dev.yml:184-192`）；登录后拿到 token。请求头名是 `token`（`ruoyi-admin/src/main/resources/application.yml:103`），值是 `Bearer <token>`（前缀来自 `application.yml:119`，前端在 `zxcm-ui/src/utils/request.js:33` 就是这么拼的）。

**第一步：看"启动预热"有没有发生**

在启动日志里搜这两句：

```
加载参数缓存数据成功
加载字典缓存数据成功
```

它们分别来自 `ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/runner/SystemApplicationRunner.java:45` 和 `:47`。如果搜不到，检查 `ruoyi-admin/src/main/resources/application.yml:14` 的 `cacheLazy` 是不是被改成了 `true`（`true` 表示懒加载、不预热，见 `:41-43`）。

**第二步：直接看 Redis 里的键**

```bash
redis-cli -h 127.0.0.1 -p 6379
SELECT 5
KEYS sys_dict:*
GET sys_dict:sys_normal_disable
TTL sys_config:sys.user.initPassword
```

**预期**：

- `KEYS` 能看到 `sys_dict:` 开头的键；
- `GET` 出来是**可读 JSON**，不是乱码——因为 `RedisConfig` 配了 `JsonJacksonCodec.INSTANCE`（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/RedisConfig.java:69`）；
- `TTL` 返回 `-1`，表示**没有过期时间**——对应 `RedisUtils.java:103` 那个不带 TTL 的 `setCacheObject`。

**第三步：用断点确认"一次命中、一次未命中"**

在 `SysConfigServiceImpl.java:84`（读缓存）和 `:91`（回写缓存）下断点，然后连续请求两次同一个参数：

- 第一次：先停在 `:84`，`getCacheObject` 返回 `null`，继续执行后会停在 `:91`（回写）；
- 第二次：只停在 `:84`，然后从 `:86` 直接返回，**不会再停在 `:88` 查库**。

**第四步：验证"绕过写路径改库，缓存不会自动更新"（本章坦白点的现场版）**

```sql
-- 只改数据库，故意不经过 updateConfig
UPDATE sys_config SET config_value = '999999' WHERE config_key = 'sys.user.initPassword';
```

改完再执行 `redis-cli GET sys_config:sys.user.initPassword`，**预期仍是旧值**——这正是 §4.8 坦白点二说的"永久驻留"。然后调用重置接口：

```
DELETE /system/config/refreshCache
请求头：token: Bearer <你的登录 token>
```

该接口对应 `ruoyi-admin/src/main/java/com/ruoyi/web/controller/system/SysConfigController.java:148-151`，需要 `system:config:remove` 权限（`SysConfigController.java:146`）。调用成功后再 `GET`，**预期变成 `999999`**。这一步把"写时更新 + 手动重置兜底"的整条链路都验证了。

---

## 五、一个操作的完整生命流程

**场景：前端加载"新增用户"页面，需要读"启用/停用"字典 + 读"初始密码"系统参数。**

```
┌─ 0. 应用启动时（早已完成） ─────────────────────────────────┐
│  SystemApplicationRunner.run()(:40)                         │
│    ├─ configService.loadingConfigCache()(:44)  → 全部参数进 Redis │
│    └─ dictTypeService.loadingDictCache()(:46)  → 全部字典进 Redis │
│  ★ 预热完成，"sys_dict:sys_normal_disable""sys_config:sys.user.initPassword" 已在 Redis │
└──────────────────────────────────────────────────────────┘
                    ↓  （用户访问页面）
┌─ 1. 请求"字典数据" ─────────────────────────────────────────┐
│  GET /system/dict/data/type/sys_normal_disable             │
│    → SysDictDataController → SysDictTypeServiceImpl        │
│    → selectDictDataByType("sys_normal_disable")(:100)      │
│    ├─ RedisUtils.getCacheObject("sys_dict:sys_normal_disable")(:101) │
│    ├─ 命中！返回 [{0=正常},{1=停用}]（已按 dictSort 排好）  │
│    └─ ★ 全程没碰数据库                                       │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 2. 请求"初始密码"参数 ────────────────────────────────────┐
│  SysConfigServiceImpl.selectConfigByKey("sys.user.initPassword")(:83) │
│    ├─ RedisUtils.getCacheObject("sys_config:sys.user.initPassword")(:84) │
│    └─ 命中！返回 "123456"                                   │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 3. 运维在后台改了"初始密码" ──────────────────────────────┐
│  PUT /system/config                                        │
│    → SysConfigServiceImpl.updateConfig(config)(:151)       │
│    ├─ baseMapper.updateById(config)(:154)   ← 先写库        │
│    └─ row>0 → RedisUtils.setCacheObject(..., 新值)(:160)   ← 再覆盖缓存 │
│  ★ 库和缓存都是新值，下次读命中新值                          │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 4.（异常情况）缓存乱了，手动重置 ─────────────────────────┐
│  SysConfigController.refreshCache()(:148-151)              │
│    → clearConfigCache()(:197)   RedisUtils.keys("sys_config:*")(:198) → 批量删 │
│    → loadingConfigCache()(:186) 全表重查 → 重新写入 Redis    │
│  ★ "清空 + 重载" = 强制回到"库的真相"                        │
└──────────────────────────────────────────────────────────┘
```

**这个流程告诉你三件事**：

| 阶段 | 动作 | 一致性策略 |
|---|---|---|
| **冷启动** | 预热（`ApplicationRunner`） | 启动时**主动**填缓存 |
| **正常读** | cache-aside（`select...`） | **读时顺带填**（miss 才查库） |
| **写** | **先写库、再更新缓存** | **"写库 + 改缓存"，尽力保一致** |
| **兜底** | `resetConfigCache` | **手动"清空+重载"** |

**★ 关键认知**：**缓存的"一致性"是"尽力而为"，不是"强保证"**。**本项目用"写时同步更新"来逼近一致，用"手动重置"做最后的兜底**——**这是"读多写少、能容忍短暂不一致"场景下的合理选择。** **如果哪天出现"改了参数但缓存还是旧的"，第一反应就该是"写库那条路径有没有漏掉更新缓存"，或者"直接点一下重置"。**

---

## 六、面试官高频问题 + 答案

**Q1：你们项目缓存是怎么做的？用了 Spring Cache 吗？**

**A**：**机制上我们配了 Spring Cache**（`@EnableCaching` + `RedissonSpringCacheManager`，`RedisConfig.java:47/134`），**但业务代码实际没用 `@Cacheable`**——**全项目的缓存注解要么零使用、要么被注释掉了**（`SysUserServiceImpl` 里那 7 处 `@CacheEvict`）。
**真正在用的是"手动 cache-aside"**：`RedisUtils` 封装了 Redisson，字典和系统参数在 Service 里手写"先查 Redis → miss 查库 → 回写"。**为什么放弃 `@Cacheable`？最可能的原因是多租户**——`@Cacheable` 的 key 默认只拼方法参数，**在多租户下会导致"跨租户串缓存"**，要处理就得自定义 key 生成器，还不如手写可控。

> **追问：既然业务不用 `@Cacheable`，为什么还留着 `@EnableCaching`？**
> **一句话答**：架子留着可以随时启用，但当前确实没用；答辩时主动说"这是历史选型残留，清理时应连同 `cacheGroup` 配置一起删"，比硬说"我们在用"更可信。

**Q2：为什么用 Redisson 不用 `RedisTemplate`？**

**A**：**Redisson 把分布式场景封装成了对象**——`RLock`（分布式锁）、`RRateLimiter`（限流）、`RMap`/`RSet`/`RAtomicLong`（分布式集合）。**用 `RedisTemplate` 你得自己写 Lua 脚本、自己处理锁续期**。**我们项目里两者并存**：业务用 Redisson（`RedisUtils`），**Redis 监控页面用 `RedisTemplate`**（`CacheController.java:42`）。

**Q3：字典和系统参数为什么要缓存？**

**A**：**它们是最典型的"读多写少 + 全站共享 + 不一致后果轻"的数据**。字典（码值映射）几乎每个页面都读、几个月才改一次；参数（开关/配置）被运行时频繁读取。**直接查库会浪费大量重复查询**。**而且它们数据量小、全站一样**——**这是"能共享缓存"的前提**（对比"我的订单"这类"因人而异"的数据，key 必须带用户维度）。

**Q4：缓存的 key 怎么设计？为什么要加前缀？**

**A**：用 **"前缀 + 业务键"**——参数是 `sys_config:{configKey}`（`SysConfigServiceImpl.java:250`，前缀 `Constants.SYS_CONFIG_KEY="sys_config:"`），字典是 `sys_dict:{dictType}`。
**加前缀是因为"批量操作"**：清空参数缓存时 `RedisUtils.keys("sys_config:*")`（`:198`）一句就能找出所有参数缓存。**没有统一前缀，你根本不知道哪些 key 属于同一类。**

**Q5：缓存和数据库怎么保持一致？**

**A**：**写数据时同步更新缓存**：新增/修改时 `setCacheObject` 覆盖（`:139`/`:160`），删除时 `deleteObject`（`:177`）。**修改用"覆盖"、删除用"删"，是因为修改的新值是现成的、覆盖更高效；删除后新值不存在、只能删缓存让下次读回填。**
**兜底**：提供 `resetConfigCache`（`:206` "清空 + 重载"）给运维手动刷。**但要承认：这是"尽力而为"的最终一致，不是强一致**——缓存天然如此。

**Q6：缓存穿透、击穿、雪崩你们怎么处理的？**

**A**：**坦白说我们处理得比较朴素**：
- **穿透**：查不到的 key **返回空串、不缓存**（`SysConfigServiceImpl.java:94`）——**严格说这会穿透**，改善方法是"缓存空值或加布隆过滤器"；
- **击穿**：**没做特殊处理**（热点 key 过期瞬间的互斥重建）——**因为字典/参数更新极少、数据量小，实际风险低**；
- **雪崩**：**我们的字典/参数缓存大多不设 TTL**（`setCacheObject(key,value)`，`RedisUtils.java:103`）——**好处是不会集体过期、坏处是数据永久驻留**。
**所以准确说法是：我们用"数据量小、更新极少"这个前提，绕开了大部分缓存问题**；**换成高并发场景，这套朴素做法不够用**。

> **追问：那为什么不上布隆过滤器？**
> **一句话答**：本项目的缓存键是"字典类型/参数键"这类有限枚举，不是海量用户输入，穿透面很小；布隆过滤器还要额外维护同步，性价比低。**方案要跟数据规模匹配。**

**Q7：`SystemApplicationRunner` 是干什么的？**

**A**：**缓存预热**。它实现 `ApplicationRunner`（`:33`），**应用启动完成后自动执行**，把全部系统参数和字典**主动加载进 Redis**（`:44`/`:46`）——**避免"第一个用户"经历"查库+回填"的冷启动慢**，特别是防止启动瞬间高并发打垮数据库。**有个 `cacheLazy` 开关**（`RuoYiConfig.java:60`，配置 `application.yml:14`）：`false` 就预热，`true` 就懒加载。

**Q8：`RedisUtils` 为什么用 `SpringUtils.getBean` 静态注入？**

**A**：因为 **`RedisUtils` 在 `ruoyi-common`，不能依赖 `ruoyi-framework` 的 `RedisConfig`**（依赖方向是 common ← framework）。**要在这个"静态工具类"里拿到容器里的 `RedissonClient`，就只能用 `SpringUtils.getBean`**（`RedisUtils.java:39`）。
**好处是"哪儿都能用"**（不用每个类都注入）；**代价是单元测试不友好**（必须先起 Spring 容器）、**以及有"容器未就绪时取到 null"的潜在风险**。**这是本项目 common 模块的既定风格**（`MessageUtils` 也一样）。

**Q9：`@Primary` 和 `destroyMethod` 在 `RedisConfig` 里起什么作用？**

**A**：**`@Primary`（`:59`）**——如果容器里有多个 `RedissonClient`，**"不指定名字注入"时用这个**（本项目只有一个，属于"声明式保险"）。**`destroyMethod="shutdown"`（`:60`）**——**应用关闭时 Spring 自动调 `RedissonClient.shutdown()`**，**优雅释放连接池，避免资源泄漏**。**资源型的 Bean 都应该有这种"收尾"。**

**Q10：如果想给字典缓存加过期时间，你改哪里？**

**A**：把 `SysDictTypeServiceImpl` 里回写缓存那行（`setCacheObject(dictKey, dictList)` `:107`/`:166`）**改成带 `Duration` 的重载** `setCacheObject(key, value, Duration.ofMinutes(30))`（`RedisUtils.java:137`）。**同时建议 TTL 加随机抖动**（比如 30±5 分钟），**避免大量字典在同一时刻集体过期**。**改完就同时解决了"永久驻留"和"雪崩"两个隐患。**

### 本章答辩卡（3 组追问级问答）

**卡 1｜问：你们用"按前缀找 key + 批量删"清空缓存，这在生产环境有什么隐患？**

- **10 秒答案**：`RedisUtils.keys(pattern)` 底层是 `RKeys.getKeysByPattern`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java:430-433`），本质上是遍历匹配键。Redis 单线程执行这类全库扫描时，键一多会阻塞其他命令，`resetConfigCache`（`SysConfigServiceImpl.java:206-209`）就可能造成一次明显卡顿。数据量大时应改用 `SCAN` 游标分批找、分批删，别用一次性全量匹配。
- **证据**：`RedisUtils.java:430-433`、`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysConfigServiceImpl.java:197-200`、`:206-209`。

**卡 2｜问：如果某类字典以后要"按租户不同"，现在的缓存键会不会串数据？**

- **10 秒答案**：会。当前字典键是 `sys_dict:{dictType}`，只拼了类型、没有租户（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysDictTypeServiceImpl.java:311-313`）。一旦同名字典在不同租户下内容不同，A 租户会读到 B 租户的缓存。改法是拼上 `tenantId`：`sys_dict:{tenantId}:{dictType}`，并在数据变更时按这个键失效。
- **证据**：键的拼法在 `SysDictTypeServiceImpl.java:311-313`；项目里现成的租户来源是 `RuoYiConfig.getProfile()` 中的 `LoginHelper.getTenantId()`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/config/RuoYiConfig.java:79-82`）。

**卡 3｜问：如果 Redis 挂了，你们的参数读取会怎样？**

- **10 秒答案**：会失败，而不是自动降级查库。`selectConfigByKey` 第一行就调 `RedisUtils.getCacheObject`（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysConfigServiceImpl.java:84`），而 `RedisUtils` 内部直接操作 `RedissonClient`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java:185-188`），没有 try-catch 兜底。改进是包一层"缓存异常就查库并记日志"，让 Redis 故障只降速、不阻断业务。
- **证据**：`SysConfigServiceImpl.java:84`、`RedisUtils.java:185-188`。

---

## 七、选型对比

### 7.1 Redis 客户端选型

| 方案 | 分布式锁/限流 | 学习成本 | 依赖体积 | 本项目 |
|---|---|---|---|---|
| **Jedis** | 要自己实现 | 低 | 小 | ✗ |
| **`RedisTemplate`（Spring Data）** | 要自己实现（Lua） | 中 | 中 | **✔（仅监控页用）** |
| **Redisson（本项目主力）** | **原生封装** | 中 | **大** | **✔** |
| Lettuce | 要自己实现 | 中 | 中 | ✗ |

**结论**：**需要"分布式锁/限流/分布式集合"时，Redisson 省下的自研成本远大于它的依赖体积。** **本项目两个都用**（业务 Redisson、监控 `RedisTemplate`）——**"用合适的工具做合适的事"。**

### 7.2 缓存实现方式

| 方式 | 侵入性 | 多租户/key 控制 | 可读性 | 本项目 |
|---|---|---|---|---|
| **`@Cacheable` 注解** | **低**（一行注解） | **弱**（key 默认只拼参数） | 好（但隐式） | **✗（配了没启用）** |
| **手动 `RedisUtils`（本项目）** | 中（要写几行） | **强**（key 自己拼） | **好（显式）** | **✔** |
| 中间件层（如缓存代理） | 无 | 弱 | 不涉及 | ✗ |

**结论**：**`@Cacheable` 适合"简单、单租户、key 明确"；多租户 / 需要精细失效控制时，手写更可控。** **本项目选了"手动但清晰"**——**这是"复杂场景下牺牲便利换可控性"的典型权衡。**

### 7.3 缓存位置

| 位置 | 速度 | 多实例共享 | 容量 | 本项目 |
|---|---|---|---|---|
| **本地缓存（Caffeine/Map）** | **最快（纳秒）** | **✗（各实例一份）** | 受 JVM 内存限制 | ✗ |
| **Redis（本项目）** | 快（毫秒级、跨网络） | **✔** | 大 | **✔** |
| 本地 + Redis 两级 | **兼顾** | 部分（本地层不共享） | 大 | ✗ |

**结论**：**分布式系统里"跨实例共享"是硬需求，所以主缓存必须放 Redis。** **`@Cacheable` 也曾支持本地缓存（Caffeine），本项目没用。** **两级缓存（本地 + Redis）能再快一个数量级，但要处理"本地层失效"的同步问题**（通常靠 Redis 发布订阅广播，本项目 `RedisUtils.publish/subscribe :74/:92` 就是干这个的底座，但没用于此）。

---

## 延伸阅读（网上优秀资料）

- **Redisson 官方 Wiki**：<https://github.com/redisson/redisson/wiki> —— **分布式对象总览**（`RBucket`/`RMap`/`RLock`/`RRateLimiter` 全在这），本项目 `RedisUtils` 的底层。
- **Redisson · Spring Cache 集成**：<https://github.com/redisson/redisson/wiki/14.-Integration-with-frameworks#141-spring-cache> —— `RedissonSpringCacheManager` 的官方用法（本项目 `RedisConfig.java:134`）。
- **Spring Cache 抽象**：Spring 官方文档搜「Cache Abstraction / @Cacheable」—— 理解 `@Cacheable`/`@CacheEvict`/`CacheManager` 的机制。
- **缓存三大问题（穿透/击穿/雪崩）**：搜「缓存穿透 缓存击穿 缓存雪崩 解决方案」—— **面试必考，务必能各说清"是什么 + 怎么解"**。
- **cache-aside vs write-through vs write-behind**：搜「缓存更新策略 Cache-Aside Write-Through」—— **理解本项目用的是哪一套、为什么**（本项目 = cache-aside + 写时覆盖）。
- **Spring Boot · `ApplicationRunner` vs `CommandLineRunner`**：Spring Boot 官方文档 —— **启动初始化的标准做法**（本项目 `SystemApplicationRunner`）。
- **RuoYi-Vue-Plus 官方文档 · 缓存与 Redisson**：搜「RuoYi-Vue-Plus 缓存 Redisson」—— **对照官方怎么讲这套设计**（本仓库即其 4.x 血统）。

---

## 本章小结 & 下一步

- **三块缓存**：**Redis 基础设施（`RedisConfig` + `RedisUtils`，全站在用）**、**Spring Cache（配了没启用）**、**业务缓存（字典/参数手动实现，在用）**。
- **`RedisConfig`**：`@Primary :59` + `destroyMethod="shutdown" :60` + `JsonJacksonCodec :69`；单机/集群二选一（`:72`/`:90`）；`cacheManager :126` 把 Redisson 接给 Spring Cache。
- **`RedisUtils`**：**静态注入 `SpringUtils.getBean(RedissonClient.class) :39`**（common 不依赖 framework）；封装 `RBucket`/`RList`/`RSet`/`RMap`/`RAtomicLong`/`RTopic`/`RKeys`；**`keys(pattern) :430` + 批量删 `:215`** 是"清空一整类缓存"的组合拳。
- **cache-aside 模式**：**读**——先查缓存（`SysConfigServiceImpl.java:84`）、miss 查库（`:88`）、回写（`:91`）；**写**——先写库（`:137`/`:154`）、成功才改缓存（`:139`/`:160`），**改覆盖、删删除**（`:177`）；**key = 前缀 + 业务键**（`:250`，`sys_config:`/`sys_dict:`）。
- **启动预热**：`SystemApplicationRunner`（`ApplicationRunner :33`）→ `loadingConfigCache :44` + `loadingDictCache :46`；`cacheLazy` 开关（`application.yml:14`）；字典预热时**只加载正常项 + 预先按 `dictSort` 排序**（`:158-164`）。
- **兜底**：`resetConfigCache :206`（清空 + 重载），入口 `SysConfigController.java:150` / `SysDictTypeController.java:125`。
- **★ 关键认知**：**"缓存能否共享"取决于 key 的维度**——字典/参数"全站一样"可共享；`LoginUser`"因人而异"，key 必须带 `userId`。**"能缓存"的三个条件：读多写少、全站共享、可容忍短暂不一致。**
- **坦白点**：① **Spring Cache 全项目未启用**（`@CacheEvict` 全被注释，`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysUserServiceImpl.java:406`/`:421`/`:448`/`:461`/`:473`/`:605`/`:624`），是"选型变更留下的死配置"；② **字典/参数缓存大多无 TTL**（`RedisUtils.java:103`）——**不会雪崩，但会永久驻留 + 存在穿透（`:94` 不缓存空值）**；③ **`RedisUtils` 静态注入对单元测试不友好**。
- **可迁移的思维**：**"场景决定方案"**（数据量小就别搞复杂 TTL 策略）、**"把计算挪到写路径"**（缓存前排序）、**"用前缀给 key 分类"**（便于批量操作）、**"一致性是尽力而为 + 兜底"**。

**下一章（第 13 章 · 文件上传与 Excel 导入导出）**：本章讲"数据放 Redis 少查库"，**第 13 章讲"文件怎么进来、怎么出去"**——上传的文件存在哪（本地 / OSS？）、文件名怎么防重（雪花 ID）、`MultipartFile` 的坑（临时文件、大小限制）、**Excel 导入导出怎么做到"一个注解搞定"**（`@Excel` + EasyExcel），以及**大文件上传/导出的内存问题**。
