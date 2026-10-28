# 附录 A · 基座横切索引

> **这是一页速查表，不是新内容。** 第 0~15 章是按「基座的一块块能力」纵向切的；但有些能力是**横着穿过所有章节**的——Sa-Token、数据权限、MyBatis-Plus 插件链、Redis、异步、日志……
>
> **本附录解决三个问题**：
> ① 当你想「再看一眼数据权限是怎么做的」或「线程池到底在哪章讲透的」，**不用翻十六份文件**，查这里；
> ② 答辩前，把**全卷 46 处「坦白点」**（项目没做好的地方）**一页过完**，心里有底；
> ③ 遇到「我要给系统加某个能力」，**速查「该看哪章、该改哪个文件」**。
>
> **使用约定**：所有定位都写成「第 X 章 §Y.Z」或「项目根相对全路径`:行号`」，可直接跳转。
> **除第 15 章明确标注「待新建」的示例文件外，所有行号都是答案册 `E:\Develop\project\zhaoxin\zhaoxin-contract\` 下真实存在的**——本卷与网上教程最大的区别，就是这一条。

---

## 一、基座能力 → 第一次讲透 / 各章回声 / 代码落点

> **读法**：「第一次讲透」是你**该去精读**的那一节；「各章回声」是同一能力在后面章节的**再次出现**（通常只讲一句 + 指路）。**同一个能力只深讲一次，其余给指针**——这是本套文档的组织原则。

| 基座能力 | **第一次讲透** | 各章回声（再次出现） | 真实代码落点 |
|---|---|---|---|
| **模块划分 / 依赖方向** | 第 0 章 §2.1、§4.2 | 第 15 章 §4.3（加模块） | `pom.xml:323-336`、`ruoyi-base/pom.xml:20-26` |
| **启动装配 / `@ComponentScan`** | 第 0 章 §4.3 | 第 15 章 §4.3（加模块必须让 admin 依赖） | `ruoyi-admin/src/main/java/com/ruoyi/RuoYiApplication.java:31-50` |
| **「框架 vs 业务」边界判定** | 第 0 章 §4.5 | 第 15 章全章（在业务侧落地） | 每个 `com.ruoyi.*` 包名 |
| **RBAC 权限模型** | 第 1 章 §1~§3 | 第 3 章（鉴权）、第 15 章 §4.2（菜单权限码） | `sys_user`/`sys_role`/`sys_menu`/`sys_role_menu` |
| **菜单三合一（M/C/F）** | 第 1 章 | 第 15 章 §4.2（插 `sys_menu`） | `sys_menu` 表结构 `script/sql/zxcm_init.sql:3988` |
| **认证链路（登录）** | **第 2 章全章** | 第 5 章（登录与会话） | `SysLoginService`、`LoginHelper`、`PlusSaTokenDao` |
| **`LoginUser` 登录态载体** | 第 2 章 | 第 3 章（取权限码）、第 4 章（取部门/用户 id） | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/model/LoginUser.java:33` |
| **鉴权：路径级 + 注解级两道拦截器** | **第 3 章 §4** | 第 15 章 §4.10（`@SaCheckPermission`） | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/SaTokenConfig.java:55`、`:79` |
| **权限码「声明在代码、授权在库」** | 第 1 章、第 3 章 | 第 15 章 §4.10 | `@SaCheckPermission` 遍布 Controller |
| **数据权限 `@DataPermission`** | **第 4 章全章** | 第 15 章 §4.7（新表怎么挂） | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/DataPermission.java` |
| **数据权限「无痕改写 SQL」** | 第 4 章 §4 | 第 15 章 §4.7 | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/handler/PlusDataPermissionHandler.java:132` |
| **数据权限双轨（标准轨 + 定制轨）** | 第 4 章 §2 | 第 15 章 §4.7（用 `stampPermission`） | `DataScopeType.java`（标准）、`ContractPermission.java`（定制）、`DataPermissionResource.java` |
| **多点登录 / 同端互斥** | **第 5 章全章** | —— | `application.yml:101-125`（`is-concurrent`/`is-share`） |
| **在线用户管理** | 第 5 章 §4 | —— | `SysUserOnlineController`、`UserActionListener` |
| **多租户隔离** | **第 6 章全章** | 第 7 章（MP 租户插件）、第 15 章 §4.13（自测项 11） | `ruoyi-tenant` 模块、`TenantProperties.java` |
| **ORM：MyBatis-Plus 插件链** | **第 7 章全章** | 第 4 章（数据权限插件是其中一环） | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/MybatisPlusConfig.java:44-57` |
| **逻辑删除 `@TableLogic`** | 第 7 章 §4 | 第 15 章 §4.4/§4.9 | `application.yml` `global-config` + `@TableLogic` 字段 |
| **自动填充（审计字段）** | 第 7 章 §4 | 第 15 章 §4.4/§4.13 | `MybatisPlusMetaObjectHandler`；`BaseEntity.java:48/55/62/69` |
| **雪花 ID / 主键策略** | 第 7 章 §4 | 第 15 章 §4.9（insert 回填主键） | `MybatisPlusConfig`、`CustomIdGenerator` |
| **分页 / `PageQuery` / `TableDataInfo`** | 第 7 章 §4、第 11 章 §4 | 第 15 章 §4.8 | `PageQuery.java`、`TableDataInfo.java` |
| **操作日志 `@Log` + `LogAspect`** | **第 8 章全章** | 第 15 章 §4.10（新接口挂日志） | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/Log.java` |
| **登录日志 / 在线动作** | 第 8 章、第 5 章 | —— | `sys_logininfor`、`UserActionListener` |
| **防重提交 `@RepeatSubmit`** | **第 9 章 §2** | 第 15 章 §4.10 | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/RepeatSubmit.java` |
| **限流 `@RateLimiter`** | 第 9 章 §2.7（★本项目未启用） | —— | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/RateLimiter.java`、`RateLimiterAspect` |
| **脱敏 `@Sensitive` + SPI** | **第 10 章 §2~§4** | 第 15 章 §4.14 | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/Sensitive.java`、`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/service/SensitiveService.java` |
| **XSS 过滤 `XssFilter`** | 第 10 章 §4 | 第 15 章 §4.14 | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/XssFilter.java:30`（注册于 `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/FilterConfig.java:47`） |
| **可重复读请求体 `RepeatableFilter`** | 第 10 章 §4 | —— | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/RepeatableFilter.java:28` |
| **统一返回 `R<T>`** | **第 11 章 §2** | 全书每个 Controller、第 15 章 §4.10（`toAjax`） | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/R.java:37/42` |
| **全局异常处理** | **第 11 章 §4** | 第 15 章 §4.9（抛 `ServiceException`） | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/web/exception/GlobalExceptionHandler.java` |
| **国际化（`MessageUtils`）** | 第 11 章 §4 | —— | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/MessageUtils.java:28` |
| **Redis 客户端（Redisson）** | **第 12 章 §4.1** | 第 9 章（防重用 Redis）、第 3 章（发号缓存） | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/RedisConfig.java` |
| **`RedisUtils` 总闸** | 第 12 章 §4.2 | 第 5 章（在线用户）、第 8 章 | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java:37` |
| **Spring Cache（★配了没用）** | 第 12 章 §4.8（坦白点一） | —— | `RedisConfig.java:47`（`@EnableCaching`）、`application.yml:341`（redisson 缓存组） |
| **字典机制（`sys_dict_*`）** | **第 12 章 §4** | 第 15 章 §4.2/§4.6（`ExcelDictConvert`） | `SysDictTypeServiceImpl`（`implements DictService`）、`DictService.java` |
| **系统参数（`sys_config`）** | 第 12 章 §4 | —— | `SysConfigServiceImpl`、`SystemApplicationRunner`（预热） |
| **文件上传（本地盘）** | **第 13 章 §4** | 第 15 章（新模块若要传附件） | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/FileUploadUtils.java` |
| **Excel 导入导出（EasyExcel）** | **第 13 章 §4** | 第 15 章 §4.6/§4.10 | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/poi/ExcelUtil.java`、`ExcelDictConvert.java` |
| **WebSocket 推送** | **第 14 章 §4** | 第 15 章 §4.14 | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketServer.java` |
| **异步 `@Async`** | 第 14 章 §4 | —— | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/AsyncConfig.java:35/48` |
| **线程池 / 优雅停机** | 第 14 章 §4 | —— | `ThreadPoolConfig.java`、`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/Threads.java:49` |
| **定时任务（XXL-Job）** | 第 14 章 §4/§7 | —— | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/XxlJobConfig.java` |
| **静态注入 `SpringUtils.getBean`** | 第 10 章 §2（`RedisUtils` 引子）、第 12 章 §4.2 | 第 13 章（`ExcelDictConvert` 取 `DictService`） | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/spring/SpringUtils.java` |

---

## 二、注解速查表

> 基座提供的**全部自定义注解**，一张表看完。左列注解 → 定义位置 → 它干什么 → 第一次讲透在哪 → 本项目用了没。

| 注解 | 定义位置（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/`） | 作用 | 讲在哪章 | 本项目 |
|---|---|---|---|---|
| **`@SaCheckPermission`** | （Sa-Token 提供，非基座定义） | 接口鉴权：比对权限码 | 第 3 章 §4 | **遍布 Controller** |
| **`@DataPermission`** | `annotation/DataPermission.java` | 数据权限组（行级过滤） | 第 4 章、第 15 章 §4.7 | **业务 Mapper 多处** |
| **`@DataColumn`** | `annotation/DataColumn.java` | 数据权限「占位符 → 真实列」映射 | 第 4 章 §4、第 15 章 §4.7 | 同上 |
| **`@Log`** | `annotation/Log.java` | 操作日志（写 `sys_oper_log`） | 第 8 章 | **写操作接口** |
| **`@RepeatSubmit`** | `annotation/RepeatSubmit.java` | 防重提交（时间窗 + 指纹） | 第 9 章 §2 | **写操作接口** |
| **`@RateLimiter`** | `annotation/RateLimiter.java` | 接口限流（令牌桶） | 第 9 章 §2.7 | **★ 一处未用（坦白点）** |
| **`@Sensitive`** | `annotation/Sensitive.java` | 字段脱敏（出参序列化时） | 第 10 章 §2~§4 | **仅 `SysUser` 2 字段** |
| **`@ExcelDictFormat`** | `annotation/ExcelDictFormat.java` | Excel 导入导出时字典翻译 | 第 12 章 §4、第 13 章 §4、第 15 章 §4.6 | **导入 VO 多处** |
| **`@Xss` / `XssValidator`** | `xss/Xss.java`、`xss/XssValidator.java` | 字段级 XSS 校验（替代全局过滤） | 第 10 章 §4 | 少量 |
| **`@TableLogic`** | （MyBatis-Plus 提供） | 逻辑删除标记 | 第 7 章 §4、第 15 章 §4.4 | **业务表普遍** |
| **`@TableName` / `@TableId` / `@TableField(fill=...)`** | （MyBatis-Plus 提供） | 表名 / 主键 / 自动填充 | 第 7 章 §4、第 15 章 §4.4 | **所有 Entity** |
| **`@SensitiveService`（SPI）** | `core/service/SensitiveService.java` | 决定「当前请求要不要脱敏」 | 第 10 章 §2 | 有实现，**返回恒 `true`（关掉脱敏）** |
| **`@Validated(AddGroup/EditGroup)`** | （Jakarta 校验 + `core/validate/`） | 分组校验（新增/修改不同规则） | 第 15 章 §4.5 | **所有 BO/Controller** |

