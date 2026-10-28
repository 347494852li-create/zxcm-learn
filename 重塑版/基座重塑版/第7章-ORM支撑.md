# 第 7 章 · ORM 支撑：MyBatis-Plus 插件链与基础封装

> **前两章都在讲"SQL 怎么被自动改写"**（租户、数据权限）。**本章把镜头拉远**，看**整个 ORM 层是怎么搭起来的**——不只是那两道改写，还有**自动填充创建人/时间、逻辑删除、雪花 ID、基础实体和 Mapper 封装**。
>
> **一句话概括本章**：**本项目让"写 CRUD"变成"调方法"**——不写 SQL、不手填创建人、不担心主键重复、不真删数据。业务开发只需要关注"字段和业务逻辑"。
>
> **本章第一次讲透**：① MyBatis-Plus 的四道插件（租户/数据权限/分页/乐观锁）**完整一条链**；② `CreateAndUpdateMetaObjectHandler` 怎么**自动填 `create_by` / `create_time`**；③ **雪花 ID** 为什么要在集群里"绑网卡"；④ **逻辑删除**（`@TableLogic`）的意义与代价；⑤ `BaseEntity` / `BaseMapperPlus` 这两个"基类封装"省了什么；⑥ 本章顺带把第 4 章的 `PlusDataPermissionInterceptor` **本体**补全（第 4 章只讲了它调用的 `Handler`）。

---

## 一、这部分是做什么的？

> ### 零基础先补概念
>
> 1. **ORM（对象关系映射）**：用 Java 对象操作数据库表。比如 `baseMapper.insert(draft)` 一行，框架帮你生成 `INSERT INTO ...`。本项目用 MyBatis-Plus（MyBatis 的增强版）。
> 2. **Wrapper（条件构造器）**：MyBatis-Plus 用来"拼 WHERE 条件"的 Java 对象。比如 `new LambdaQueryWrapper<SysDept>().eq(SysDept::getStatus, "0")`（`SysDeptServiceImpl.java:454-455`），等价于 `WHERE status = '0'`；好处是字段名用方法引用，写错编译就报错。
> 3. **插件链（InnerInterceptor）**：MyBatis-Plus 允许多个插件排成一队，在 SQL 执行前依次改写。本项目的顺序是"租户 → 数据权限 → 分页 → 乐观锁"（`MybatisPlusConfig.java:47-55`）。
> 4. **自动填充（MetaObjectHandler）**：插入/更新时，框架回调你写的 Handler，把 `create_by`、`create_time` 这些公共字段填好，业务代码不用手写（`CreateAndUpdateMetaObjectHandler.java:36`）。

### 1.1 "ORM 支撑"是什么

**ORM（对象关系映射）** 就是"用 Java 对象操作数据库表"。本项目的 ORM 层基于 **MyBatis-Plus**（MyBatis 的增强版）。它提供的支撑分四块：

| 支撑 | 解决的问题 | 本项目关键类 |
|---|---|---|
| **SQL 自动改写** | 租户隔离、数据权限、分页、乐观锁 | `MybatisPlusInterceptor` + 4 个 `InnerInterceptor` |
| **字段自动填充** | `create_by` / `create_time` 不用手写 | `CreateAndUpdateMetaObjectHandler` |
| **主键生成** | 分布式下 ID 不重复 | `DefaultIdentifierGenerator`（雪花 ID） |
| **公共字段 + 通用方法** | 少写重复代码 | `BaseEntity` / `BaseMapperPlus` |

**"支撑"的意思是**：这些**不是业务功能，而是"让业务功能好写"的底座**。业务模块（`zxcm-contract` 等）**直接复用**，不用各自实现。

### 1.2 你在写业务时会"看不见"它，但它一直在

举个最普通的业务代码（新增一条草稿）：

```java
// Service 里
ContractDraft draft = BeanCopyUtils.copy(bo, ContractDraft.class);
baseMapper.insert(draft);          // ← 就这一行
```

**这一行 `insert` 背后，至少有 6 件事在发生**（本章主角；前 5 件是 ORM 层直接做的，第 6 件是整条请求链路的兜底）：

1. **主键**：自动生成雪花 ID（不用你设 `id`）；
2. **创建人**：自动填 `create_by` = 当前登录用户名；
3. **创建时间 / 更新时间**：自动填 `create_time` / `update_time`；
4. **租户**：SQL 被自动加 `tenant_id`（第 6 章）；
5. **逻辑删除字段**：`del_flag` 的初始值由**数据库默认值或实体赋值**负责（框架不会替你填）；框架保证的是**后续查询自动带 `del_flag='0'`、`delete` 变成 `UPDATE`**（见 §4.4）；
6. **异常兜底**：`insert` 抛出的异常由全局异常处理器转成统一响应（第 11 章）——这不是 `insert` 自动做的，但一次请求一定会经过它。

**你只写了一行 `insert`，但系统在背后做了这么多事**——这就是"ORM 支撑"的价值。

---

## 二、站在架构师 / 开发经理角度：为什么这么设计？

### 2.1 为什么要"自动填充"创建人/时间？

**如果不自动填**，每个新增/修改方法都要写：

```java
draft.setCreateBy(LoginHelper.getUsername());     // 每个 insert 都写
draft.setCreateTime(new Date());                  // 每个 insert 都写
draft.setUpdateBy(LoginHelper.getUsername());     // 每个 update 都写
draft.setUpdateTime(new Date());                  // 每个 update 都写
```

**问题**：① **几十上百个方法重复**；② **一定会漏**（漏填 `update_time` 的 bug 很常见）；③ **改口径要改所有地方**（比如"改成填 user_id 而不是 username"）。

**自动填充的价值**：**声明在实体上（`@TableField(fill = ...)`），实现集中在 Handler 里，一次配好、处处生效。** 业务代码**一行都不用写**。

### 2.2 为什么主键要用"雪花 ID"？

**主键方案对比**：

| 方案 | 集群唯一？ | 有序？ | 业务含义泄露？ | 本项目 |
|---|---|---|---|---|
| 数据库自增 | 单库唯一，**分库冲突** | 有序 | 泄露业务量 | ✗ |
| UUID | 全局唯一 | **无序**（索引差） | 不泄露 | ✗ |
| **雪花 ID** | **全局唯一（机器位区分）** | **大体有序** | 不泄露 | **✔** |

