# 附录 B · 零基础精读：一次请求的完整流程

> **这篇文档写给谁**：完全没有前端基础、看到 `process.env.xxx`、`[xxx]:`、`.then()` 就发怵的读者。
>
> **目标**：跟着「编号规则列表」这一次真实请求，从页面加载一路走到后端返回、页签刷新，**每一步都讲清楚「谁做的、在哪做的、为什么这么做」**。第二部分再把沿途出现的 15 个前端知识点单独补课。
>
> **配套章节**：[第 0 章 §六 Q10](第0章-前端全景与运行机制.md)（代理的一句话答案）、[第 1 章 §五](第1章-请求层与登录鉴权.md)（请求层地图）。本篇是它们的**零基础展开版**，不是重复：正文讲「流程」，第二部分讲「概念」。
>
> **行号约定**：前端文件相对 `zxcm-ui/`，后端文件相对仓库根。行号以当前仓库为准。

---

## 这篇怎么读

**第一步（必读）**：先读 [〇、先看结论](#〇先看结论这条请求的一生) → [一、12 步逐行走查](#一12-步逐行走查) → [二、URL 演进表](#二url-演进表一张表看懂前缀怎么来怎么没)。这三节是同一条主线的三种看法：一张图、一次走查、一张表。

**第二步（正文先跳过概念细节）**：正文里标着「知识点 N」的括号，第一遍只需知道「这里有个概念」，不用停下来。等 12 步走完，再回第二部分按编号补课；第二部分每个知识点都是独立小节，可以跳着读。§3.5、§3.6 讲的是生产环境与选型，第一遍也可以先跳过。

**第三步（一定要动手）**：读完主线，照 [五、自己动手验证](#五自己动手验证) 在浏览器 DevTools 里真的跟一次请求，并把 §5.1 的「证据卡」字段抄下来。看懂十遍，不如自己抄一遍。

**卡住时查哪里**：

| 卡住的词 | 先看本篇 | 想再深入（见《Vue2 从 0 到精通》） |
|---|---|---|
| `created` / `data()` / `this` | 知识点 13 | 第 3 章 §2、§7.3；第 0 章 §5.5 |
| `.then()` | 知识点 9 | 第 0 章 §5.11 |
| `process.env` | 知识点 7 | 第 7 章 §5.2~5.4 |
| `[表达式]:` | 知识点 8 | 第 0 章 §5.6（对象） |
| axios / 拦截器 | 知识点 6、10 | 第 7 章 §7.2~7.4 |
| 代理 / 前缀匹配 | 知识点 4、12 | 第 7 章 §4.3、第 11 章 §七 坑 28 |
| 同源 / 跨域 | 知识点 2 | 第 7 章 §4.3 |
| 页面数据不更新 | 知识点 13 | 第 10 章 §2、第 11 章 §二 坑 2 |
| DevTools 不会看 | §5.1 | 第 11 章 §9.2 |

---

## 目录

**从这里开始**
- [这篇怎么读](#这篇怎么读)

**第一部分：流程**
- [〇、先看结论](#〇先看结论这条请求的一生)
- [一、12 步逐行走查](#一12-步逐行走查)
- [二、URL 演进表](#二url-演进表一张表看懂前缀怎么来怎么没)
- [三、关键问题专答](#三关键问题专答)
- [四、常见误解清单](#四常见误解清单)
- [五、自己动手验证](#五自己动手验证)
- [六、动手改一改](#六动手改一改)
- [七、答辩口径](#七答辩口径)

**第二部分：前端知识补课**
- [知识点 1：URL 的组成](#知识点-1url-的组成)
- [知识点 2：同源策略与 CORS](#知识点-2同源策略与-cors)
- [知识点 3：Node.js、npm、package.json](#知识点-3nodejsnpmpackagejson)
- [知识点 4：Vue CLI / webpack / devServer / 代理中间件](#知识点-4vue-cli--webpack--devserver--代理中间件)
- [知识点 5：ES Module 与 `@` 别名](#知识点-5es-module-与--别名)
- [知识点 6：axios 与「请求配置对象」](#知识点-6axios-与请求配置对象)
- [知识点 7：`.env` 环境变量与编译期替换](#知识点-7env-环境变量与编译期替换)
- [知识点 8：JS 对象与计算属性名 `[expr]`](#知识点-8js-对象与计算属性名-expr)
- [知识点 9：Promise 与 `.then()`](#知识点-9promise-与-then)
- [知识点 10：拦截器](#知识点-10拦截器)
- [知识点 11：相对 URL 如何被补全](#知识点-11相对-url-如何被补全)
- [知识点 12：代理、前缀匹配、pathRewrite](#知识点-12代理前缀匹配pathrewrite)
- [知识点 13：Vue 基础](#知识点-13vue-基础)
- [知识点 14：Cookie 与 token](#知识点-14cookie-与-token)
- [知识点 15：JSON、HTTP 状态码与业务 code](#知识点-15jsonhttp-状态码与业务-code)
- [术语总表](#术语总表)
- [小结与下一步](#小结与下一步)
- [本章答辩卡：3 组追问级问答](#本章答辩卡3-组追问级问答)

---

# 第一部分：流程

## 〇、先看结论：这条请求的「一生」

我们要追踪的是：**打开「编号规则」页面后，页面顶部的页签数据是怎么来的**。

> 小提醒：这个页面不是常见的 `el-table` 列表页，而是用 `el-tabs` 把每条编号规则渲染成一个页签（`index.vue:8-13`）。本文后面为了贴合 RuoYi 的通用说法，有时仍会写「列表/表格」，你可以统一理解成「这一页要展示的数据」。

```
浏览器里的 Vue 页面
  │  ① created() 自动执行 → getList()
  │  ② listBillRule(queryParams)              api 函数只负责"填单"
  │  ③ axios 实例补上前缀、拦截器加 token/拼参数
  ▼
浏览器发出请求：
  GET http://localhost/dev-api/conf/billRule/list?pageNum=1&pageSize=10
  │  ④ 同源（页面也在 localhost:80），浏览器放行
  ▼
webpack-dev-server（Node 进程，监听 80）
  │  ⑤ 用代理表匹配 '/dev-api' → 命中
  │  ⑥ pathRewrite 去掉前缀 → /conf/billRule/list?...
  │  ⑦ 转发到 target: http://localhost:8080
  ▼
Spring Boot（8080）
  │  ⑧ @RequestMapping("/conf/billRule") + @GetMapping("/list")
  │  ⑨ 查库，返回 { code:200, msg:"查询成功", rows:[...], total:实际条数 }
  ▼
原路返回
  │  ⑩ 响应拦截器：看 code，最后 return res.data
  │  ⑪ 页面拿到 { code, msg, rows, total }，读 response.rows
  ▼
Vue 更新 this.billRuleList → 页签重新渲染
```

**参与者和职责一览：**

| 角色 | 跑在哪 | 干什么 | 代码位置 |
|---|---|---|---|
| Vue 页面 | 浏览器 | 触发请求、接收数据、渲染页签 | `src/views/conf/billRule/index.vue` |
| api 函数 | 浏览器 | 把「接口路径 + 方法 + 参数」整理成请求配置 | `src/api/conf/billRule.js:13-19` |
| axios 实例 | 浏览器 | 拼 `baseURL`、走拦截器、真正发 HTTP | `src/utils/request.js:19-24` |
| 浏览器 | 本机 | 解析相对地址、做同源检查、发请求 | 浏览器自身行为 |
| webpack-dev-server | 本机 Node 进程 | 提供页面文件；把 `/dev-api` 请求转发到 8080 | `vue.config.js:31-46` |
| Spring Boot | 本机 8080 | 接收接口请求、查库、返回 JSON | `ConfBillRuleController.java:66,76` |

> **一句话记住**：浏览器只认识 `localhost:80`；`8080` 是 Node 进程在背后转发的，浏览器从头到尾没见过它。

---

## 一、12 步逐行走查

### 第 1 步：页面加载，自动发起查询

`src/views/conf/billRule/index.vue:265-267`：

```js
created() {
  this.getList()
},
```

`created` 是 Vue 组件的**生命周期钩子**：组件实例创建完成、还没显示到屏幕上时，Vue 会自动调用它一次。所以不是用户点了什么，而是**一进页面就查**。生命周期钩子的完整顺序见《Vue2 从 0 到精通》第 3 章 §7.2。

`getList()` 在 `index.vue:291-299`：

```js
getList() {
  this.loading = true
  listBillRule(this.queryParams).then((response) => {
    this.billRuleList = response.rows
    this.total = response.total
    ...
  })
}
```

- `this.loading = true`：把 `loading` 置为 `true`。**注意**：本页模板并没有把 `loading` 绑定到任何元素上，所以实际不会出现转圈遮罩——这是从 RuoYi 列表页模板沿用下来的变量（`index.vue:198` 声明）。
- `listBillRule(this.queryParams)`：把查询条件交给 api 函数。
- `.then((response) => {...})`：请求是异步的，**先发出去，等响应回来再执行这里**（知识点 9；更基础的解释见《Vue2 从 0 到精通》第 0 章 §5.11）。

`this.queryParams` 定义在 `index.vue:219-230`（整个 `data()` 从 `index.vue:192` 开始；本节用到的 `loading`、`total`、`billRuleList` 分别声明在 `index.vue:198`、`:208`、`:210`）：

```js
queryParams: {
  pageNum: 1,
  pageSize: 10,
  name: undefined,      // 用户没填的条件，值是 undefined
  code: undefined,
  rule: undefined,
  thisNumber: undefined,
  startNumber: undefined,
  numberLength: undefined,
  numberType: undefined,
  thisNumberDate: undefined
}
```

> **易错点**：`undefined` 表示「没有值」。后面拼 URL 时，`undefined` 的字段会被**跳过**，不会变成字符串 `"undefined"`。

---

### 第 2 步：`listBillRule` 从哪来

页面顶部 `index.vue:171-177`：

```js
import {
  listBillRule,
  getBillRule,
  delBillRule,
  addBillRule,
  updateBillRule
} from '@/api/conf/billRule'
```

这是 ES Module 的**具名导入**（知识点 5）：

- `@` 是别名，等于 `src/`（`vue.config.js:57-59`），所以 `@/api/conf/billRule` 就是 `src/api/conf/billRule.js`。
- `import { 名字 } from '文件'`：从那个文件里取出同名的导出函数。
- 被导入的函数可以直接调用，就像它定义在当前文件里。

---

### 第 3 步：api 函数只是在「填一张请求单」

`src/api/conf/billRule.js:13-19`：

```js
export function listBillRule(query) {
  return request({
    url: "/conf/billRule/list",
    method: "get",
    params: query,
  });
}
```

它**自己不发请求**，只是构造一个普通的 JS 对象（请求配置），交给 `request`：

| 字段 | 值 | 含义 |
|---|---|---|
| `url` | `/conf/billRule/list` | 后端接口路径 |
| `method` | `get` | HTTP 方法 |
| `params` | `query`（即 `queryParams`） | 查询参数，最终会变成 `?a=1&b=2` |

关键观察：**`url` 里没有域名、没有端口、没有 `/dev-api`**。只有后端 Controller 自己声明的路径。这是整条链路能跨环境复用的前提。

---

### 第 4 步：`request` 是什么——预先创建好的 axios 实例

`src/utils/request.js` 开头：

```js
import axios from 'axios'                       // :1  引入 axios 库
...
const service = axios.create({                  // :19 创建实例
  baseURL: process.env.VUE_APP_BASE_API,        // :21 公共前缀
  timeout: 10000                                // :23 超时 10 秒
})
...
export default service                          // :172 默认导出
```

`axios.create(...)` 创建的 `service` 是一个**配置好默认值的请求函数**。之后每次 `service(config)` 都会自动带上：

- `baseURL = '/dev-api'`
- `timeout = 10000`

`billRule.js:1` 的 `import request from "@/utils/request"` 拿到就是这个默认导出的 `service`。

> 为什么不在每个 api 文件里都写一遍 `baseURL`？因为统一封装一处，改环境、加请求头、加拦截器都只改一个地方（知识点 6、10）。

---

### 第 5 步：`process.env.VUE_APP_BASE_API` 为什么等于 `'/dev-api'`

这是零基础读者最容易卡住的地方，拆成三件事：

**（1）值来自 `.env.development`**

`zxcm-ui/.env.development:8`：

```
VUE_APP_BASE_API = '/dev-api'
```

`package.json:8` 的 `dev` 脚本是：

```
"dev": "set NODE_OPTIONS=--openssl-legacy-provider && vue-cli-service serve"
```

`vue-cli-service serve` 默认以 **development 模式**启动，Vue CLI 会读取 `.env.development`，把 `键=值` 放进 `process.env`。读的时候会**去掉等号两边空格、去掉最外层引号**，所以值是字符串 `/dev-api`。

**（2）只有 `VUE_APP_` 开头的变量会给前端用**

Vue CLI 规定：只有以 `VUE_APP_` 开头的环境变量才会被注入到浏览器代码里。这是安全边界——机器的 `PATH`、数据库密码等不会被前端代码看见。

**（3）替换发生在编译时，不是运行时**

浏览器里根本没有 `process` 这个对象（它是 Node.js 的全局变量）。你写的：

```js
baseURL: process.env.VUE_APP_BASE_API,
```

在 `npm run dev` 启动时，webpack 会把它**直接替换成字符串字面量**。浏览器实际收到的是：

```js
baseURL: '/dev-api',
```

所以「浏览器不认识 `process.env`」并不是问题——这段代码到浏览器之前已经被改写了（知识点 7）。

> **易混淆**：`vue.config.js:37` 也写了 `process.env.VUE_APP_BASE_API`，但那个文件是 **Node 进程直接执行**的，读的是真实的运行时环境变量。同一个写法，两种机制。

---

### 第 6 步：请求拦截器先过一遍「安检」

`request.js:27` 注册了请求拦截器。下面是和本次请求有关的完整代码（行尾注释标的是真实行号）：

```js
service.interceptors.request.use(config => {                  // :27 注册「请求发出前」的关卡
  const isToken = (config.headers || {}).isToken === false    // :29 调用方可显式关闭 token
  const isRepeatSubmit = (config.headers || {}).repeatSubmit === false  // :31
  if (getToken() && !isToken) {                               // :32 Cookie 里有 token 才加
    config.headers['token'] = 'Bearer ' + getToken()          // :33 写进请求头
  }
  if (config.method === 'get' && config.params) {             // :36 只处理 GET 的 params
    let url = config.url + '?' + tansParams(config.params);   // :37 拼查询串
    url = url.slice(0, -1);                                   // :38 去掉末尾的 &
    config.params = {};                                       // :39 清空，防 axios 再拼一次
    config.url = url;                                         // :40 用拼好的地址覆盖
  }
  // :42-64 是 POST/PUT 的防重复提交，本次 GET 不进入
  return config                                               // :65 必须把 config 交回去
}, error => {
  return Promise.reject(error)                                // :67 注册/执行失败时抛出
})
```

**逐行读：这个拦截器到底做了什么**

1. `service.interceptors.request.use(fn)` 不是立刻执行 `fn`，而是把 `fn` **登记**到 axios 实例上；之后每一次用 `service(...)` 发请求，axios 都会先调用它。
2. `config` 是这次请求的「配置对象」，`url`、`method`、`params`、`headers` 全在里面。拦截器拿到的是**同一个对象**的引用，改它就是改这次请求。
3. `:29` 的 `isToken`：默认拿到 `undefined`，`undefined === false` 不成立，所以默认**不跳过** token；只有某个接口显式写 `headers: { isToken: false }` 才会跳过。
4. `getToken()` 去 Cookie 里读 `Admin-Token`（`src/utils/auth.js:5-7`）。
5. `:33` 是 token 真正进入请求头的瞬间：键名 `token`，值是 `'Bearer ' + 令牌`。注意 `Bearer` 后面有一个空格，少了它后端解析会失败。
6. `:36` 只对 **GET** 且带了 `params` 的请求生效；POST/PUT 的参数在 `data` 里，不会走这段。
7. `:37` 调 `tansParams` 把对象拼成 `键=值&键=值&`，`null` 和 `undefined` 字段被跳过。
8. `:38` 的 `slice(0, -1)` 去掉 `tansParams` 多留的最后一个 `&`。
9. `:39` 把 `config.params` 清空成 `{}`；否则 axios 之后还会把原 `params` 再拼一次，URL 里出现两遍参数。
10. `:65` 必须 `return config`。忘了 return，axios 下一步拿到 `undefined`，请求会直接失败。
11. 第二个参数 `error => Promise.reject(error)` 处理的是拦截器本身出错的情况，和业务错误无关。

「拦截器」= 请求真正交给浏览器之前，必经的一段函数（知识点 10；拦截器的整体用法见《Vue2 从 0 到精通》第 7 章 §7.3）。本项目的请求拦截器做两件与本次请求相关的事。

**（1）加 token（`request.js:32-34`）**

```js
if (getToken() && !isToken) {
  config.headers['token'] = 'Bearer ' + getToken()
}
```

- `getToken()` 从 Cookie 里读 `Admin-Token`（`src/utils/auth.js:3-7`，`const TokenKey = 'Admin-Token'`）。
- 读到了就放进请求头 `token`，值形如 `Bearer xxxxx`。
- 后端 `application.yml:103` 配的正是 `token-name: token`，两边对得上。

**（2）把 GET 参数拼成查询串（`request.js:36-41`）**

```js
if (config.method === 'get' && config.params) {
  let url = config.url + '?' + tansParams(config.params);
  url = url.slice(0, -1);
  config.params = {};
  config.url = url;
}
```

`tansParams` 在 `src/utils/ruoyi.js:238-258`，规则是：

1. 遍历对象的每个键；
2. 值为 `null` / `undefined` 的跳过；
3. 其余拼成 `键=值&`；
4. 最后去掉末尾多余的 `&`。

拿第 1 步的 `queryParams` 走一遍：

| 键 | 值 | 拼进去吗 | 结果 |
|---|---|---|---|
| `pageNum` | `1` | 是 | `pageNum=1&` |
| `pageSize` | `10` | 是 | `pageSize=10&` |
| `name` | `undefined` | 否 | — |
| `code` | `undefined` | 否 | — |
| 其余 | `undefined` | 否 | — |

拼完是 `pageNum=1&pageSize=10&`，`slice(0, -1)` 去掉末尾 `&`，得到：

```
pageNum=1&pageSize=10
```

此时：

- `config.url` = `/conf/billRule/list?pageNum=1&pageSize=10`
- `config.params` = `{}`（清空，防止 axios 再拼一次）

> **说明**：axios 本身也能自动把 `params` 拼成查询串；这里手动拼是为了统一走 `tansParams`（它能处理嵌套对象）。这是本项目的写法选择，不是 axios 的硬性要求。

---

### 第 7 步：axios 把 `baseURL` 和 `url` 拼起来

axios 的拼接规则（axios 0.24.0，`combineURLs`）：

> 如果 `url` 不是完整地址（没有以「协议 + `//`」开头，例如 `http://`、`https://`），就执行
> `baseURL`（去掉末尾 `/`）+ `/` + `url`（去掉开头 `/`）。

代入：

```
baseURL = '/dev-api'
url     = '/conf/billRule/list?pageNum=1&pageSize=10'
────────────────────────────────────────────────────────
结果    = '/dev-api/conf/billRule/list?pageNum=1&pageSize=10'
```

**注意：这里仍然没有主机名和端口。** 它依然是一个以 `/` 开头的相对地址，接下来交给浏览器补全（知识点 11）。

---

### 第 8 步：浏览器补上「当前页面所在的服务器」

当前页面地址是 `http://localhost/`：

- 协议 `http`
- 主机 `localhost`
- 端口 `80`（http 默认端口，地址里省略）

开发服务器的端口在 `vue.config.js:12`：

```js
const port = process.env.port || process.env.npm_config_port || 80
```

请求地址以 `/` 开头，浏览器会把它接到**当前页面的 origin** 后面：

```
http://localhost/dev-api/conf/billRule/list?pageNum=1&pageSize=10
```

**同源检查**：

| 维度 | 页面 | 请求 | 是否一致 |
|---|---|---|---|
| 协议 | `http` | `http` | 是 |
| 主机 | `localhost` | `localhost` | 是 |
| 端口 | `80` | `80` | 是 |

三者一致 → 同源 → 浏览器直接发出，不做跨域拦截（知识点 2）。

> 如果这里直接写 `http://localhost:8080/...`，端口不同就是跨域，会走 CORS 流程。代理的意义之一就是让浏览器**根本不去发跨域请求**。

---

### 第 9 步：devServer 代理表匹配——重点中的重点

#### （1）`[process.env.VUE_APP_BASE_API]:` 是什么语法

`vue.config.js:35-44`：

```js
proxy: {
  [process.env.VUE_APP_BASE_API]: {
    target: `http://localhost:8080`,
    changeOrigin: true,
    pathRewrite: {
      ['^' + process.env.VUE_APP_BASE_API]: ''
    }
  }
}
```

**逐行读：这张代理表每一行在说什么**

1. `:35` `proxy: {`：这是 `devServer` 的代理表。表里每一项的格式都是「一个前缀 → 一个转发目标」；请求路径以某个键开头，就按那一项处理。
2. `:37` `[process.env.VUE_APP_BASE_API]: {`：这一项的前缀**不是写死的字符串**，而是方括号里算出来的值。`VUE_APP_BASE_API` 来自 `.env.development:8`，所以键最终是 `'/dev-api'`。
3. `:38` `target: \`http://localhost:8080\``：命中后往哪里转。协议、主机、端口都来自这一行；路径部分来自原请求。
4. `:39` `changeOrigin: true`：转发时把 HTTP 请求头里的 `Host` 改成目标地址的 host（`localhost:8080`）。不开的话，后端看到的 Host 还是浏览器原来的 `localhost`。
5. `:40-42` `pathRewrite`：转发前改写路径。键 `['^' + VUE_APP_BASE_API]` 算出来是字符串 `'^/dev-api'`，值是空串 `''`，合起来的意思是「把开头的 `/dev-api` 删掉」。
6. `:43-44` 两个 `}`：结束当前代理项和 `proxy` 对象。devServer 最终拿到的是一张普通对象，键就是 `'/dev-api'`。

方括号包住一个表达式，这是 JS 对象的**计算属性名**（computed property name，知识点 8；JS 对象的系统讲法见《Vue2 从 0 到精通》第 0 章 §5.6）：

```js
const key = 'abc'
const obj = { [key]: 1 }     // 等价于 { abc: 1 }
```

所以这两处：

```js
[process.env.VUE_APP_BASE_API]                 // → '/dev-api'
['^' + process.env.VUE_APP_BASE_API]           // → '^/dev-api'
```

算完之后，`vue.config.js` 交给 devServer 的，就是一张普通对象：

```js
proxy: {
  '/dev-api': {
    target: 'http://localhost:8080',
    changeOrigin: true,
    pathRewrite: {
      '^/dev-api': ''
    }
  }
}
```

**「匹配」靠的是算出来的键 `'/dev-api'`；方括号语法只负责让键名动态生成。** 这样写的好处是：代理前缀和 axios 的 `baseURL` 读同一个环境变量，改一处两边同步，不会出现「前端加了 `/api`、代理还写 `/dev-api`」的错位。

#### （2）代理表怎么拿来匹配请求

webpack-dev-server 收到请求后，看请求的**路径部分** `/dev-api/conf/billRule/list`，逐条和代理对象的键比较，规则是：

> **请求路径是否以这个键开头**（前缀匹配，类似路由前缀）。

`/dev-api/conf/billRule/list` 以 `/dev-api` 开头 → 命中。

不看域名、不看查询串、不做精确相等。如果配了多个键，按配置顺序依次比较，命中即止——所以更具体的前缀建议写在前面（本项目只有一个键，不涉及顺序问题）。

#### （3）命中后依次做三件事

**第一件：`pathRewrite` 去掉前缀**

`pathRewrite` 的键 `'^/dev-api'` 是一个正则字符串：

- `^` 表示「字符串开头」；
- 整条规则的意思是「把开头的 `/dev-api` 替换成空字符串」。

```
原路径：/dev-api/conf/billRule/list?pageNum=1&pageSize=10
改后：  /conf/billRule/list?pageNum=1&pageSize=10
```

**第二件：`target` 拼出目标地址**

```js
target: `http://localhost:8080`
```

协议 + 主机 + 端口来自 `target`，路径来自上一步改写后的结果：

```
http://localhost:8080/conf/billRule/list?pageNum=1&pageSize=10
```

**第三件：`changeOrigin: true` 改写 Host 头**

代理转发时，默认会保留浏览器原来的 `Host: localhost`。`changeOrigin: true` 会把它改成目标地址的 Host（`localhost:8080`）。有些后端会按 Host 做校验或生成绝对链接，打开它更稳妥。

#### （4）真正发请求的是 Node 进程，不是浏览器

这是理解代理的关键：

```
浏览器 ──请求 ①──> devServer（Node，localhost:80）
devServer（Node）──请求 ②──> Spring Boot（localhost:8080）
```

**这是两段独立请求。**

- 请求 ①：浏览器只跟自己同源的 devServer 说话，不涉及跨域；
- 请求 ②：Node 进程发起的服务端请求，**不受浏览器的同源策略约束**。

所以跨域问题消失了。后端也永远看不到 `/dev-api`——它在请求 ② 之前就被 `pathRewrite` 去掉了。

---

### 第 10 步：请求到达 Spring Boot

后端 `application.yml`：

- `:30` `port: 8080`
- `:33` `context-path: /`（没有额外路径前缀）

`zxcm-contract/src/main/java/com/ruoyi/conf/controller/ConfBillRuleController.java`：

```java
@RequestMapping("/conf/billRule")                 // :66 类级路径
public class ConfBillRuleController extends BaseController {

    @GetMapping("/list")                          // :76
    public TableDataInfo<ConfBillRuleVo> list(ConfBillRuleBo bo, PageQuery pageQuery) {   // :77
```

最终映射路径 = 类级 `/conf/billRule` + 方法级 `/list` = `/conf/billRule/list`，与前端发来的路径一致。

参数绑定：

- `pageNum=1&pageSize=10` → `PageQuery`（分页参数）
- 其他查询条件 → `ConfBillRuleBo`

**Controller 之后还有两跳（这一步才是真正查库）：**

1. `ConfBillRuleController.java:77-78`：`list(...)` 把参数交给 `iConfBillRuleService.queryPageList(bo, pageQuery)`。
2. `zxcm-contract/src/main/java/com/ruoyi/conf/service/impl/ConfBillRuleServiceImpl.java:93-100`：先按查询条件组装 `LambdaQueryWrapper`（`:94`），再调 `baseMapper.selectVoPage(pageQuery.build(), lqw)`（`:95`）执行分页查询，最后用 `TableDataInfo.build(result)`（`:99`）包成前端看到的 `rows/total`。
3. `ConfBillRuleMapper.java:26` 本身没有写 SQL，它继承的 `BaseMapperPlus` 提供 `selectVoPage`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/mapper/BaseMapperPlus.java:225`）；查哪张表由实体上的 `@TableName("conf_bill_rule")` 决定（`ConfBillRule.java:34`）。

> 如果是「新增」而不是「查询」，链路是：页面 `submitForm()`（`index.vue:362`）→ `addBillRule`（`api/conf/billRule.js:30-36`，参数放在 `data` 里）→ `ConfBillRuleController.java:110-112` 的 `add(...)` → `ConfBillRuleServiceImpl.java:130-138` 的 `insertByBo`，其中 `baseMapper.insert(add)`（`:133`）就是落库那一下。

返回 `TableDataInfo`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/page/TableDataInfo.java:36`），字段：

| 字段 | 行号 | 含义 |
|---|---|---|
| `total` | `:43` | 总记录数 |
| `rows` | `:49` | 当前页数据列表 |
| `code` | `:55` | 业务状态码 |
| `msg` | `:61` | 提示信息 |

JSON 大致是（下面的 `id`、`total` 数字只作结构示意，实际值以你的数据库为准）：

```json
{
  "code": 200,
  "msg": "查询成功",
  "rows": [ { "id": 1, "code": "APPROVE_ORDER_RULE", ... } ],
  "total": 3
}
```

请求头里的 `token: Bearer xxx` 由 Sa-Token 拦截器校验：`token-name: token`（`application.yml:103`）表示去名为 `token` 的请求头里找令牌，`token-prefix: "Bearer"`（`:119`）表示令牌值前面的 `Bearer ` 由框架剥掉。如果 token 不存在或已过期，这一步不会走到查询逻辑，而是返回一段 JSON，里面**业务字段 `code` 是 401**（由 `GlobalExceptionHandler.java:98-103` 包装）。注意：HTTP 状态码通常仍是 200，所以前端响应拦截器判的是 `res.data.code`，不是 HTTP 状态码。

---

### 第 11 步：响应回来，拦截器再「剥一层」

`request.js:71` 是响应拦截器：

```js
service.interceptors.response.use(res => {
  const code = res.data.code || 200;      // :73
  ...
  if (code === 401) { ... }               // :80-97
  else if (code === 500) { ... }          // :98-103
  else if (code !== 200) { ... }          // :104-108
  else {
    return res.data                       // :110
  }
})
```

几个要点：

1. `res` 是 axios 的**原始响应对象**，包含 `status`、`headers`、`data` 等。
2. 后端返回的 JSON 在 `res.data` 里。
3. 拦截器最后 `return res.data`，也就是**把后端 JSON 原样交给页面**。

所以页面 `.then((response) => ...)` 里的 `response` 已经是（数字同样只是示意）：

```js
{ code: 200, msg: '查询成功', rows: [...], total: 3 }
```

页面因此可以直接写 `response.rows`、`response.total`（`index.vue:294-295`），**不需要再写 `response.data.rows`**。

> 这是新手最常见的错误之一：以为拦截器剥掉的是 JSON 外面那层，于是多写一个 `.data`，结果拿到 `undefined`。本项目的约定是：**拦截器只剥 axios 自己的响应对象，不剥后端业务字段**。

如果 `code` 不是 200：

- `401` → 弹一个确认框「登录状态已过期」；用户点「重新登录」后才执行退出并跳回登录页（`request.js:80-97`）；
- `500` 或其他 → 弹错误提示，`Promise.reject`，页面不会进入 `.then` 的成功分支。

响应拦截器的整体设计见《Vue2 从 0 到精通》第 7 章 §7.4。

---

### 第 12 步：页面拿到数据并重绘

`index.vue:294-298`：

```js
this.billRuleList = response.rows           // :294
this.total = response.total                 // :295
this.activeName = response.rows[0].code     // :296 默认选中第一个页签
this.currentData = response.rows[0]         // :297 右侧表单显示第一条规则
this.loading = false                        // :298
```

- `billRuleList` 在 `data()` 里声明为数组（`index.vue:210`）；
- Vue 2 的响应式系统能感知对它的**整体重新赋值**（原理见《Vue2 从 0 到精通》第 10 章 §2）；
- 模板里的 `v-for` 依赖这个数组（`index.vue:9`），数据一变，页签自动重新渲染；
- `activeName` 决定哪个页签处于选中态（模板 `index.vue:4` 的 `v-model="activeName"`），`currentData` 是右侧表单绑定的对象；
- `loading = false` 把加载标记置回 `false`；本页模板没绑定它，所以这一行没有可见效果（见第 1 步的说明）。

> **空数据提醒**：`:296` 直接取了 `response.rows[0].code`。如果数据库一条记录都没有，`rows[0]` 是 `undefined`，这里会报 `Cannot read properties of undefined (reading 'code')`。这不是请求失败，而是页面没有处理空列表（见 §5.3 卡点表）。

至此，一次请求的完整生命周期结束。

---

## 二、URL 演进表：一张表看懂前缀怎么来、怎么没

这是整篇最值得收藏的一张表。**同一条请求，在每一站长什么样：**

| 阶段 | 此刻的地址 | 谁加工的 | 代码位置 |
|---|---|---|---|
| 1. api 函数里 | `/conf/billRule/list` + `params` 对象 | 开发者在 api 文件里写死 | `api/conf/billRule.js:15` |
| 2. 请求拦截器之后 | `/conf/billRule/list?pageNum=1&pageSize=10` | `tansParams` 拼查询串 | `request.js:36-41` |
| 3. axios 拼 baseURL 后 | `/dev-api/conf/billRule/list?pageNum=1&pageSize=10` | axios `combineURLs` | `request.js:21` |
| 4. 浏览器补全后 | `http://localhost/dev-api/conf/billRule/list?...` | 浏览器按同源补 origin | 浏览器行为 |
| 5. devServer 匹配 | 命中代理键 `/dev-api` | 前缀匹配 | `vue.config.js:37` |
| 6. pathRewrite 后 | `/conf/billRule/list?pageNum=1&pageSize=10` | 正则去掉 `^/dev-api` | `vue.config.js:40-42` |
| 7. 实际发往后端 | `http://localhost:8080/conf/billRule/list?...` | `target` 拼主机端口 | `vue.config.js:38` |
| 8. 后端收到的路径 | `/conf/billRule/list` | 与 Controller 映射对上 | `ConfBillRuleController.java:66,76` |

**看这张表的三个收获**：

1. `/dev-api` 只存在于第 3~5 站，是**前端开发期的临时前缀**；
2. 真正决定请求去哪的，是第 7 站的 `target`；
3. 后端只认识 `/conf/billRule/list`，它对 `/dev-api` 一无所知。

---

## 三、关键问题专答

### 3.1 `[process.env.VUE_APP_BASE_API]:` 为什么就能「匹配」？

**先纠正一个说法**：匹配不是这个语法完成的。

- `[表达式]:` 只负责**生成键名**。`process.env.VUE_APP_BASE_API` 的值是 `'/dev-api'`，所以这行等价于 `'/dev-api': {...}`。
- 真正做匹配的是 webpack-dev-server：它把**请求路径**和**代理对象的键**做**前缀比较**，`/dev-api/...` 以 `/dev-api` 开头，就命中。

所以完整因果是：

```
.env.development 提供值 '/dev-api'
  → 方括号语法把它变成对象键
  → devServer 用这个键做前缀匹配
  → 命中后按 target/pathRewrite 转发
```

### 3.2 axios 是怎么拼出 `/dev-api/conf/billRule/list` 的？

三步：

1. api 函数给出 `url: '/conf/billRule/list'`；
2. 请求拦截器把 `params` 拼成 `?pageNum=1&pageSize=10` 加在 `url` 后面；
3. axios 发现 `url` 不是完整地址，就把 `baseURL`（`'/dev-api'`）拼在前面，斜杠自动规整。

最终得到 `/dev-api/conf/billRule/list?pageNum=1&pageSize=10`。

### 3.3 为什么浏览器 Network 面板里只看得到 `localhost`，看不到 8080？

因为浏览器**只发了请求 ①**（到 devServer）。请求 ②（devServer → 8080）是 Node 进程内部发起的，**不经过浏览器**，所以浏览器开发者工具看不到。

想看请求 ②，去看 `npm run dev` 那个终端窗口，通常会打印类似：

```
[HPM] Proxy created: /dev-api -> http://localhost:8080
[HPM] GET /dev-api/conf/billRule/list -> http://localhost:8080
```

（`HPM` = http-proxy-middleware。不同版本的日志格式可能略有差别。）

### 3.4 后端为什么收到的是 `/conf/...` 而不是 `/dev-api/conf/...`？

因为 `pathRewrite: { '^/dev-api': '' }` 在转发前把开头的 `/dev-api` 删掉了。这就是为什么 Controller 上只写 `/conf/billRule`。

### 3.5 代理算「前端技术」吗？

严格说，它是**前端开发工具链**的一部分：

- 由 Vue CLI 启动、跑在 Node 进程里；
- 只在 `npm run dev` 时存在；
- `npm run build` 产物里没有它，生产环境由 Nginx 承担同样职责。

所以答辩时说「用 webpack-dev-server 的 proxy 解决开发期请求转发」，比说「前端写了个代理」准确。

### 3.6 本项目后端已经配了 CORS，为什么还要代理？

这点要诚实说清楚。后端确实在 `ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/ResourcesConfig.java:50-67` 配了全局 `CorsFilter`：

```java
config.setAllowCredentials(true);
config.addAllowedOriginPattern("*");
config.addAllowedHeader("*");
config.addAllowedMethod("*");
```

也就是说，**跨域不是本项目的唯一动机**，甚至不是最强动机。保留代理的理由是：

1. **统一前缀**：业务代码里只写 `/dev-api`，不关心后端地址；
2. **环境切换**：开发 `/dev-api`，生产 `/prod-api`（`.env.production:17`），业务代码零改动；
3. **生产同构**：生产环境本来就要用 Nginx 做 `/prod-api` 转发，开发期用同一套「前缀 → 转发」心智模型；
4. **隐藏后端地址**：前端源码和打包产物里不出现 `localhost:8080`。

> 这是本仓库一个值得答辩时主动讲的细节：**代理在，但后端也开了宽松 CORS**。不要把「代理就是为了解决跨域」当成唯一答案。

---

## 四、常见误解清单

| 误解 | 事实 |
|---|---|
| 「`process.env` 在浏览器里能读环境变量」 | 浏览器没有 `process`。值在**编译时**就被替换成字符串了 |
| 「`VUE_APP_BASE_API` 是运行时配置」 | 它是构建期常量。改 `.env` 后必须**重启** `npm run dev` 才生效 |
| 「`[key]:` 是数组语法」 | 它是对象的**计算属性名**，方括号里放表达式 |
| 「代理会修改后端代码/配置」 | 代理只改它转发出去的那份请求，后端无感知 |
| 「浏览器直接请求了 8080」 | 浏览器只请求 devServer；8080 是 Node 进程发的 |
| 「Network 里没有 8080，说明没连后端」 | 服务端那一跳浏览器看不见，要去终端看代理日志 |
| 「`response` 是 axios 原始响应，要写 `response.data.rows`」 | 本项目响应拦截器已 `return res.data`，页面直接 `response.rows` |
| 「`undefined` 参数会变成字符串 `'undefined'`」 | `tansParams` 会跳过 `null`/`undefined` |
| 「生产环境也靠 devServer 代理」 | 生产没有 devServer，靠 Nginx |
| 「后端没配 CORS 才需要代理」 | 本项目后端配了宽松 CORS；代理另有统一前缀/环境切换等理由 |

---

## 五、自己动手验证

前提：装好依赖（`zxcm-ui` 下 `npm install`）、启动后端 8080、`npm run dev` 启动前端，并确认已经登录（Cookie 里有 `Admin-Token`）。

### 5.1 完整跟一次请求（DevTools 实操）

这一节不是「看一眼」，而是按顺序把一条请求从面板里「抓出来、抄下来、对上号」。

1. **打开面板**：在编号规则页面按 `F12`（或 `Ctrl+Shift+I`）打开 DevTools，点顶部 **Network**（网络）面板。
2. **设置过滤**：勾上 **Preserve log**（保留日志），在过滤条选 **Fetch/XHR**。这样页面跳转或刷新后请求记录也不会丢。
3. **清空并复现**：点左上角的 🚫 清空记录，然后按 `F5` 刷新页面。`created()` 会自动执行一次 `getList()`，所以请求应该立刻出现。
4. **找到那条请求**：在 Name 列找 `list`（完整显示可能是 `list?pageNum=1&pageSize=10`）。找不到时，在过滤框输入 `billRule` 再试。注意：如果打开 DevTools 太晚，请求已经发完了，必须刷新一次页面才会重新出现。
5. **逐个面板核对**：单击这条请求，右侧会有一排分页签。按下面的「证据卡」逐项抄写，抄完再对照预期值。

**证据卡（把这张表抄到自己的笔记里）**

| 要抄的字段 | 在哪个面板 | 预期值 |
|---|---|---|
| Request URL | **Headers → General** | `http://localhost/dev-api/conf/billRule/list?pageNum=1&pageSize=10`（端口以你的 devServer 为准，默认 80） |
| Request Method | **Headers → General** | `GET` |
| Status Code | **Headers → General** | `200`（这是 HTTP 层的状态） |
| Remote Address | **Headers → General** | 以 `:80` 结尾（例如 `[::1]:80`、`127.0.0.1:80`），**不是 `:8080`** |
| `token` 请求头 | **Headers → Request Headers** | 以 `Bearer ` 开头，后面是令牌（Sa-Token 的 token-style 是 uuid，见 `application.yml:121`） |
| Query String Parameters | **Payload** 页签里的 **Query String Parameters** | 只有 `pageNum=1`、`pageSize=10` 两行；没有 `name`、`code` 等 undefined 字段 |
| 响应体 `code` / `msg` | **Response** | `200` / `查询成功`（`TableDataInfo.java:76-77`） |
| 响应体 `total` | **Response** | 等于你数据库 `conf_bill_rule` 当前未删除的记录数，**以你自己的环境为准，不要照抄别人的数字** |
| 响应体 `rows` | **Response** | 一个数组；长度不超过 `pageSize=10`，每条含 `id`、`name`、`code`、`rule` 等字段 |

6. **核对 token 的来源**：切到 **Application**（应用）面板 → **Cookies** → 当前源，找到 `Admin-Token`。它的值应该和请求头里 `Bearer ` 后面的部分一致。这能直观看到知识点 14 说的「Cookie 负责存、请求头负责传」。
7. **Console 面板补一刀**：在 **Console** 里输入 `process` 回车，预期报 `Uncaught ReferenceError: process is not defined`。但页面请求地址又是对的——因为 `process.env.VUE_APP_BASE_API` 在编译时就已经被替换成字符串了（知识点 7）。

> 如果第 3 步刷新后连 `list` 请求都看不到，先别怀疑代码：确认当前页面确实是编号规则页面、已经登录、Network 过滤没有选错类型，然后看 `npm run dev` 的终端有没有编译报错。

### 5.2 分项验证

以下是在 §5.1 基础上做的逐项确认或破坏性实验。

**验证 1：浏览器只联系了 80 端口**

F12 → Network → 刷新编号规则页面 → 找到 `list` 请求：

- Request URL 应该是 `http://localhost/dev-api/conf/billRule/list?...`
- 不是 `http://localhost:8080/...`

**验证 2：token 头确实加上了**

同一个请求 → Headers → Request Headers，应能看到：

```
token: Bearer xxxxx
```

**验证 3：参数确实被拼进 URL**

URL 里应有 `?pageNum=1&pageSize=10`，而不是一个 JSON 请求体。

**验证 4：代理日志**

看 `npm run dev` 的终端，代理命中时应打印 `[HPM]` 开头的转发日志。

**验证 5：后端断开时的表现**

停掉 Spring Boot，再刷新页面：

- Network 里请求显示失败；
- 页面弹出「后端接口连接异常」（`request.js:115-116` 把 `Network Error` 翻译成了中文）。

这说明请求确实转发到了 8080，只是目标没起来。不同版本的代理在目标不可达时也可能返回 500/504，此时页面提示会是「系统接口xxx异常」（`request.js:121-123`）；判断依据是 Network 里的目标地址和终端日志，不是提示文案。

**验证 6：改前缀，观察两处联动**

把 `.env.development:8` 改成 `/api-test`，重启 `npm run dev`：

- axios 的 `baseURL` 变成 `/api-test`；
- 代理键也变成 `/api-test`（因为两处读同一个变量）；
- 请求照常工作。

改完记得改回去。

### 5.3 常见卡点速查表

出问题时不要从头猜，先按「现象」找到对应的面板：

| 现象 | 先看哪个面板 / 位置 | 最可能的原因 | 怎么确认 |
|---|---|---|---|
| URL 对不上：出现 `:8080`，或没有 `/dev-api` | Network → Headers → Request URL | `.env` 改了没重启；api 里写了完整地址；`baseURL` 没生效 | 对照 [二、URL 演进表](#二url-演进表一张表看懂前缀怎么来怎么没)；在 Sources 里搜 `baseURL` |
| 请求头里没有 `token` | Network → Request Headers；Application → Cookies | 没登录；Cookie 里没有 `Admin-Token`；接口显式传了 `isToken: false` | Console 输入 `document.cookie`，或在 Application → Cookies 里找 `Admin-Token` |
| 代理 404：URL 是 `/dev-api/...`，但状态码 404 | Network → General / Response；`npm run dev` 终端 | 代理键和 `baseURL` 不一致；后端接口路径不是 `/conf/billRule/list`；`pathRewrite` 没去掉前缀 | 看终端 `[HPM]` 日志里转发到的目标 URL；再用浏览器直接访问 `http://localhost:8080/conf/billRule/list` 对照 |
| 请求失败 / 500 / 504，页面提示连接异常 | Network → General；后端控制台；`npm run dev` 终端 | 后端 8080 没启动；端口被占用；数据库连接失败 | 看后端启动日志第一条异常；确认 8080 端口在监听 |
| Console 出现 CORS 红字 | Console；Network → Headers → Response Headers | 请求真的发到了另一个 origin（例如把 URL 写成了 `http://localhost:8080/...`），没有经过代理 | 看 Request URL 的协议+主机+端口是否与页面一致 |
| `response.rows` 是 `undefined` | Network → Response；Sources 里打断点 | 多写了一层 `.data`；后端返回的不是 `TableDataInfo` | 在 `.then` 里 `console.log(response)`；对照 `TableDataInfo.java:43-61` |
| 页面报 `Cannot read properties of undefined (reading 'code')` | Network → Response；Console | 查询结果 `rows` 是空数组，但页面直接取了 `rows[0].code`（`index.vue:296`） | 看 Response 里 `rows` 是否为 `[]`；空库时先造一条编号规则 |
| URL 里出现字符串 `undefined` | Network → Payload | 参数不是经过 `tansParams` 拼的，而是手工字符串拼接 | 确认 GET 请求仍走 `request.js:36-41` 这段逻辑 |
| 页面空白 / 静态资源 404 | Console；Network 的 JS 请求 | 多半是路由或构建问题，不是请求层问题 | 先看 Console 第一条红字 |

---

## 六、动手改一改

### 练习 1：改开发端口

`vue.config.js:12`：

```js
const port = process.env.port || process.env.npm_config_port || 80
```

把默认值 `80` 改成 `81`，重启。页面变成 `http://localhost:81/`，请求也自动变成 `http://localhost:81/dev-api/...`——**因为请求是相对地址，永远跟着页面 origin 走**。

### 练习 2：加第二个代理前缀

假设还有一个服务在 9090，希望 `/report-api` 开头的请求转过去：

```js
proxy: {
  [process.env.VUE_APP_BASE_API]: {
    target: `http://localhost:8080`,
    changeOrigin: true,
    pathRewrite: { ['^' + process.env.VUE_APP_BASE_API]: '' }
  },
  '/report-api': {
    target: 'http://localhost:9090',
    changeOrigin: true,
    pathRewrite: { '^/report-api': '' }
  }
}
```

观察：代理对象可以有多个键，按前缀分别命中。这也是「一张表」的直观体现。

### 练习 3：模拟生产路径

把 `.env.development:8` 改成 `/prod-api`，同时在 `proxy` 里加一个 `/prod-api` 的键指向 8080。你会看到：**同样的业务代码，只靠环境变量就能换前缀**——这正是生产 Nginx 做的事。

---

## 七、答辩口径

**30 秒版：**

> 开发期前端跑在 80 端口，后端在 8080。axios 实例的 `baseURL` 从 `.env.development` 读到 `/dev-api`，所有接口自动带这个前缀；webpack-dev-server 按前缀匹配 `/dev-api`，`pathRewrite` 去掉前缀后转发到 8080。浏览器全程只跟同源的 devServer 通信。生产环境没有 devServer，由 Nginx 做同样的转发。

**2 分钟版（含设计取舍）：**

> 请求从前端页面 `created` 触发，api 函数只给路径和参数；`request.js` 里的 axios 实例加 `baseURL`，请求拦截器补 token、拼查询串。浏览器把相对地址补成 `localhost/dev-api/...` 发出去。devServer 的代理表以 `/dev-api` 为键做前缀匹配，改写路径后转发到 8080。响应拦截器统一判 `code`，最后把后端 JSON 原样返回，页面拿到 `rows/total` 渲染页签。
>
> **为什么这么设计**：业务代码里不出现后端地址，开发/生产只换环境变量；代理和 `baseURL` 读同一个变量，避免前缀不一致。补充一点：本项目后端其实也配了全局 CorsFilter，所以代理的主要价值不是「救跨域」，而是统一前缀、环境切换和生产一致性。

---

# 第二部分：前端知识补课（零基础）

> 下面每个知识点都是**独立小节**：先讲「是什么」，再给最小例子，最后指回本项目哪里用到了。可以按需跳读。

## 知识点 1：URL 的组成

一个 URL 拆开看：

```
http://localhost:8080/dev-api/conf/billRule/list?pageNum=1&pageSize=10#top
└─┬─┘ └───┬───┘ └┬┘ └──────────┬──────────┘ └──────────┬──────────┘ └┬┘
协议      主机     端口          路径                      查询串          片段
```

| 部分 | 本例 | 说明 |
|---|---|---|
| 协议 scheme | `http` | 怎么通信，还有 `https` |
| 主机 host | `localhost` | 服务器地址，也可以是域名/IP |
| 端口 port | `8080` | 省略时：http 默认 80，https 默认 443 |
| 路径 path | `/dev-api/conf/billRule/list` | 服务器上的资源位置 |
| 查询串 query | `pageNum=1&pageSize=10` | `?` 开始，`&` 分隔多个 `键=值` |
| 片段 hash | `#top` | 浏览器内部定位，不会发给服务器 |

**相对 URL vs 绝对 URL**：

- 绝对：`http://localhost:8080/xxx`，自带协议和主机；
- 相对：`/dev-api/xxx`（以 `/` 开头，接到当前 origin 根上）或 `xxx`（不以 `/` 开头，接到当前目录上）。

本项目的 api 路径全部以 `/` 开头，所以浏览器会用**页面的 origin** 补全它们。

---

## 知识点 2：同源策略与 CORS

**同源（same origin）** 的定义：

> 协议、主机、端口**三者完全相同**，才算同源。

举例（假设页面在 `http://localhost:80`）：

| 目标地址 | 同源？ | 原因 |
|---|---|---|
| `http://localhost/dev-api/x` | 是 | 三者一致（80 可省略） |
| `http://localhost:8080/x` | 否 | 端口不同 |
| `https://localhost/x` | 否 | 协议不同 |
| `http://127.0.0.1/x` | 否 | 主机字符串不同 |

**同源策略**是浏览器的一条安全规则：不同源的请求，默认不允许随意读取响应。

**CORS（跨域资源共享）** 是「例外机制」：后端在响应里返回 `Access-Control-Allow-Origin` 等头，浏览器看到后才放行。

关键认知：

1. **CORS 是浏览器执行的规则**，不是服务器拦你；
2. 服务器之间的请求（Node → Spring Boot）**不受 CORS 约束**；
3. 所以「代理」能绕过跨域——它让浏览器只跟同源服务器说话，再由服务器转发。

本项目后端也配了 CORS（`ruoyi-base/ruoyi-framework/src/main/java/com/ruoyi/framework/config/ResourcesConfig.java:50-67`），但开发期仍使用代理，理由见 §3.6。

---

## 知识点 3：Node.js、npm、package.json

**Node.js**：让 JavaScript 能脱离浏览器运行的运行时。前端开发用的构建工具（webpack、devServer）都跑在 Node 里。

**npm**：Node 的包管理器，用来安装依赖、运行脚本。

**package.json**：项目说明书，关键字段：

```json
{
  "scripts": {
    "dev": "set NODE_OPTIONS=--openssl-legacy-provider && vue-cli-service serve",
    "build:prod": "vue-cli-service build"
  },
  "dependencies": { "axios": "0.24.0", "vue": "2.6.12" },
  "devDependencies": { "@vue/cli-service": "4.4.6" }
}
```

本项目对应行号：`package.json:8`（dev 脚本）、`:41`（axios）、`:64`（vue）、`:78`（cli-service）。

**`npm run dev` 做了什么**：

1. npm 找到 `scripts.dev` 这条命令；
2. 执行 `vue-cli-service serve`；
3. Vue CLI 读取 `.env.development`，启动 webpack 编译；
4. 启动 webpack-dev-server，监听 80 端口；
5. 打开浏览器。

所以「devServer 代理」能生效的前提，是你在**跑开发服务器**，而不是打开一个打包后的 `dist/index.html`。

---

## 知识点 4：Vue CLI / webpack / devServer / 代理中间件

四者的关系：

```
Vue CLI（脚手架/命令行工具，4.4.6）
   │ 负责：读配置、组织构建流程
   ▼
webpack（打包器）
   │ 负责：把 .vue/.js/.css 编译打包成浏览器能跑的 JS
   ▼
webpack-dev-server（开发服务器，Node 进程）
   │ 负责：提供页面、热更新、以及 proxy 代理
   ▼
http-proxy-middleware（代理中间件）
     负责：真正实现"前缀匹配 + 改写 + 转发"
```

你只需要记住：

- **webpack** 是「打包」的；
- **devServer** 是「开发期开服务器」的，`vue.config.js` 里的 `devServer` 就是配它；
- **proxy** 是 devServer 的一个能力，底层由代理中间件实现；
- 这三样**只在开发期存在**，生产环境是静态文件 + Nginx。

`vue.config.js` 就是 Vue CLI 的配置文件，`devServer` 段的注释也写了：`// webpack-dev-server 相关配置`（`vue.config.js:30`）。

---

## 知识点 5：ES Module 与 `@` 别名

**导出**（`src/api/conf/billRule.js:13-19`）：

```js
export function listBillRule(query) { ... }   // 具名导出
```

**导入**（`index.vue:171-177`）：

```js
import { listBillRule } from '@/api/conf/billRule'   // 具名导入
```

规则：

- 名字必须对上；
- 一个文件可以导出多个，导入时用 `{}` 选需要的；
- `export default xxx` 是默认导出，导入时不用 `{}` 且名字随便起（如 `import request from '@/utils/request'`）。

**`@` 别名**在 `vue.config.js:57-59`：

```js
resolve: {
  alias: {
    '@': resolve('src')
  }
}
```

把 `@` 映射到 `src/` 的绝对路径，好处是无论文件嵌套多深，`@/utils/request` 都指向同一个位置，不用写 `../../../../utils/request`。

---

## 知识点 6：axios 与「请求配置对象」

**axios** 是浏览器里发 HTTP 请求的库（本项目 0.24.0）。典型用法：

```js
import axios from 'axios'
axios.get('/x')                        // 便捷方法
axios({ url: '/x', method: 'get' })    // 传配置对象
```

**请求配置对象**就是描述「这次请求长什么样」的普通对象：

| 字段 | 作用 | 本项目示例 |
|---|---|---|
| `url` | 请求路径 | `/conf/billRule/list` |
| `method` | HTTP 方法 | `get` / `post` / `put` / `delete` |
| `params` | 查询参数（拼到 `?` 后面） | `queryParams` |
| `data` | 请求体（POST/PUT 用） | 新增/修改的表单对象 |
| `headers` | 请求头 | `token`、`Content-Type` |
| `timeout` | 超时时间 | 实例上统一 10 秒 |

**`axios.create()`** 创建实例，可以预设默认值。本项目在 `src/utils/request.js:19-24`：

```js
const service = axios.create({
  baseURL: '/dev-api',
  timeout: 10000
})
```

之后 `service({ url: '/x' })` 就自动带上这些默认值。

**`baseURL` 的拼接行为**：相对 `url` 前自动加 `baseURL`，绝对 `url`（`http://...`）则忽略 `baseURL`。

---

## 知识点 7：`.env` 环境变量与编译期替换

**`.env` 文件**：一行一个 `键=值` 的纯文本配置。

本项目三个环境文件：

| 文件 | 何时加载 | 前缀 | 行号 |
|---|---|---|---|
| `.env.development` | `npm run dev` | `/dev-api` | `:8` |
| `.env.production` | `npm run build` | `/prod-api` | `:17` |
| `.env.staging` | `npm run build:stage` | `/stage-api` | `:19` |

**Vue CLI 的两条规则**（系统讲解见《Vue2 从 0 到精通》第 7 章 §5.3）：

1. 只有 `VUE_APP_` 开头的变量会注入前端代码；
2. **注入是编译期替换**，不是运行时读取。

你能在源码里写 `process.env.VUE_APP_BASE_API`，只是因为构建工具会把它替换成字符串。浏览器里不存在 `process`。

**验证方法**：在浏览器控制台输入 `process`，会报 `process is not defined`；但页面里的请求地址却正确——因为它早就被替换成字面量了。

**改完 `.env` 必须重启**：因为替换发生在启动编译时，热更新不一定重新读取环境文件。

---

## 知识点 8：JS 对象与计算属性名 `[expr]`

**对象字面量**：

```js
const user = { name: '张三', age: 20 }
```

**键名规则**：

- 普通写法：`{ key: value }`，键名就是字面量 `key`；
- 计算属性名：`{ [表达式]: value }`，先算表达式，再拿结果当键。

```js
const k = 'name'
const a = { [k]: '张三' }        // { name: '张三' }

const prefix = '/dev-api'
const b = { [prefix]: 1 }        // { '/dev-api': 1 }
```

**为什么项目要这么写**：

```js
[process.env.VUE_APP_BASE_API]: { ... }        // 代理键
['^' + process.env.VUE_APP_BASE_API]: ''       // 正则键
```

如果写死 `'/dev-api'`，那么 axios 的 `baseURL` 改了、代理键忘改，就会出现「请求带 `/api`、代理只认 `/dev-api`」的错位。用同一个变量生成两处，**从机制上避免不一致**。

---

## 知识点 9：Promise 与 `.then()`

JS 的请求是**异步**的：发出请求后，程序不会停在那里等，而是继续往下执行；响应回来后再执行回调。

**Promise** 就是「未来会有一个结果」的占位对象。

```js
listBillRule(params)
  .then(response => {
    // 成功：response 是响应数据
    this.billRuleList = response.rows
  })
  .catch(error => {
    // 失败
    console.error(error)
  })
```

本项目 `index.vue:293` 只用了 `.then`，错误提示由 `request.js:71-110` 的响应拦截器统一处理。

**async/await** 是 Promise 的另一种写法（本项目 api 层未使用，但你会见到）：

```js
async function load() {
  const response = await listBillRule(params)
  this.billRuleList = response.rows
}
```

`await` 只是让代码看起来像同步，本质还是 Promise。

---

## 知识点 10：拦截器

**拦截器（interceptor）** = 在请求/响应流程中「必经的关卡」。

本项目两处：

**请求拦截器**（`request.js:27`），在请求发出前：

- 加 token；
- 把 GET 参数拼成查询串；
- 做前端防重复提交；
- 最后必须 `return config`，把配置交回给 axios。

**响应拦截器**（`request.js:71`），在响应回来后：

- 读 `res.data.code`；
- 401 弹重登；
- 其他错误统一提示；
- 成功则 `return res.data`。

**为什么用拦截器**：全站几十个接口，如果每个 api 函数都自己写「加 token、判错误码」，重复且容易漏。集中到关卡里，业务代码只关心「路径 + 参数」。

---

## 知识点 11：相对 URL 如何被补全

浏览器解析请求地址时：

| 写法 | 补全结果（页面在 `http://localhost:80/app/`） |
|---|---|
| `http://x.com/a` | 原样使用（绝对地址） |
| `/dev-api/a` | `http://localhost:80/dev-api/a`（接到 origin 根） |
| `dev-api/a` | `http://localhost:80/app/dev-api/a`（接到当前目录） |

本项目所有 api 的 `url` 都以 `/` 开头，因此永远接到**页面 origin 的根**，不受页面路径影响。

这也解释了一个现象：**页面端口一改，请求端口自动跟着改**——因为请求根本没有写端口，它跟着页面走。

---

## 知识点 12：代理、前缀匹配、pathRewrite

**代理（proxy）**：一个中间人。客户端把请求发给它，它代替客户端把请求转给真正的目标，再把结果带回来。

**本项目代理的三要素**（定义在 `vue.config.js:35-44`，系统讲解见《Vue2 从 0 到精通》第 7 章 §4.3）：

| 配置 | 作用 | 例子 |
|---|---|---|
| 键 `'/dev-api'` | **前缀匹配**：请求路径以它开头就命中 | `/dev-api/xxx` 命中 |
| `target` | 转发目标（协议+主机+端口） | `http://localhost:8080` |
| `pathRewrite` | 转发前改写路径 | `'^/dev-api': ''` 去掉前缀 |
| `changeOrigin` | 改写 Host 头 | `Host: localhost:8080` |

**正则里的 `^`**：表示「字符串开头」。`'^/dev-api'` 只匹配开头的 `/dev-api`，如果路径中间出现 `/dev-api` 不会被误删。

**前缀匹配不是正则匹配**：代理键是普通字符串，devServer 用「以它开头」判断；`pathRewrite` 的键才是正则。

**代理不改变后端**：它只影响转发出去的那份请求。

---

## 知识点 13：Vue 基础

一个 `.vue` 文件通常三段：

```vue
<template>
  <!-- 页面结构（HTML） -->
</template>

<script>
export default {
  data() { return { list: [] } },     // 数据
  created() { this.getList() },       // 生命周期
  methods: { getList() { ... } }      // 方法
}
</script>

<style>
/* 样式 */
</style>
```

本流程用到的几个点：

| 概念 | 含义 | 本项目位置 |
|---|---|---|
| `data()` | 声明组件的响应式数据 | `index.vue:192` 起；`queryParams` 在 `:219-230` |
| `created()` | 组件创建后自动执行一次 | `index.vue:265` |
| `methods` | 组件的方法集合 | `index.vue:268` 起 |
| `this` | 指向当前组件实例，用来访问 data 和 methods | `this.getList()`（`index.vue:293`） |
| `v-for` | 循环渲染列表 | `index.vue:9` 的顶部页签 |
| 响应式 | 数据变了，界面自动更新 | `this.billRuleList = ...` 后页签刷新 |

**响应式的直观理解**：Vue 2 会追踪 `data` 里的属性，一旦你把新值赋给它们，Vue 就知道「界面里用到这个值的地方需要重画」。系统讲解见《Vue2 从 0 到精通》第 3 章 §2 和第 10 章 §2。

---

## 知识点 14：Cookie 与 token

**Cookie**：浏览器为某个网站保存的一小段键值数据，页面刷新后还在。

本项目 `src/utils/auth.js:1-15`：

```js
import Cookies from 'js-cookie'          // :1 浏览器 Cookie 的工具库
const TokenKey = 'Admin-Token'           // :3 Cookie 的键名

export function getToken() {             // :5
  return Cookies.get(TokenKey)           // :6 读
}
export function setToken(token) {        // :9
  return Cookies.set(TokenKey, token)    // :10 写
}
export function removeToken() {          // :13
  return Cookies.remove(TokenKey)        // :14 删
}
```

登录成功后，`store/modules/user.js:57` 调 `setToken(res.data.token)` 把后端返回的 token 存进 Cookie；之后每个请求由拦截器读出（`request.js:32-34`），放进请求头：

```js
config.headers['token'] = 'Bearer ' + getToken()
```

后端 `application.yml:103` 的 `token-name: token` 表示 Sa-Token 去名为 `token` 的请求头读令牌；`:115` 的 `is-read-head: true` 打开「从请求头读」，`:117` 的 `is-read-cookie: false` 则明确不从 Cookie 读。

> 注意区分：Cookie 是**存储位置**，请求头 `token` 是**传输方式**。本项目把 token 存 Cookie、再用请求头传，两者不是一回事。

---

## 知识点 15：JSON、HTTP 状态码与业务 code

**JSON**：一种文本格式，用来在前后端之间传结构化数据。

```json
{ "code": 200, "msg": "查询成功", "rows": [], "total": 0 }
```

**HTTP 状态码**：由 HTTP 协议规定，例如 200 成功、401 未认证、403 无权限、500 服务器错误。

**业务 code**：后端在 JSON 里再放一个 `code` 字段，表示业务结果。

本项目同时存在两层：

- HTTP 状态码：由服务器/框架决定；
- 业务 `code`：`request.js:73` 读 `res.data.code` 判断。

响应拦截器主要看业务 `code`：

| code | 处理 |
|---|---|
| 200 | 返回数据 |
| 401 | 弹重新登录 |
| 500 | 弹错误 |
| 其他 | 通知错误 |

这就是为什么「接口返回 200 但页面提示失败」也可能发生——HTTP 层成功，业务层失败。

---

## 术语总表

| 术语 | 一句话解释 | 本项目出现位置 |
|---|---|---|
| 开发服务器 devServer | `npm run dev` 起的 Node 进程，提供页面并转发请求 | `vue.config.js:31` |
| 代理 proxy | 中间人：收下请求，改写后转发给目标 | `vue.config.js:35-44` |
| 前缀匹配 | 请求路径以代理键开头即命中 | `vue.config.js:37` |
| pathRewrite | 转发前用正则改写路径 | `vue.config.js:40-42` |
| changeOrigin | 转发时改写 Host 头 | `vue.config.js:39` |
| target | 转发目标地址 | `vue.config.js:38` |
| baseURL | axios 给所有请求加的公共前缀 | `request.js:21` |
| 请求拦截器 | 请求发出前的统一处理 | `request.js:27` |
| 响应拦截器 | 响应返回后的统一处理 | `request.js:71` |
| 环境变量 | `.env` 文件里的配置 | `.env.development:8` |
| 计算属性名 | `[表达式]: 值`，动态生成对象键 | `vue.config.js:37` |
| 同源 | 协议+主机+端口完全一致 | 浏览器规则 |
| CORS | 跨域时浏览器要求的放行机制 | 后端 `ResourcesConfig.java:51` |
| token | 登录凭证，放在请求头里 | `request.js:33` |
| 生命周期钩子 | Vue 在特定时机自动调用的函数 | `index.vue:265` |

---

## 小结与下一步

**这篇文档要你记住的三件事**：

1. **浏览器只知道 devServer**：请求先到 `localhost:80`，`8080` 是 Node 进程在背后转发；
2. **`/dev-api` 是编译期常量 + 代理键**：前者来自 `.env.development`，后者用同一个变量算出来，所以两处永远一致；
3. **代理只在开发期存在**：生产由 Nginx 承担同样职责，业务代码里不写死后端地址。

**如果还有哪一步没看懂**：

- 看不懂「为什么要分两段请求」→ 回看 [知识点 2](#知识点-2同源策略与-cors) 和 [§3.3](#33-为什么浏览器-network-面板里只看得到-localhost看不到-8080)；
- 看不懂 `[xxx]:` → 回看 [知识点 8](#知识点-8js-对象与计算属性名-expr)；
- 看不懂 `process.env` → 回看 [知识点 7](#知识点-7env-环境变量与编译期替换)；
- 看不懂 `.then` → 回看 [知识点 9](#知识点-9promise-与-then)；
- 想看请求层的整体地图 → 回 [第 1 章 §四](第1章-请求层与登录鉴权.md)；
- 想看前端怎么启动、路由怎么装 → 回 [第 0 章 §五](第0章-前端全景与运行机制.md)。

**下一步建议**：带着这篇的 URL 演进表，去读一个**带 POST 请求**的页面（例如新增编号规则，`api/conf/billRule.js:30-36`），观察 `params` 和 `data` 的区别——这是从「读懂」走向「能改」的第一道坎。

---

## 本章答辩卡：3 组追问级问答

> 用法：先自己答一遍，再对照「10 秒答案」和证据行号。三组都围绕同一条链：**页面触发 → axios → 代理 → Controller → Service → Mapper → 数据库 → 响应回页面**。

### 第 1 组：从页面触发到后端入库，中间经过了哪些层？

**问题**：「你说你负责前端请求这一块，那从页面点一下到数据落库，中间到底走了哪些层？」

**10 秒答案**：
> 页面在 `created` 里调 `getList`，api 函数只把「路径 + 方法 + 参数」整理成配置对象；axios 实例补上 `/dev-api` 前缀，请求拦截器加 token、把 GET 参数拼进 URL；浏览器按同源补全地址，发到 devServer 的 80 端口；devServer 用 `/dev-api` 做前缀匹配，`pathRewrite` 去掉前缀后转发到 8080；Controller 接参后调 Service，Service 组装查询条件、Mapper 执行 SQL 读写 `conf_bill_rule` 表；响应回来先过响应拦截器，`return res.data`，页面再拿 `rows/total` 更新页签。新增则走 POST：参数放在 `data` 请求体里，Controller 的 `add` → Service 的 `insertByBo` → `baseMapper.insert` 落库。

**证据**：
- 触发与消费：`index.vue:265-267`、`index.vue:291-299`；新增点击入口 `index.vue:362-389`
- 查询 API / 新增 API：`api/conf/billRule.js:13-19`、`api/conf/billRule.js:30-36`
- axios 实例与请求拦截器：`request.js:19-24`、`request.js:27-41`；POST 的 JSON 请求体格式来自 `request.js:15`
- 代理：`vue.config.js:35-44`
- Controller：`ConfBillRuleController.java:66`、`ConfBillRuleController.java:76-78`；新增在 `:110-112`
- Service：`ConfBillRuleServiceImpl.java:93-100`；新增在 `:130-138`（落库是 `:133`）
- 表映射：`ConfBillRule.java:34`
- 响应拦截器：`request.js:71-110`

**追问预判**：「为什么 api 函数不直接写 `axios.get`？」——为了统一 `baseURL`、token 和错误处理，全站只改一处。

### 第 2 组：为什么 Network 里看不到 8080？404 时怎么判断断在哪一段？

**问题**：「浏览器 Network 里根本没有 8080，你怎么证明请求到了后端？万一 404，你怎么定位？」

**10 秒答案**：
> 因为代理是两段独立请求：浏览器只请求同源的 devServer（80 端口），devServer 这个 Node 进程再请求 8080；第二段不经过浏览器，所以 Network 看不到。404 时按 URL 演进表逐站对：api 里是不是 `/conf/billRule/list`、拦截器拼参后是不是带 `?pageNum=1&pageSize=10`、编译后的 `baseURL` 是不是 `/dev-api`、代理键是不是同一个值、`pathRewrite` 之后与后端 Controller 的类级 + 方法级路径是否一致。哪一站对不上就改哪一站；必要时用 `curl` 直接打 8080，把「前端代理问题」和「后端路由问题」分开。

**证据**：
- 代理键与路径改写：`vue.config.js:37-43`
- `baseURL`：`request.js:21`
- GET 参数拼接：`request.js:36-41`
- api 里的接口路径：`api/conf/billRule.js:15`
- 后端端口与上下文路径：`application.yml:30`、`application.yml:33`
- Controller 映射：`ConfBillRuleController.java:66`、`ConfBillRuleController.java:76`
- 两段请求的完整对照：本篇 [二、URL 演进表](#二url-演进表一张表看懂前缀怎么来怎么没)

**追问预判**：「生产环境也靠 devServer 吗？」——不靠。生产没有 devServer，由 Nginx 用 `/prod-api` 反向代理，规则相同（`.env.production:17`）。

### 第 3 组：token 是怎么从登录带到这个请求的？没带或过期会怎样？

**问题**：「登录之后，token 是怎么跟着每个请求走的？如果 token 失效，前端怎么处理？」

**10 秒答案**：
> 登录成功后 `setToken` 把 token 写进 Cookie `Admin-Token`；每个请求发出前，请求拦截器读 Cookie，把 `'Bearer ' + token` 写进请求头 `token`。后端 Sa-Token 按 `token-name: token` 从请求头取令牌，并用 `token-prefix: Bearer` 去掉前缀。令牌无效或过期时，全局异常处理器返回业务 `code=401` 的 JSON；响应拦截器弹重登确认框并 `Promise.reject`，页面不会进入 `.then` 的成功分支。

**证据**：
- 登录写 Cookie：`store/modules/user.js:57`；`auth.js:9-11`
- 读 Cookie / 加请求头：`auth.js:5-7`、`request.js:32-34`
- 后端读取配置：`application.yml:103`、`:115`、`:119`
- 401 的包装与前端处理：`GlobalExceptionHandler.java:98-103`、`request.js:80-97`
- HTTP 状态码与业务 code 的区别：本篇 [知识点 15](#知识点-15jsonhttp-状态码与业务-code)

**追问预判**：「为什么不直接让后端从 Cookie 读？」——本项目明确把「Cookie 只做存储、请求头做传输」：`is-read-head: true`、`is-read-cookie: false`（`application.yml:115-117`），所以后端只认请求头，前端拦截器保证每个请求都带上。