> **空接口「校验分组」**：`core/validate/AddGroup.java`、`EditGroup.java`——它们只是「分组标签」，配合 `@Validated(AddGroup.class)` 使用（第 15 章 §4.5）。

---

## 三、基座模块 / 关键类清单

> 「想找某个能力的实现类，往哪找」——按模块列。

### 3.1 模块职责

| 模块 | 全路径 | 一句话职责 |
|---|---|---|
| **ruoyi-admin** | `ruoyi-admin/` | 启动入口 + Web 控制层（系统管理 Controller）；打包产物 `zxcm-admin.jar` |
| **ruoyi-common** | `ruoyi-base/ruoyi-common/` | 通用工具、注解、异常、基础类（`R`/`BaseEntity`/`LoginHelper`/`RedisUtils`） |
| **ruoyi-framework** | `ruoyi-base/ruoyi-framework/` | 框架装配与切面（Sa-Token、Redis、MyBatis-Plus、AOP、WebSocket、异步、XXL-Job） |
| **ruoyi-system** | `ruoyi-base/ruoyi-system/` | 系统管理（用户/角色/菜单/字典/部门/岗位/参数） |
| **ruoyi-tenant** | `ruoyi-base/ruoyi-tenant/` | 多租户隔离 |
| **ruoyi-generator** | `ruoyi-base/ruoyi-generator/` | 代码生成器（第 15 章 §7.1） |