**雪花 ID 的核心**是"**时间戳 + 机器 ID + 序列号**"，**靠"机器 ID"保证多台机器生成的 ID 不碰撞**。

**本项目的一个关键细节**（`MybatisPlusConfig.java:97-100`）：

```java
@Bean
public IdentifierGenerator idGenerator() {
    return new DefaultIdentifierGenerator(NetUtil.getLocalhost());   // ★ 用本机网卡信息当"机器 ID"
}
```

**注释写得很明白**（`:94-95`）：「**使用网卡信息绑定雪花生成器，防止集群雪花 ID 重复**」。

**为什么用"网卡"？** 雪花 ID 需要"机器 ID"来区分不同节点。硬编码机器 ID（如手动给每台机器配 1、2、3）**容易配错、扩容要改配置**。**用本机 IP（网卡信息）自动推算出机器位**——**同一台机器 IP 固定，不同机器 IP 不同**，天然区分。**这是"零配置集群"的巧思。**

> ⚠️ **坦白点（网卡方案的边界）**：用 IP 推导机器位，**在多网卡、容器化（K8s 每次重建 Pod IP 会变）场景下可能不稳**。而且**"同一台机器上跑两个实例"会拿到同一机器位 → 可能冲突**。**生产上更稳的做法是"显式配置 workerId"或用中心化发号器**。本项目是单体部署（第 0 章），**这个方案够用**，但要如实说清它的适用边界。

### 2.3 为什么要"逻辑删除"而不是"真删"？

**逻辑删除 = 不执行 `DELETE`，而是 `UPDATE 表 SET del_flag='删除' WHERE id=?`。** 查询时自动加 `del_flag='正常'`。

**为什么？** 因为业务上**"删除"往往不是"忘记"**：

- **数据可追溯**：合同、审批这些数据，**删了要能找回**（审计、纠纷取证）；
- **外键完整**：真删会破坏关联（删了部门，用户的 `dept_id` 就成了悬空外键）；
- **误删可恢复**：逻辑删除**天然可撤销**。

**代价是什么？** ——**① 表会越来越大**（删的数据还在）；**② 每条查询都多一个 `del_flag` 条件**（框架自动加）；**③ 唯一索引要特殊处理**（同一"名字"逻辑删除后还能再建同名的，唯一索引要带上 `del_flag`）。

**本项目怎么用？** 在实体字段上标 `@TableLogic`，比如 `SysTenant.delFlag`（`SysTenant.java:46-47`）和 `SysRole.delFlag`（`SysRole.java:113`）。**标了之后，MyBatis-Plus 自动处理"删除变 UPDATE"和"查询加条件"。**

### 2.4 为什么要 `BaseEntity` 和 `BaseMapperPlus`？

**它们的价值是"消除重复"**：

- **`BaseEntity`**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/BaseEntity.java:33`）：把**每张表都有的公共字段**（`create_by` / `create_time` / `update_by` / `update_time` / `params`）抽到一个基类，**所有实体继承它**，不用每个实体重复声明。
- **`BaseMapperPlus`**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/mapper/BaseMapperPlus.java:49`）：在 MyBatis-Plus 的 `BaseMapper<T>` 之上，**加了 VO 转换系列方法**（`selectVoList` / `selectVoPage` / `selectVoById`），**省掉"entity → vo"的手工转换**。

**这是"好的抽象"的典型**：**把"所有表都一样的部分"沉淀到基类，业务实体只写"自己特有的字段"。** 但也要注意 **YAGNI**——**本项目用了 `BaseEntity`，是因为"公共字段"确实是全表统一的**；如果你项目里表结构千差万别，强行抽基类反而是负担。

### 2.5 ★ 四道插件为什么是"这个顺序"？（承接第 6 章）

再看一遍 `MybatisPlusConfig.java:44-57` 的顺序：

```
① TenantLineInnerInterceptor      :47   ← 租户（最外层隔离）
② PlusDataPermissionInterceptor   :51   ← 数据权限（第 4 章）
③ PaginationInnerInterceptor      :53   ← 分页
④ OptimisticLockerInnerInterceptor:55   ← 乐观锁
```

**一个反直觉的点**：**分页为什么要排在数据权限"之后"？** ——因为**分页的 `count` 语句必须和数据查询用"同样的 WHERE"**。如果分页先执行、数据权限后执行，**count 就会漏算数据权限条件**（总数对不上）。**所以"改条件"的插件（租户、数据权限）必须在"包分页"之前。**

**乐观锁排最后**，因为它只处理 `UPDATE`（加 `version` 条件），和其他插件不冲突。

> 📌 **记住这条链，面试问"你们 MyBatis-Plus 配了哪些插件、什么顺序、为什么"就能答满。**

---

## 三、需要哪些技术？

| 技术 / 机制 | 在本章里具体指什么 | 精确定位 | 详解在哪 |
|---|---|---|---|
| **`MybatisPlusInterceptor`** | 插件总入口（四道插件的容器） | `MybatisPlusConfig.java:45` | §2.5 |
| **`TenantLineInnerInterceptor`** | 多租户自动加条件 | `MybatisPlusConfig.java:47` | 第 6 章 |
| **`PlusDataPermissionInterceptor`** | 数据权限改写 SQL（第 4 章的载体） | `PlusDataPermissionInterceptor.java:46` | §4.7 |
| **`PaginationInnerInterceptor`** | 分页（自动 count + limit） | `MybatisPlusConfig.java:69` | §4.1 |
| **`OptimisticLockerInnerInterceptor`** | 乐观锁（`version` 字段） | `MybatisPlusConfig.java:81` | §4.1 |
| **`CreateAndUpdateMetaObjectHandler`** | 自动填创建人/时间 | `CreateAndUpdateMetaObjectHandler.java:36` | §4.2 |
| **`BaseEntity`** | 公共字段基类 | `BaseEntity.java:33` | §4.3 |
| **`@TableLogic`** | 逻辑删除标记 | 如 `SysRole.java:113` | §4.4 |
| **`@TableField(fill=...)`** | 标记"这个字段要自动填" | `BaseEntity.java:48/:55/:62/:69` | §4.2 |
| **`DefaultIdentifierGenerator`** | 雪花 ID 生成器 | `MybatisPlusConfig.java:97` | §4.5 |
| **`BaseMapperPlus`** | 扩展的通用 Mapper（含 VO 转换） | `BaseMapperPlus.java:49` | §4.6 |
| **`@InterceptorIgnore`** | 跳过某个插件 | 见第 6 章 §2.4 | §4.7 |

---

## 四、功能具体怎么实现（带着你读懂）

### 4.1 四道插件的注册

`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/MybatisPlusConfig.java:44-57`：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    // ① 多租户
    TenantLineInnerInterceptor tenantLineInnerInterceptor = new TenantLineInnerInterceptor();  // :47
    tenantLineInnerInterceptor.setTenantLineHandler(new CommonTenantHandler());                // :48
    interceptor.addInnerInterceptor(tenantLineInnerInterceptor);                               // :49
    // ② 数据权限
    interceptor.addInnerInterceptor(dataPermissionInterceptor());                              // :51
    // ③ 分页
    interceptor.addInnerInterceptor(paginationInnerInterceptor());                             // :53
    // ④ 乐观锁
    interceptor.addInnerInterceptor(optimisticLockerInnerInterceptor());                       // :55
    return interceptor;
}
```

**分页插件的两个配置**（`:69-76`）：

```java
paginationInnerInterceptor.setMaxLimit(-1L);      // :72 单页上限 -1 = 不限
paginationInnerInterceptor.setOverflow(true);     // :74 页码越界时回到最后一页（而不是空）
```

> ⚠️ **坦白点（单页上限设成"不限"）**：`:72` 把 `maxLimit` 设为 `-1`，意思是**"单页可以查任意多行"**。**好处**是导出大表不受限制；**风险**是**"前端传 `pageSize=100000000` 就能一次拉爆内存"**——这是**典型的 DoS 风险点**。**更安全的做法是设一个合理上限（如 500 或 1000）**。**面试若问"分页安全吗"，这是要点破的。**
>
> ⚠️ **另一个坦白点**：文件末尾注释（`:111-113`）**提到了 `BlockAttackInnerInterceptor`（拦截全表删除/更新）和 `IllegalSQLInnerInterceptor`（SQL 规范检查）**，但**代码里并没有注册它们**（`:44-57` 只有四个）。**也就是说"防止全表 UPDATE/DELETE"这层保护本项目没开**——**写错 SQL（漏 WHERE）会真的全表更新**。这是**运维风险**，值得知道。
>
> ⚠️ **第三个坦白点**：乐观锁插件虽然注册了（`:55` 的 `optimisticLockerInnerInterceptor()`），但**全项目没有实体字段标 `@Version`**（可自行 grep 确认）——也就是说这层能力目前**没有业务在用**，属于"框架标配、先挂着"的状态。**面试别把"乐观锁"说成项目已落地的功能。**

### 4.2 自动填充：`CreateAndUpdateMetaObjectHandler`

**第一步：实体字段上声明"要填"**（`BaseEntity.java`）：

```java
@TableField(fill = FieldFill.INSERT)          // :48 只在 insert 时填
private String createBy;

@TableField(fill = FieldFill.INSERT)          // :55
private Date createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)   // :62 insert 和 update 都填
private String updateBy;