### 3.2 基座里「最该记住」的几个类

| 类 | 位置 | 为什么重要 | 讲在哪章 |
|---|---|---|---|
| `R` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/R.java` | 统一返回；`SUCCESS=200`/`FAIL=500` | 第 11 章 §2 |
| `TableDataInfo` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/page/TableDataInfo.java:36` | 分页返回；`build(IPage)` | 第 11 章 §2 |
| `BaseEntity` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/BaseEntity.java:33` | 审计字段 + `params`，被所有 Entity 继承 | 第 7 章 §4、第 15 章 §4.4 |
| `BaseController` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/controller/BaseController.java:26` | `toAjax` / `getLoginUser` | 第 15 章 §4.10 |
| `BaseMapperPlus` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/mapper/BaseMapperPlus.java:49` | `selectVoById/List/Page`，省 VO 转换 | 第 7 章、第 15 章 §4.7 |
| `LoginHelper` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/helper/LoginHelper.java:41` | 取当前登录用户 / `loginId` | 第 2 章、第 6 章 |
| `RedisUtils` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/redis/RedisUtils.java:37` | 全项目 Redis 总闸（静态注入 Redisson） | 第 12 章 §4.2 |
| `SpringUtils` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/spring/SpringUtils.java:27` | 静态取 Bean（框架层绕依赖方向的唯一出路） | 第 10 章 §2、第 12 章 §4.2 |
| `GlobalExceptionHandler` | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/web/exception/GlobalExceptionHandler.java:53` | 全局异常兜底 | 第 11 章 §4 |
| `PlusSaTokenDao` | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/satoken/dao/PlusSaTokenDao.java:32` | Sa-Token 与 Redis 的唯一接线处 | 第 2 章 |
| `SaTokenConfig` | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/SaTokenConfig.java:44` | 两道拦截器（`:55`/`:79`） | 第 3 章 §4 |
| `PlusDataPermissionHandler` | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/handler/PlusDataPermissionHandler.java:64` | 数据权限 SQL 改写核心 | 第 4 章、第 15 章 §4.7 |
| `FileUploadUtils` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/FileUploadUtils.java:47` | 上传四步 + 安全校验 | 第 13 章 §4 |
| `ExcelUtil` | `ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/poi/ExcelUtil.java:40` | 导入导出封装 | 第 13 章 §4 |
| `WebSocketSessionManager` | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/websocket/WebSocketSessionManager.java:33` | 会话管理 + 3 秒防抖 | 第 14 章 §4 |
| `ThreadPoolConfig` / `Threads` | `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/ThreadPoolConfig.java:37`、`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/Threads.java:29` | 线程池 + 优雅停机 | 第 14 章 §4 |