@TableField(fill = FieldFill.INSERT_UPDATE)   // :69
private Date updateTime;
```

**第二步：配置 Handler**（`MybatisPlusConfig.java:88-91`）：

```java
@Bean
public MetaObjectHandler metaObjectHandler() {
    return new CreateAndUpdateMetaObjectHandler();
}
```

**第三步：Handler 里写"填什么"**（`CreateAndUpdateMetaObjectHandler.java:36`）：

```java
public class CreateAndUpdateMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {                    // :39
        if (metaObject.getOriginalObject() instanceof BaseEntity) {
            BaseEntity baseEntity = (BaseEntity) metaObject.getOriginalObject();
            Date current = ObjectUtil.isNotNull(baseEntity.getCreateTime())
                    ? baseEntity.getCreateTime() : new Date();         // :43-44
            baseEntity.setCreateTime(current);                         // :45
            baseEntity.setUpdateTime(current);                         // :46
            String username = StringUtils.isNotBlank(baseEntity.getCreateBy())
                    ? baseEntity.getCreateBy() : getLoginUsername();    // :47-48 没填才用登录名
            baseEntity.setCreateBy(username);                          // :50
            baseEntity.setUpdateBy(username);                          // :52
        }
    }

    @Override
    public void updateFill(MetaObject metaObject) {                    // :60
        BaseEntity baseEntity = (BaseEntity) metaObject.getOriginalObject();
        baseEntity.setUpdateTime(new Date());                          // :66 ★ 不管空不空，强制更新
        String username = getLoginUsername();
        if (StringUtils.isNotBlank(username)) {
            baseEntity.setUpdateBy(username);                          // :70
        }
    }

    private String getLoginUsername() {                                // :81
        if (new Boolean(true).equals(LoginHelper.noLogin.get())) {
            return "admin";                                            // :83 ★ 无登录态 → 记名 admin
        }
        try {
            return LoginHelper.getLoginUser().getUsername();            // :92
        } catch (Exception e) {
            log.warn("自动注入警告 => 用户未登录");
            return null;                                               // :90 未登录 → null
        }
    }
}
```

**逐行读：自动填充 Handler 到底改了什么**

| 行号 | 这一行在干什么 | 为什么这么写 |
|---|---|---|
| `:39` | `insertFill` 是"插入前"的回调 | MyBatis-Plus 在生成 INSERT 参数前调用它 |
| `:40`、`:54-56` | 整个方法包在 try/catch 里 | 填充失败会抛 `ServiceException`，避免写一半 |
| `:41` | 只处理 `BaseEntity` 的子类 | 不是所有实体都继承 `BaseEntity`；不继承的就不自动填 |
| `:43` | 有 `createTime` 就用旧值，否则 `new Date()` | 数据迁移时能保留原创建时间 |
| `:45-46` | `createTime` 和 `updateTime` 都设为同一个时间 | 新增那一刻，"创建时间=更新时间" |
| `:47-48` | `createBy` 有值就尊重，没有才取登录名 | "自动填充但不霸道" |
| `:50/:52` | `createBy` 和 `updateBy` 都填 | 新增时更新人就是创建人 |
| `:60` | `updateFill` 是"更新前"的回调 | 只在 `UPDATE` 时触发 |
| `:64-66` | **无条件** `setUpdateTime(new Date())` | 更新时间必须反映"最近一次修改"，不能被旧值覆盖 |
| `:67-70` | 取不到用户名就不覆盖 `updateBy` | 避免把原来记录的操作人擦成 null |
| `:81-83` | `noLogin=true` 时返回 `"admin"` | 定时任务这类"无登录态"的数据也有明确来源 |
| `:87-91` | 真取不到登录用户就返回 `null` | 不抛异常打断业务，但会留下"空创建人"的数据 |

**三个设计细节：**

1. **`insert` 允许"已有值优先"**（`:43`、`:47`）：如果调用方**已经设了 `createBy`/`createTime`**（比如数据迁移时想保留原始创建人），**就尊重它、不覆盖**。**这是"自动填充但不霸道"的体贴。**
2. **`update` 强制刷新 `updateTime`**（`:66`）：注释写「不管为不为空」——**更新时间必须反映"最近一次修改"，不能被旧值覆盖**。
3. **无登录态记名 `"admin"`**（`:82-83`）：`noLogin` 标记（第 6 章 §2.5）时，把创建人记成 `admin`。**这让"系统自动生成的数据"有明确来源**（不是 null）。

> ⚠️ **坦白点（未登录时 `create_by` 可能是 null）**：`:86-91`，**如果既没设 `noLogin`、又真的取不到登录用户，`createBy` 会是 `null`**。所以"哪些数据的 `create_by` 是空的"**本身就是一条排查线索**（要么是没标 `noLogin` 的后台任务写的，要么是异常写入）。

### 4.3 `BaseEntity`：公共字段的家

`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/BaseEntity.java:33-79`：

| 字段 | 填充策略 | 说明 |
|---|---|---|
| `searchValue` | — | `@TableField(exist = false)`（`:41`）**不是数据库字段**，只用来接收前端"通用搜索值" |
| `createBy` | `INSERT`（`:48`） | 创建者 |
| `createTime` | `INSERT`（`:55`） | 创建时间 |
| `updateBy` | `INSERT_UPDATE`（`:62`） | 更新者 |
| `updateTime` | `INSERT_UPDATE`（`:69`） | 更新时间 |
| `params` | — | `@TableField(exist = false)`（`:76`）**不是数据库字段**，用来装"额外查询参数"（如 `beginTime`/`endTime` 时间范围） |

**关键理解**：**`exist = false` 的字段是"内存字段"**——它们**参与 Java 逻辑、但不参与 SQL**。`params` 尤其常见：**用来传递"不想在实体里固化的动态查询条件"**（时间范围、模糊搜索词），XML 里用 `params.beginTime` 取。

> 📌 **注意 `BaseEntity` 里没有 `delFlag`**！**逻辑删除字段是每个实体自己声明的**（见下 §4.4）——因为**不是每张表都要逻辑删除**（有些日志表就是物理删除）。**这个"不放进基类"的取舍是对的**（避免 YAGNI）。

### 4.4 逻辑删除：`@TableLogic`

**用法**：在实体里声明一个字段并标 `@TableLogic`。本项目里各实体自己声明，例如：

```java
// SysRole.java:113
@TableLogic
private String delFlag;

// SysTenant.java:46-47（值含义：-1 删除 / 1 正常）
@TableLogic
private String delFlag;
```

**标了之后 MyBatis-Plus 自动做两件事**：

1. **`deleteById(id)` 变成** `UPDATE sys_role SET del_flag='2' WHERE id=?`（不是真 DELETE）；
2. **框架生成的 `selectXxx` 自动加** `WHERE del_flag='0'`（"已删除"的查不出来）。

**"删除值 / 未删除值"由全局配置决定**：`mybatis-plus.global-config.dbConfig.logicDeleteValue: 2`、`logicNotDeleteValue: 0`（`ruoyi-admin/src/main/resources/application.yml:254-256`）。所以本项目运行时是 **0=正常、2=删除**；如果某个实体用 `@TableLogic(value="0", delval="1")` 显式覆盖，才以实体为准。

> ⚠️ **注意注释打架**：`SysTenant` 字段注释写 `-1:删除 1:正常`（`SysTenant.java:44-47`），`ContractDraft` 实体注释写 `0 未删除 1 已删除`（`ContractDraft.java:139-142`），但全局配置是 `2/0` 且这两个字段都没显式覆盖。**以全局配置（2/0）和你实际查到的值为准**，别照着某处注释断言。

> ⚠️ **坦白点（逻辑删除的代价与"手写 SQL 的风险"）**：**框架只对"它自己生成的 SQL"加 `del_flag` 条件**。**你手写的 XML 原生 SQL（`@Select` 或 `mapper.xml`）不会自动加**——**必须自己写 `AND del_flag='0'`**。**这就是"为什么有的列表里会冒出已删除数据"的常见原因。** 排查时先看"这条 SQL 是框架生成的还是手写的"。

### 4.5 雪花 ID 生成器

`MybatisPlusConfig.java:97-100`（已在 §2.2 讲原理）：

```java
@Bean
public IdentifierGenerator idGenerator() {
    return new DefaultIdentifierGenerator(NetUtil.getLocalhost());   // 用本机 IP 推算机器位
}
```

**实体主键怎么声明？** 用 `@TableId`。示例 `SysTenant.java:41-42`：

```java
@TableId(value = "id")
private Long id;      // 不指定 type → 走全局 idGenerator（application.yml:248 的 idType: ASSIGN_ID）
```

**注意本项目主键类型是 `Long`**（雪花 ID 是 64 位长整型）。**前端接收时要注意"JS 大整数精度丢失"**——**雪花 ID 超过 `Number.MAX_SAFE_INTEGER`（2^53）时，前端 `Number` 会失真**，所以**前后端一般把 Long 序列化成 String 传输**（这属于前端/序列化话题，前端卷有专章）。

### 4.6 `BaseMapperPlus`：通用 Mapper 的封装

`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/mapper/BaseMapperPlus.java:49`：

```java
public interface BaseMapperPlus<M, T, V> extends BaseMapper<T> {      // :49  M=Mapper 自己, T=实体, V=VO
```

**它比原生 `BaseMapper<T>` 多了什么？** 三块：

**① 批量操作**（省掉循环调用）：

```java
default boolean insertBatch(Collection<T> entityList) { ... }         // :74 批量插入
default boolean updateBatchById(Collection<T> entityList) { ... }     // :81 批量更新
default boolean insertOrUpdateBatch(Collection<T> entityList) { ... } // :88 批量插入或更新
default boolean insertOrUpdate(T entity) { ... }                      // :138 单条"有则更新、无则插入"
```

**② VO 转换系列**（`entity → vo` 不用手写）：

```java
default V selectVoById(Serializable id) { ... }                       // :150 按 ID 查 VO
default <C> C selectVoById(Serializable id, Class<C> voClass)         // :157
default List<V> selectVoList(Wrapper<T> wrapper)                      // :210 查列表 VO
default <P extends IPage<V>> P selectVoPage(IPage<T> page, Wrapper<T> wrapper)  // :225 分页 VO
```

**③ 泛型反射工具**：`currentVoClass()`（`:55`）/ `currentModelClass()`（`:59`）——**运行时用反射从泛型签名里拿到"VO 类、实体类"**，所以上面的 VO 方法**不用你传 Class 参数**。

**为什么需要"VO 转换"？** 因为**实体（Entity）直接返回给前端往往不合适**：

- 实体含**敏感字段**（密码、内部标记）；
- 前端要的**字段名/结构**和表不一样（要 `deptName` 而不是 `deptId`）；
- 关联查询的**附加字段**（不在表里）。

**所以标准做法是"Entity 查库 → VO 出参"**。`BaseMapperPlus` 把这一步**封装成一行**（`selectVoPage`）。

> ⚠️ **坦白点（`selectVoById` 不自动加数据权限）**：`BaseMapperPlus` 的 VO 方法内部调的是**普通 `selectById`/`selectList`**——所以**"这条 SQL 有没有租户/数据权限"，取决于它底层走的方法有没有被拦到**。单条 `selectById` 一般**不加数据权限**（第 4 章 §4.1 讲过：单条查询别标 `@DataPermission`），**但会自动加租户**（除非 `@InterceptorIgnore`）。**心里要清楚"哪些查询是安全的"。**

### 4.7 补全：`PlusDataPermissionInterceptor`（第 4 章的"本体"）

第 4 章讲了 `PlusDataPermissionHandler`（**干活的人**），**没讲它被谁调**。就是它——

`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/interceptor/PlusDataPermissionInterceptor.java:46`：

```java
public class PlusDataPermissionInterceptor extends JsqlParserSupport implements InnerInterceptor {  // :46
    private final PlusDataPermissionHandler dataPermissionHandler = new PlusDataPermissionHandler();  // :48

    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, ...) {   // :51 ★ 查询前
        if (InterceptorIgnoreHelper.willIgnoreDataPermission(ms.getId())) {  // :53 ★ 先看有没有 @InterceptorIgnore
            return;
        }
        if (dataPermissionHandler.isInvalid(ms.getId())) {                   // :57 没有 @DataPermission 就不管
            return;
        }
        PluginUtils.MPBoundSql mpBs = PluginUtils.mpBoundSql(boundSql);
        mpBs.sql(parserSingle(mpBs.sql(), ms.getId()));                      // :62 ★ 解析并改写 SQL
    }

    @Override
    public void beforePrepare(StatementHandler sh, ...) {                    // :66 更新/删除前
        if (sct == SqlCommandType.UPDATE || sct == SqlCommandType.DELETE) {  // :70
            mpBs.sql(parserMulti(mpBs.sql(), ms.getId()));                   // :75
        }
    }

    @Override
    protected void processSelect(Select select, ...) {                       // :80 处理 SELECT
        // PlainSelect → setWhere；SetOperationList（UNION）→ 逐个子查询 setWhere
    }

    @Override
    protected void processUpdate(Update update, ...) {                       // :92 处理 UPDATE
        Expression sqlSegment = dataPermissionHandler.getSqlSegment(update.getWhere(), (String) obj, false);
        if (null != sqlSegment) { update.setWhere(sqlSegment); }             // :95
    }