---

## 四、配置项索引（`application*.yml`）

> 「某个开关在哪个文件哪一行」——查这张表。**配置文件路径**：`ruoyi-admin/src/main/resources/application.yml`（主）、`application-dev.yml` / `application-prod.yml`（环境）。

| 配置 | 值 | 位置 | 讲在哪章 |
|---|---|---|---|
| 应用名 / 版本 | `zxcm` / `v0.9` | `application.yml:4/6` | 第 0 章 §4.4 |
| HTTP 端口 | `8080` | `application.yml:30` | 第 0 章 |
| Servlet 容器线程（配置块名为 `undertow`，**实际容器是 Tomcat**） | io=8 / worker=888 | `application.yml:45/47` | 第 0 章 §4.4 |
| 激活 Profile | `@profiles.active@` | `application.yml:67` | 第 0 章 §4.4 |
| 上传大小限制 | `80MB` | `application.yml:72/74` | 第 13 章 §4（坦白点二） |
| Jackson 时间格式 | （`jackson` 块） | `application.yml:88` | 第 11 章 |
| **Sa-Token：token 名** | `token` | `application.yml:103` | 第 2 章 |
| **Sa-Token：有效期** | `186400`（秒，注释写"一天"实际≈51.8h） | `application.yml:105` | 第 5 章（坦白点） |
| **Sa-Token：`is-concurrent`** | `true`（允许多端） | `application.yml:109` | 第 5 章 |
| **Sa-Token：`is-share`** | `false`（不共享 token） | `application.yml:111` | 第 5 章 |
| 免登录白名单 | `security.excludes` | `application.yml:156` | 第 3 章 §4 |
| `/profile/**` 免登录 | —— | `application.yml:167` | 第 13 章 §4（坦白点一） |
| **XSS 过滤开关/排除/匹配** | `true` / `/system/notice` / `/system/*,/monitor/*,/tool/*` | `application.yml:315/317/319` | 第 10 章 §4（坦白点二） |
| **全局线程池：开关** | **`false`（★整个块失效）** | `application.yml:324` | 第 14 章 §4（坦白点一） |
| 全局线程池：核心/最大/队列/空闲 | 8 / 16 / 128 / 300 | `application.yml:326/328/330/332` | 第 14 章 §4 |
| 全局线程池：拒绝策略 | `CALLER_RUNS_POLICY` | `application.yml:338` | 第 14 章 §4 |
| Redisson 缓存组 | `redissonCacheMap` | `application.yml:341` | 第 12 章 §4（坦白点一） |
| MyBatis-Plus：mapper 包 | `com.ruoyi.**.mapper` | `application.yml:209` | 第 7 章 §4 |
| MyBatis-Plus：XML 位置 | `classpath*:mapper/**/*Mapper.xml` | `application.yml:211` | 第 7 章 §4 |
| **文件上传根路径** | `D:\java project\document\Resources` | `application-dev.yml:4`、`application-prod.yml:4` | 第 13 章 §4（坦白点三） |

---

## 五、缺口清单汇总（全卷 46 处「坦白点」一网打尽）

> **这是本卷最该反复看的一页。** 用户要求「缺口主动写出来」——把散在各章的坦白点汇到一处，**答辩前过一遍**。**主动交代缺口的开发者，比假装完美的开发者更可信。**
>
> 标注 ★ 的是「**面试很可能被追问**」的重点坦白点。

### 第 3 章 · 鉴权：接口权限（1 处）

| # | 缺口 | 位置 |
|---|---|---|
| 1 | **白名单里有"看着像文件访问"的路径被放行**（`/common/temp/resource`，注释写"文件对比"）——加白名单前必须问「这个接口真的不需要登录吗」 | 第 3 章 §4 |

### 第 4 章 · 数据权限（3 处）

| # | 缺口 | 位置 |
|---|---|---|
| 2 | ★ **标准轨 `DataScopeType.DEPT` 用单数 `#{#user.deptId}`，而 `LoginUser` 是多部门 `deptIds` 数组**——标准轨**先天不完备**，业务都走定制轨（`MYDEPT` 系列） | 第 4 章 §2（`DataScopeType.java:50`） |
| 43 | ★★ **`elseSql` 兜底是死分支**——handler 里 `isSuccess = true` 写死，`:161-163`/`:187-189` 的 `if (!isSuccess)` 永远不成立，「算不出就 `1=0`」的设计**实际没生效** | 第 4 章 §4.4（`PlusDataPermissionHandler.java:158/161-163`） |
| 44 | ★★ **配置缺失时默认放行，安全默认值方向反了**——定制轨某列值为空时 `return ""`，等于不加条件、看到全部数据；正确做法应是收敛到最小权限（空集） | 第 4 章 §4.4（`PlusDataPermissionHandler.java:142-143`） |

### 第 5 章 · 多点登录（5 处）

| # | 缺口 | 位置 |
|---|---|---|
| 3 | **`DeviceType` 定义了 5 种，本项目只用了 PC**（两处登录写死 `DeviceType.PC`） | 第 5 章 §4（`SysLoginService.java:154/179`） |
| 4 | **`max-login-count` 未显式配置**——"同账号最多 12 处在线"是**隐式继承框架默认**，产品文档没提 | 第 5 章 §4 |
| 5 | ★ **`timeout` 注释写"一天"，实际 `186400` 秒 ≈ 51.8 小时**——注释与值不符，以数值为准 | 第 5 章 §4（`application.yml:105`） |
| 6 | **`deptName`/`loginLocation` 被注释掉了**——在线用户页**看不到"部门""登录地点"两列** | 第 5 章 §4（`UserActionListener.java:70/76`） |
| 7 | ★ **`searchTokenValue` 底层走 `RedisUtils.keys` 全库扫描**——在线用户量大时是**性能隐患**（`keys` 会阻塞 Redis） | 第 5 章 §4 |

### 第 6 章 · 多租户（4 处）

| # | 缺口 | 位置 |
|---|---|---|
| 8 | ★ **"免隔离表清单"硬编码在 Java 里**（`TenantProperties.getInstance()`），**加系统级共享表必须改代码发版** | 第 6 章 §4（`TenantProperties`） |
| 9 | **`TenantProperties.getInstance()` 无锁**——并发首次调用可能 `new` 两次（因初始化很轻，影响小） | 第 6 章 §4 |
| 10 | ★ **`clearCache()` 只清 `LOGIN_CACHE`，没清 `noLogin`/`currentTenantId` 两个 ThreadLocal**——线程池复用线程时**可能"继承"上一个任务的租户**（串数据风险） | 第 6 章 §4（`LoginHelper.java:113`） |
| 11 | **默认租户 `00000000` 是双刃剑**——好处永不空租户，坏处"误落该租户会看到数据" | 第 6 章 §4 |

### 第 7 章 · ORM 支撑（5 处）

| # | 缺口 | 位置 |
|---|---|---|
| 12 | ★ **分页 `maxLimit = -1`（不限单页上限）**——`pageSize` 传巨大值可**拉爆内存**（DoS 风险） | 第 7 章 §4（`MybatisPlusConfig:72`） |
| 13 | ★ **`BlockAttackInnerInterceptor`（防全表更新/删除）注释提到却没注册**——**漏写 `WHERE` 会真的全表更新** | 第 7 章 §4（`MybatisPlusConfig:44-57`） |
| 14 | **雪花 ID 靠网卡 IP 推机器位**——**多网卡 / 容器化（Pod IP 变化）/ 同机多实例**时可能冲突 | 第 7 章 §4 |
| 15 | **未登录（且没标 `noLogin`）时 `create_by` 可能是 `null`**——"哪些数据 `create_by` 为空"本身是排查线索 | 第 7 章 §4 |
| 45 | **乐观锁插件注册了，但全项目没有字段标 `@Version`**——这层能力实际未启用，答辩时别把「乐观锁」说成已落地 | 第 7 章 §4（`MybatisPlusConfig.java:55`） |

### 第 8 章 · 操作日志与登录日志（4 处）

| # | 缺口 | 位置 |
|---|---|---|
| 16 | ★ **`businessType` 存的是 `ordinal()`（枚举下标）**——**中途往枚举中间插值会让历史数据含义错位** | 第 8 章 §4（`BusinessType.java:25-75`） |
| 17 | **上传/导入接口的参数被过滤**（`MultipartFile` 被跳过）——`operParam` 里看不到"谁上传了哪个文件" | 第 8 章 §4 |
| 18 | **`operLocation`（IP 归属地）被注释掉了**——和第 5 章 `loginLocation` 是同一个缺口 | 第 8 章 §4 |
| 19 | **操作日志只有手动清空，无自动归档** | 第 8 章 §4 |

### 第 9 章 · 防重提交与限流（5 处）

| # | 缺口 | 位置 |
|---|---|---|
| 20 | ★★ **`@RateLimiter` 全项目一处未用**——限流组件（注解 + 切面 + 令牌桶 + 维度枚举）**全写好了但没启用** | 第 9 章 §2.7 |
| 21 | **限流异常抛的是 `RuntimeException`**——会被 `GlobalExceptionHandler` 当**系统异常**（返回"操作失败"而非"限流提示"） | 第 9 章 §2.7（`RateLimiterAspect.java:59`） |
| 22 | **`RepeatSubmitProperties` 定义了却无人注入**（遗留/未接线） | 第 9 章 §2 |
| 23 | **防重的"成功才删 key"依赖返回值是 `R`**——返回 `void`/其它类型时语义失效 | 第 9 章 §2（`RepeatSubmitAspect` 判断 `instanceof R`） |
| 24 | ★ **唯一约束不是每张表都有**——幂等的最后一道防线**依赖开发者自觉** | 第 9 章 §4 |

### 第 10 章 · 脱敏与 XSS 与请求包装（4 处）

| # | 缺口 | 位置 |
|---|---|---|
| 25 | ★★ **脱敏被"一行代码"关掉**——`SysSensitiveServiceImpl.isSensitive()` **恒返回 `true`**，全项目**明文输出** | 第 10 章 §4.4（`SysSensitiveServiceImpl.java:37`） |
| 26 | ★ **XSS 过滤只覆盖 `/system/*,/monitor/*,/tool/*`**——**业务接口没过滤** | 第 10 章 §4.9（`application.yml:319`） |
| 27 | **`@Sensitive` 全项目只有 `SysUser` 两个字段在用** | 第 10 章 §4.4 |
| 28 | **`isSensitive()` 命名与语义相反**（返回 `true` = 不脱敏） | 第 10 章 §4.4 |