    @Override
    protected void processDelete(Delete delete, ...) {                       // :100 处理 DELETE
        Expression sqlSegment = dataPermissionHandler.getSqlSegment(delete.getWhere(), (String) obj, false);
        if (null != sqlSegment) { delete.setWhere(sqlSegment); }             // :103
    }
}
```

**这里补上了三块第 4 章没讲的拼图：**

1. **它继承 `JsqlParserSupport`**（`:46`）——**这就是"SQL 被解析成语法树再改写"的入口**。`parserSingle` / `parserMulti`（`:62`/`:75`）是 JSqlParser 提供的解析方法。**第 4 章说的"JSqlParser 改写"就发生在这几行。**
2. **`beforeQuery` vs `beforePrepare`**（`:51`/`:66`）：**查询走 `beforeQuery`，更新/删除走 `beforePrepare`**（`beforePrepare` 里只处理 `UPDATE`/`DELETE`，不处理 `INSERT`）。所以数据权限**对 SELECT/UPDATE/DELETE 都生效**，不只是查询。
3. **`processSelect` 处理 `UNION`**（`:84-87`）：如果 SQL 是 `UNION`（集合运算，`SetOperationList`），**它会遍历每个子查询分别加条件**——**这是"复杂 SQL 也不漏"的细节**。

**另外 `PlusDataPermissionInterceptor` 是被 `new` 出来的**（`MybatisPlusConfig.java:62-64`），不是 Spring Bean——**所以它和 `CommonTenantHandler` 一样，拿不到容器注入，只能用 `new` + 静态单例/内部字段。**（第 6 章 §2.6 讲过同样的原因。）

### 4.8 动手验证（建议真的跑一遍）

**验证 1：看表里到底有哪些"公共字段"**

```sql
SELECT ID, CREATE_BY, CREATE_TIME, UPDATE_BY, UPDATE_TIME, TENANT_ID, DEL_FLAG
FROM contract_draft ORDER BY CREATE_TIME DESC LIMIT 5;
```

预期：能看到 `CREATE_BY`、`CREATE_TIME`、`UPDATE_BY`、`UPDATE_TIME`、`TENANT_ID`、`DEL_FLAG`，且 `ID` 是 19 位左右的雪花 ID（不是 1、2、3 自增）。

**验证 2：新增一条草稿，看四个字段是否自动填**

1. 本项目 `mybatis-plus.configuration.logImpl` 是 `StdOutImpl`（`application.yml:239`），**控制台默认就会打印 SQL**。若你的环境关掉了，再临时加：
   ```yaml
   logging:
     level:
       com.ruoyi.contract.mapper: debug
   ```
2. 在页面上新增一条合同草稿（后端接口是 `POST /contract/contractDraft`，`ContractDraftController.java:77/:180`）。
3. 看日志里的 `==>  Preparing: INSERT INTO contract_draft ...`，预期列里包含 `ID`、`CREATE_BY`、`CREATE_TIME`、`UPDATE_BY`、`UPDATE_TIME`、`TENANT_ID`，且 `TENANT_ID` 的值来自当前租户。
4. 再查库：
   ```sql
   SELECT ID, CREATE_BY, CREATE_TIME, UPDATE_BY, UPDATE_TIME, TENANT_ID, DEL_FLAG
   FROM contract_draft ORDER BY CREATE_TIME DESC LIMIT 1;
   ```
   预期：`CREATE_BY` 是当前登录用户名，`CREATE_TIME`/`UPDATE_TIME` 是刚才的时间，`DEL_FLAG='0'`（以实体注释为准，`ContractDraft.java:139-142`）。

**验证 3：断点看填充顺序**

- `CreateAndUpdateMetaObjectHandler.java:43`：看 `createTime` 进来时是 null 还是已有值。
- `CreateAndUpdateMetaObjectHandler.java:47`：看 `createBy` 是否为空、`getLoginUsername()` 返回谁。
- `CreateAndUpdateMetaObjectHandler.java:66`：改一次数据，看 `updateTime` 是否被**强制**覆盖（即使你传了旧值）。

**验证 4：逻辑删除是不是"假删"**

在页面上删除刚建的草稿（`DELETE /contract/contractDraft/{id}`，`ContractDraftController.java:213`），然后：

```sql
SELECT ID, DEL_FLAG FROM contract_draft WHERE ID = <被删除的ID>;
```

预期：**行还在**，`DEL_FLAG` 变成 **`2`**（全局 `logicDeleteValue: 2`，`application.yml:254`）——注意 `ContractDraft` 实体注释写的是 `1`（`ContractDraft.java:139-142`），建表注释写的是 `2`（`zxcm_init.sql:1104`），**以全局配置和实际运行值为准**；列表接口再也查不到它。

**验证 5：确认真没有"全表更新保护"**

```bash
grep -n "BlockAttackInnerInterceptor" ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/MybatisPlusConfig.java
```

预期：只在末尾注释里出现（`:111`），**没有 `addInnerInterceptor(...)` 注册**——所以 §4.1 的坦白点成立。

---

## 五、一个操作的完整生命流程：一次"新增"的完整链路

以「张三新增一条合同草稿」（`POST /contract/contractDraft` → `baseMapper.insert(draft)`）为例，看清**所有"支撑"如何在一次插入里协同**：

```
Controller 收到 BO → Service 转成 Entity（ContractDraft）→ baseMapper.insert(draft)
  │  （此刻 draft 里：id=null, createBy=null, createTime=null, updateTime=null）
  │
  ▼ ┌─ ① 主键生成（MybatisPlusConfig.java:97 的 idGenerator）──────────────
    │   为 draft.id 生成雪花 ID（本机 IP 推算机器位 + 时间戳 + 序列号）
    └────────────────────────────────────────────────────────────────
  │
  ▼ ┌─ ② 字段自动填充（CreateAndUpdateMetaObjectHandler.insertFill :39）─
    │   createTime = updateTime = new Date()
    │   createBy = updateBy = getLoginUsername()  → 张三
    └────────────────────────────────────────────────────────────────
  │
  ▼ ┌─ ③ SQL 被四道插件依次改写（MybatisPlusConfig.java:45）─────────────
    │   原始：INSERT INTO contract_draft (id, name, ...) VALUES (?, ?, ...)
    │   ① 租户插件：VALUES 里补上 tenant_id = '00000000'           第6章
    │   ② 数据权限插件：INSERT 不处理（只处理 SELECT/UPDATE/DELETE）：51
    │   ③ 分页插件：INSERT 不处理
    │   ④ 乐观锁插件：INSERT 不处理（只在 UPDATE 加 version 条件；项目暂无 @Version 字段）
    └────────────────────────────────────────────────────────────────
  │
  ▼ MySQL 执行 → 一行落库（含雪花 ID、创建人、创建时间、租户）
  │
  ▼ 返回：影响行数 1（Service 判断 > 0 即成功）
```

**对照"查询"再看一遍**（`selectPage`）：

| 插件 | INSERT 时 | SELECT 时 |
|---|---|---|
| 租户 | 补 `tenant_id` 到 VALUES | 补 `tenant_id` 到 WHERE |
| 数据权限 | 不处理 | **补数据权限到 WHERE**（第 4 章） |
| 分页 | 不处理 | **包 `LIMIT` + 生成 count** |
| 乐观锁 | 不处理 | 不处理（只 UPDATE；项目暂无 `@Version` 字段，实际未启用） |
| 自动填充 | 填 4 个字段 | 不填 |
| 雪花 ID | 生成主键 | 不生成 |

**一张表看懂"不同操作触发哪些支撑"**——**这就是"ORM 支撑"的全景。**

---

## 六、面试官高频问题 + 答案（企业级视角）

**Q1：你们 MyBatis-Plus 配了哪些插件？什么顺序？**

四道（`MybatisPlusConfig.java:44-57`）：**① `TenantLineInnerInterceptor`（多租户）→ ② `PlusDataPermissionInterceptor`（数据权限）→ ③ `PaginationInnerInterceptor`（分页）→ ④ `OptimisticLockerInnerInterceptor`（乐观锁）**。**顺序关键**：改条件的（租户、数据权限）**必须在分页之前**，否则分页的 count 会漏算条件、总数对不上。

**Q2：`create_by` / `create_time` 怎么自动填的？**

实体字段标 `@TableField(fill = ...)`（`BaseEntity.java:48/55/62/69`），配合容器里的 `CreateAndUpdateMetaObjectHandler`（`MybatisPlusConfig.java:88` 注册）。它实现 `MetaObjectHandler` 的 `insertFill` / `updateFill`，**在插入/更新时把值写进去**。**业务代码不用管**。

**Q3：自动填充时如果调用方已经设了值怎么办？**

**`insert` 时"已有值优先"**（`CreateAndUpdateMetaObjectHandler.java:43/:47`）——比如数据迁移想保留原始创建人，就尊重它。**`update` 时 `updateTime` 强制刷新**（`:66`，注释"不管为不为空"），但 `updateBy` 只在能取到登录名时才覆盖（`:69`）。

**Q4：主键用什么？为什么不用自增？**

**雪花 ID**（`MybatisPlusConfig.java:97`）。自增在**分库/集群下会冲突**，UUID **无序、索引差**。雪花 ID **全局唯一（靠机器位）+ 大体有序**。**本项目的关键细节**：用 `NetUtil.getLocalhost()`（本机网卡 IP）**推算机器位**，实现"零配置集群不重复"（注释 `:94-95` 明说）。**边界**：容器化/多实例同机可能不稳，生产建议显式配 workerId。

**Q5：逻辑删除怎么实现的？有什么坑？**

字段标 `@TableLogic`（如 `SysRole.java:113`），**框架自动把 `DELETE` 变成 `UPDATE`、查询自动加 `del_flag` 条件**。**坑**：① **手写的原生 SQL（XML / `@Select`）不会自动加条件**，得自己写 `AND del_flag='0'`——**这是"列表里冒出已删数据"的常见原因**；② 表会越来越大；③ 唯一索引要带上 `del_flag`。**删除值由全局配置决定**：本项目 `logicDeleteValue: 2`、`logicNotDeleteValue: 0`（`application.yml:254-256`），但部分实体/建表注释写的是 `1` 或 `-1`，**以全局配置和运行值为准**。

**Q6：`BaseEntity` 和 `BaseMapperPlus` 是干什么的？**

`BaseEntity`（`:33`）抽公共字段（创建人/时间、`params` 等），**所有实体继承**；`BaseMapperPlus`（`:49`）在 `BaseMapper<T>` 上加**批量操作** + **VO 转换**（`selectVoPage` 等，`:225`）。**价值都是"消除重复"**——业务只写自己特有的东西。**注意**：`BaseEntity` 里**没有 `delFlag`**（逻辑删除各表自己声明），因为不是每张表都要逻辑删除。

**Q7：为什么要有 Entity → VO 的转换？**

**Entity 直接出参常常不合适**：含敏感字段、字段名和前端要的不一致、需要关联查询的附加字段。所以标准做法是 **Entity 查库 → VO 出参**。`BaseMapperPlus.selectVoPage`（`:225`）把这步封装成一行，**靠泛型反射 `currentVoClass()`（`:55`）自动知道转成哪个 VO**。

**Q8：数据权限拦截器在哪一层改写 SQL？**

`PlusDataPermissionInterceptor`（`PlusDataPermissionInterceptor.java:46`）**继承 `JsqlParserSupport` 实现 `InnerInterceptor`**。`beforeQuery`（`:51`）拦查询、`beforePrepare`（`:66`）拦更新/删除（不处理 INSERT），用 `parserSingle`/`parserMulti` 把 SQL 解析成语法树，再在 `processSelect`（`:80`）/`processUpdate`（`:92`）/`processDelete`（`:100`）里把条件**塞进 WHERE**。**它甚至处理 `UNION`**（`:84-87` 遍历每个子查询）。**这就是第 4 章"JSqlParser 无痕改写"的确切位置。**

**Q9：分页插件有什么配置？安全吗？**

`setMaxLimit(-1L)`（`:72`）**不限单页上限**、`setOverflow(true)`（`:74`）**页码越界回到最后一页**。**坦白说"不限上限"有 DoS 风险**（`pageSize` 传巨大值就能拉爆内存）——**更安全是设合理上限**。**另一个坦白点**：注释里提到 `BlockAttackInnerInterceptor`（防全表更新/删除）和 `IllegalSQLInnerInterceptor`，**但本项目没注册**（`:44-57`），**"防全表误操作"这层没开**。

### 本章答辩卡（追问级）

**① 问：雪花 ID 用网卡 IP 推机器位，如果同一台机器起了两个后端实例，会出什么问题？**

**10 秒答案**：两个实例在**同一台机器**上会拿到**同一个机器位**（`NetUtil.getLocalhost()` 返回同一个 IP），同一毫秒内的序列号也各自从 0 开始，理论上可能生成重复 ID。更稳的做法是**显式配置不同的 `workerId`** 或用中心化发号器（如 Redis/号段）。本项目按单体部署设计，所以够用。

**证据**：`E:\Develop\project\zhaoxin\zhaoxin-contract\ruoyi-base\ruoyi-framework\src\main\java\com\ruoyi\framework\config\MybatisPlusConfig.java:94-100`（注释与 `new DefaultIdentifierGenerator(NetUtil.getLocalhost())`）。

**② 问：`insertFill` 为什么先判断 `instanceof BaseEntity`，不直接给所有实体填？**

**10 秒答案**：因为只有继承 `BaseEntity` 的实体才保证有 `createBy/createTime/updateBy/updateTime` 这几个字段；别的实体（比如只映射部分列的对象）没有这些属性，强转会抛异常。**自动填充只对"公共字段基类"生效**，这是显式约定。

**证据**：`E:\Develop\project\zhaoxin\zhaoxin-contract\ruoyi-base\ruoyi-framework\src\main\java\com\ruoyi\framework\handler\CreateAndUpdateMetaObjectHandler.java:41-42`；`E:\Develop\project\zhaoxin\zhaoxin-contract\ruoyi-base\ruoyi-common\src\main\java\com\ruoyi\common\core\domain\BaseEntity.java:33`（`implements Serializable`，四个公共字段在 `:48-70`）。

**③ 问：`BaseMapperPlus.selectVoById()` 查一条数据，会自动加数据权限和租户条件吗？**

**10 秒答案**：**租户会自动加**（除非 Mapper/方法标了 `@InterceptorIgnore`）；**数据权限一般不会**——因为它底层调 `selectById`，而这个方法上通常没有 `@DataPermission`，`PlusDataPermissionInterceptor.beforeQuery` 在 `isInvalid` 时直接返回（`PlusDataPermissionInterceptor.java:57-59`）。所以单条查询的"行级可见性"要单独考虑。

**证据**：`E:\Develop\project\zhaoxin\zhaoxin-contract\ruoyi-base\ruoyi-common\src\main\java\com\ruoyi\common\core\mapper\BaseMapperPlus.java:150-163`（selectVoById → selectById）；`E:\Develop\project\zhaoxin\zhaoxin-contract\ruoyi-base\ruoyi-framework\src\main\java\com\ruoyi\framework\interceptor\PlusDataPermissionInterceptor.java:57-59`（无注解跳过）；`E:\Develop\project\zhaoxin\zhaoxin-contract\ruoyi-base\ruoyi-framework\src\main\java\com\ruoyi\framework\config\MybatisPlusConfig.java:47-49`（租户插件全局生效）。

---

## 七、选型对比：ORM 支撑的技术选型

### 7.1 ORM 框架

| 框架 | 特点 | 自动填充 | 逻辑删除 | 多租户 |
|---|---|---|---|---|
| **MyBatis-Plus（本项目）** | MyBatis 增强，SQL 可控 | **内置** | **内置** | **内置插件** |
| 原生 MyBatis | 最灵活，全手写 | 自己实现 | 自己实现 | 自己实现 |
| JPA / Hibernate | 全自动，对象化 | `@PrePersist` | `@Where` | 需第三方 |

**结论**：**国内后台系统几乎清一色 MyBatis-Plus**——**"SQL 可控"（能优化、能拦截改写）比 JPA 的"全自动"更适合复杂业务**。**本项目选它，正是看中"能拦截改写 SQL"（租户 + 数据权限全靠这个）。**

### 7.2 主键方案

| 方案 | 集群唯一 | 有序 | 泄露业务量 | 本项目 |
|---|---|---|---|---|
| 自增 | ✗（分库冲突） | ✔ | ✔ | ✗ |
| UUID | ✔ | ✗（索引差） | ✗ | ✗ |
| **雪花 + 网卡机器位（本项目）** | ✔ | 大体有序 | ✗ | **✔** |

### 7.3 删除策略

| 策略 | 可追溯 | 表体积 | 关联完整 | 本项目 |
|---|---|---|---|---|
| 物理删除 | ✗ | 小 | ✗ | ✗ |
| **逻辑删除（本项目）** | **✔** | 大 | **✔** | **✔**（多数表） |
| 归档表 | ✔ | 主表小 | 需处理 | ✗ |

**结论**：**合同、审批这类"要可追溯"的数据，逻辑删除是必然选择。**

---

## 延伸阅读（网上优秀资料）

- **MyBatis-Plus 官方文档**：<https://baomidou.com/> —— 核心功能（CRUD、条件构造器、`BaseMapper`）从这里入门。
- **MyBatis-Plus · 自动填充**：<https://baomidou.com/guides/auto-fill-field/> —— `MetaObjectHandler` + `@TableField(fill=...)` 的官方示例（本项目 `CreateAndUpdateMetaObjectHandler` 的原型）。
- **MyBatis-Plus · 逻辑删除**：<https://baomidou.com/guides/logic-delete/> —— `@TableLogic` 的用法与坑。
- **MyBatis-Plus · 主键策略**：<https://baomidou.com/guides/key-generate/> —— 雪花 ID / `IdentifierGenerator` 的说明。
- **雪花算法（Snowflake）原理**：搜「雪花算法 原理 机器位」，理解"时间戳 + 机器 ID + 序列号"的 64 位结构（本项目网卡取机器位的底层原理）。

---

## 本章小结 & 下一步

- **ORM 支撑 = 四块**：SQL 自动改写（4 插件）、字段自动填充（`MetaObjectHandler`）、主键生成（雪花 ID）、基类封装（`BaseEntity`/`BaseMapperPlus`）。
- **四道插件顺序**（`MybatisPlusConfig.java:47/51/53/55`）：**租户 → 数据权限 → 分页 → 乐观锁**；改条件的必须在分页前。
- **自动填充**：字段标 `@TableField(fill=...)`，`CreateAndUpdateMetaObjectHandler` 填值；**insert 已有值优先、update 强制刷 `updateTime`**。
- **雪花 ID**：`DefaultIdentifierGenerator(NetUtil.getLocalhost())`——**用网卡 IP 推算机器位**，零配置防集群重复。
- **逻辑删除**：`@TableLogic`；**手写 SQL 不会自动加 `del_flag` 条件**（大坑）。
- **`PlusDataPermissionInterceptor`**（`:46`）是第 4 章"改写 SQL"的本体：继承 `JsqlParserSupport`，`beforeQuery`/`beforePrepare` 拦，`processSelect/Update/Delete` 改，连 `UNION` 都处理。
- **坦白点**：① 分页 `maxLimit=-1` **不限单页**（DoS 风险）；② **`BlockAttackInnerInterceptor` 注释提到但没注册**（无全表误操作保护）；③ 雪花 ID 网卡方案在**容器/多实例**场景不稳；④ 未登录时 `create_by` **可能为 null**。

**下一章（第 8 章 · 操作日志与登录日志）**：本章讲的是"数据怎么进数据库"，**第 8 章讲"人在系统里干了什么怎么被记下来"**——`@Log` 注解 + `LogAspect` 切面怎么把每次操作写进 `sys_oper_log`；登录成功/失败怎么进 `sys_logininfor`；以及**异步记录 + 敏感字段脱敏**这些工程细节。