### 第 11 章 · 异常体系与统一返回（4 处）

| # | 缺口 | 位置 |
|---|---|---|
| 29 | ★ **`BaseException`/`UserException` 没有专属处理器**——**登录失败 code 是 500**（和系统错混在一起） | 第 11 章 §4.8 |
| 30 | ★ **兜底 handler 把原始 `e.getMessage()` 返回前端**——可能泄漏表名/SQL/路径（**信息泄漏风险**） | 第 11 章 §4.8（`GlobalExceptionHandler.java:163/173`） |
| 31 | **`GlobalException` 是零引用的死代码** | 第 11 章 §4.8 |
| 32 | **`TableDataInfo` 同名类在项目里有两份**（`ruoyi-common` 泛型版 + `zxcm-workflow` 非泛型版，**包名相同**） | 第 11 章 §4.8 |

### 第 12 章 · 缓存与 Redis 与字典与系统配置（3 处）

| # | 缺口 | 位置 |
|---|---|---|
| 33 | ★ **Spring Cache（`@Cacheable`）配好了却完全没用**——业务代码里的缓存注解全被注释掉 | 第 12 章 §4.8（`RedisConfig.java:47`、`SysUserServiceImpl` 各处） |
| 34 | ★ **字典/参数缓存大多没有 TTL**——`setCacheObject(key, value)` 无过期时间，**不会雪崩但会永久驻留**；且**不缓存空值 → 存在穿透风险** | 第 12 章 §4.8（`RedisUtils.java:103`、`SysConfigServiceImpl.java:94`） |
| 35 | **`RedisUtils` 用静态注入（`SpringUtils.getBean`）**——对**单元测试不友好** | 第 12 章 §4.8 |

### 第 13 章 · 文件上传与 Excel（5 处）

| # | 缺口 | 位置 |
|---|---|---|
| 36 | ★ **`/profile/**` 是免登录的**——**谁拿到 URL 谁就能看文件**（URL 泄漏即文件公开） | 第 13 章 §4.13（`application.yml:167`） |
| 37 | **两套上传大小限制不一致**（Servlet 层 80MB vs 代码层 100MB）——**实际 80MB 先生效，代码层那个形同虚设** | 第 13 章 §4.13 |
| 38 | ★ **文件路径写死 Windows 绝对路径，且 dev/prod 是同一个**（`D:\java project\document\Resources`） | 第 13 章 §4.13（`application-dev.yml:4`/`application-prod.yml:4`） |
| 39 | **`CommonController.uploadFile` 自己 catch 了异常**——未走全局处理器 | 第 13 章 §4.13（`CommonController.java:116`） |
| 46 | ★ **用户 Excel 导入不是「全成功或全失败」**——监听器每行校验通过就立刻 `insertUser`，Controller 没有把整份导入包进事务；前面成功的行不会因后面失败而回滚 | 第 13 章 §4.11（`SysUserImportListener.java:136/175-180`） |

### 第 14 章 · WebSocket 与异步与线程池与定时任务（3 处）

| # | 缺口 | 位置 |
|---|---|---|
| 40 | ★★ **`thread-pool.enabled: false`**——`maxPoolSize`/`queueCapacity`/`keepAliveSeconds`/`rejectedExecutionHandler` **全部不生效**，只有 `corePoolSize` 起作用 | 第 14 章 §4.11（`application.yml:324`） |
| 41 | ★ **WebSocket 用裸线程池 `Executors.newScheduledThreadPool(20)`**——**不受 Spring 管理、无优雅停机、默认 `AbortPolicy` 会丢消息** | 第 14 章 §4.11（`WebSocketSessionManager.java:39`） |
| 42 | ★ **WebSocket 以 token 为键推消息**——**推送方必须知道接收人 token、多端只推到一个 token**，改成 `userId` 更合理 | 第 14 章 §4.11 |

**合计 46 处缺口**（第 3 章 1 + 第 4 章 3 + 第 5 章 5 + 第 6 章 4 + 第 7 章 5 + 第 8 章 4 + 第 9 章 5 + 第 10 章 4 + 第 11 章 4 + 第 12 章 3 + 第 13 章 5 + 第 14 章 3）。

> **怎么用这一页**：**答辩前，把标 ★ 的挑出来**（约 20 处），每处想清「**这是什么问题、为什么会这样、怎么改**」——被追问时**主动说**，比被问出来强得多。
> **最该记住的 3 处**：#25（脱敏被关）、#33（Spring Cache 配了没用）、#40（线程池配置假生效）——它们都是「**看起来很完备、实际没生效**」的典型，最能体现你「真的读过源码」。

---

## 六、「我要找 X」→ 去第几章

| 我想找…… | 去 |
|---|---|
| 基座由哪些模块组成、依赖方向 | 第 0 章 §2.1、§4.2 |
| 网上教程和本仓库版本对不上 | 第 0 章 §3.2 |
| 一个类该放基座还是业务 | 第 0 章 §4.5 |
| 权限体系的整体图（认证/鉴权/数据权限/多端） | 第 1 章 §1 |
| RBAC 是怎么建表的 | 第 1 章 §1~§3 |
| 登录全链路（验证码→token→Redis） | 第 2 章全章 |
| `LoginUser` 里装了什么东西 | 第 2 章 §4 |
| `@SaCheckPermission` 怎么被拦截、怎么比对 | 第 3 章 §4 |
| 401 和 403 怎么区分 | 第 3 章 §4 |
| 不同角色看到不同的数据（数据权限） | 第 4 章全章 |
| 数据权限的 SpEL 模板、自定义维度 | 第 4 章 §2、§4 |
| 同一账号多设备登录 / 顶人下线 | 第 5 章全章 |
| 在线用户是怎么查出来的 | 第 5 章 §4 |
| 多租户是怎么隔离的、哪些表免隔离 | 第 6 章全章 |
| MyBatis-Plus 插件链怎么配的 | 第 7 章 §4 |
| 逻辑删除、审计字段自动填充 | 第 7 章 §4 |
| 操作日志怎么记的、`@Log` 怎么写 | 第 8 章全章 |
| 为什么表单不能重复提交 | 第 9 章 §2 |
| 限流组件在哪、为什么没启用 | 第 9 章 §2.7 |
| 手机号怎么脱敏、为什么没生效 | 第 10 章 §4.4 |
| XSS 是怎么过滤的、覆盖了哪些接口 | 第 10 章 §4.9 |
| 统一返回 `R` 和分页 `TableDataInfo` | 第 11 章 §2 |
| 异常是怎么被统一兜住的 | 第 11 章 §4 |
| Redis 客户端怎么创建、`RedisUtils` 怎么用 | 第 12 章 §4 |
| 字典和系统参数是怎么缓存的 | 第 12 章 §4 |
| 文件上传存哪、怎么防路径穿越 | 第 13 章 §4 |
| Excel 导入导出怎么把"值"翻成"标签" | 第 13 章 §4、第 15 章 §4.6 |
| WebSocket 消息推送怎么做的 | 第 14 章 §4 |
| `@Async` 为什么要配线程池 | 第 14 章 §4 |
| 定时任务为什么用 XXL-Job | 第 14 章 §4、§7 |
| **怎么在基座上从零加一个新模块** | **第 15 章全章** |
| 加模块最容易踩的坑 | 第 15 章 §4.3、Q6 |
| **这个项目有哪些坑 / 缺口** | **本附录 §五** |

---

## 七、术语表（本卷贯穿的名词，一句话）

| 术语 | 一句话解释 | 详见 |
|---|---|---|
| **基座** | `ruoyi-base` + `ruoyi-admin`，所有业务模块共用的「通用能力层」 | 第 0 章 §1 |
| **版本红线** | 网上教程多为 5.x（`org.dromara`/`jakarta`），本仓库是 `4.0.1s`（`com.ruoyi`/`javax`） | 第 0 章 §3.2 |
| **依赖方向铁律** | 业务依赖框架，框架**绝不**反向依赖业务 | 第 0 章 §4.2 |
| **RBAC** | 用户 → 角色 → 权限的三层模型 | 第 1 章 §1 |
| **菜单三合一** | `sys_menu` 一张表同时管「菜单显示 + 前端路由 + 权限码」 | 第 1 章 |
| **功能权限 vs 数据权限** | 前者=能不能进这个接口（`@SaCheckPermission`）；后者=进去后能看哪些行（`@DataPermission`） | 第 1 章、第 4 章 |
| **认证 vs 鉴权** | 认证=你是谁（登录）；鉴权=你能不能调（比对权限码） | 第 1 章、第 2/3 章 |
| **`LoginUser`** | 登录态载体，装「每次请求都要用、查库太重」的东西（权限码/角色/租户/部门） | 第 2 章 §4 |
| **`loginId`** | 「租户 + 用户类型 + 用户 id」拼成的登录标识 | 第 2 章、第 6 章 |
| **无痕改写 SQL** | 数据权限靠 JSqlParser 解析 SQL、注入 `WHERE`，业务代码无感 | 第 4 章 §4 |
| **双轨数据权限** | 标准轨 `DataScopeType`（框架自带）+ 定制轨 `ContractPermission`（本项目按模块） | 第 4 章 §2 |
| **同端互斥 / 多点登录** | 控制同一账号在多设备/同设备多会话的在线策略（`is-concurrent`/`is-share`） | 第 5 章 |
| **多租户** | 一套系统隔离多个租户的数据（靠 `tenant_id` + MP 租户插件） | 第 6 章 |
| **免隔离表** | 多租户下**不**加 `tenant_id` 条件的共享表清单（**本项目硬编码在 Java**） | 第 6 章 §4 |
| **逻辑删除** | 删除不真删，把 `del_flag` 置 `2`（`@TableLogic`） | 第 7 章 §4 |
| **自动填充** | MyBatis-Plus 在 insert/update 前自动写审计字段（`createBy`/`updateTime`） | 第 7 章 §4 |
| **VO 映射** | `BaseMapperPlus` 的 `selectVo*` 直接返回 VO，省手写转换 | 第 7 章、第 15 章 §4.7 |
| **操作日志** | `@Log` + AOP，把写操作记进 `sys_oper_log` | 第 8 章 |
| **防重提交** | `@RepeatSubmit`：同用户+同参数+短时间内的第二次请求被拒 | 第 9 章 §2 |
| **限流** | `@RateLimiter`：令牌桶限制接口 QPS（**本项目未启用**） | 第 9 章 §2.7 |
| **脱敏** | 出参序列化时把手机号/身份证等变成 `138****5678`（**本项目被关**） | 第 10 章 §4.4 |
| **XSS 过滤** | 清洗入参里的 HTML/脚本（**本项目只覆盖系统管理接口**） | 第 10 章 §4.9 |
| **静态注入** | 框架层用 `SpringUtils.getBean(接口.class)` 拿实现，绕开依赖方向 | 第 10 章 §2、第 12 章 §4.2 |
| **统一返回 `R`** | `R<T>{code,msg,data}`，HTTP 恒 200，业务码在 body | 第 11 章 §2 |
| **旁路缓存（cache-aside）** | 读时先查缓存、未命中查库再回填；写时更新库 + 更新缓存 | 第 12 章 §4 |
| **缓存穿透/雪崩** | 穿透=查不存在的 key 打到库；雪崩=大量 key 同时过期 | 第 12 章 §4 |
| **字典** | `sys_dict_type` + `sys_dict_data`，一处配置、前端+后端+Excel 多处翻译 | 第 12 章 §4、第 15 章 §4.2 |
| **`@ExcelDictFormat`** | Excel 导入导出时把字典「值」翻成「标签」 | 第 13 章 §4、第 15 章 §4.6 |
| **拒绝策略 `CallerRunsPolicy`** | 线程池满了不丢任务，让调用方线程自己去跑（自动反压） | 第 14 章 §4 |
| **优雅停机** | `Threads.shutdownAndAwaitTermination`：shutdown → awaitTermination → shutdownNow | 第 14 章 §4 |
| **XXL-Job** | 分布式定时任务调度中心（本项目用它而非 Quartz/`@Scheduled`） | 第 14 章 §4、§7 |
| **三层对象** | Entity（面表）/ BO（面入参）/ VO（面出参） | 第 15 章 §2.1 |
| **接入点** | 基座给业务预留的「钩子」：权限码、日志注解、数据权限注解、字典…… | 第 15 章 §1.2 |
| **校验分组** | 一个 BO 用 `@Validated(AddGroup/EditGroup)` 应付「新增/修改」两套校验 | 第 15 章 §4.5 |

---

## 本卷导航回顾

| 章 | 文件 | 主题 |
|---|---|---|
| 0 | [第0章-基座全景与版本坐标](第0章-基座全景与版本坐标.md) | 基座全景 + 版本红线 + 模块划分 |
| 1 | [第1章-权限体系总览](第1章-权限体系总览.md) | 权限四件事 + RBAC 模型 |
| 2 | [第2章-认证：登录全链路](第2章-认证：登录全链路.md) | 登录 → `LoginUser` → token 落 Redis |
| 3 | [第3章-鉴权：接口权限](第3章-鉴权：接口权限.md) | 两道拦截器 + 权限码比对 |
| 4 | [第4章-数据权限](第4章-数据权限.md) ★ | `@DataPermission` + SpEL 无痕改写 SQL |
| 5 | [第5章-多点登录](第5章-多点登录.md) ★ | 多端在线 + 顶人下线 + 在线用户 |
| 6 | [第6章-多租户](第6章-多租户.md) | 租户隔离 + 免隔离表 |
| 7 | [第7章-ORM支撑](第7章-ORM支撑.md) | MyBatis-Plus 插件链 + 逻辑删除 + 填充 |
| 8 | [第8章-操作日志与登录日志](第8章-操作日志与登录日志.md) | `@Log` + AOP + 日志表 |
| 9 | [第9章-防重提交与限流](第9章-防重提交与限流.md) | `@RepeatSubmit` + `@RateLimiter` |
| 10 | [第10章-脱敏与XSS与请求包装](第10章-脱敏与XSS与请求包装.md) | `@Sensitive` + `XssFilter` + 请求包装 |
| 11 | [第11章-异常体系与统一返回](第11章-异常体系与统一返回.md) | `R` + `GlobalExceptionHandler` |
| 12 | [第12章-缓存与Redis与字典与系统配置](第12章-缓存与Redis与字典与系统配置.md) | Redisson + `RedisUtils` + 字典/参数 |
| 13 | [第13章-文件上传与Excel](第13章-文件上传与Excel.md) | 本地盘上传 + EasyExcel |
| 14 | [第14章-WebSocket与异步与线程池与定时任务](第14章-WebSocket与异步与线程池与定时任务.md) | WS + `@Async` + 线程池 + XXL-Job |
| 15 | [第15章-基于基座写一个新模块](第15章-基于基座写一个新模块.md) ★★ | 综合实战：从零加一个模块 |
| **A** | 本附录 | 横切索引 + 注解 / 配置 / 缺口速查 |
| B | [附录B-框架学习路线图](附录B-框架学习路线图.md) | 网上资料「先学什么后学什么」路线 |
