# 第 5 章 · Vue Router 路由与导航

> **这一章解决什么问题**：点左侧菜单，为什么页面就换了？地址栏里的路径怎么对应到 `.vue` 文件？为什么没登录会被踢到登录页？后端返回的菜单又是怎么变成前端路由的？这一章把整条链路走通。
>
> **学完标准**：能看懂 `src/router/index.js` 和 `src/permission.js`；能说清「输入一个 URL → 显示某个页面」的全过程；能自己加一条路由；能解释 `addRoutes` 和 `beforeEach` 在权限控制里的作用。

---

## 一、路由解决什么问题

没有路由时，SPA 只有一页。用户点「岗位管理」「合同台账」，你得自己判断「现在该显示哪个组件」，还要自己处理浏览器前进/后退。

Vue Router 把这件事接管了：

```
地址栏 URL  →  路由表里查匹配项  →  渲染对应的组件
```

- URL 变了，页面不刷新，只换组件；
- 浏览器前进/后退可用；
- 可以传参（`/system/dict-data/index/5`）、可以做跳转拦截（守卫）。

本项目的版本是 **Vue Router 3.4.9**（`zxcm-ui/package.json:69`），配套 Vue 2。Vue Router 4 是给 Vue 3 的，API 完全不同（`createRouter()`），不要混。

---

## 二、路由是怎么装进来的

### 2.1 入口：两行代码

`src/router/index.js:1-4`：

```js
import Vue from 'vue'
import Router from 'vue-router'

Vue.use(Router)
```

`Vue.use(Router)` 把路由插件装到 Vue 上，之后组件里才有 `this.$router`、`this.$route`。

### 2.2 创建路由实例

`src/router/index.js:209-213`：

```js
export default new Router({
  mode: 'history', // 去掉url中的#
  scrollBehavior: () => ({ y: 0 }),
  routes: constantRoutes
})
```

| 配置 | 含义 |
|---|---|
| `mode: 'history'` | 用 HTML5 History 模式，URL 里没有 `#` |
| `scrollBehavior` | 每次切换路由，页面滚回顶部 |
| `routes` | 初始路由表（公共路由） |

### 2.3 挂到根实例

`src/main.js` 里：

```js
import router from './router'
```

`src/main.js:209-214`：

```js
new Vue({
  el: "#app",
  router,
  store,
  render: (h) => h(App),
})
```

把 `router` 交给根实例后，所有组件都能通过 `this.$router` / `this.$route` 使用路由。

> `src/main.js:59` 还有一句 `import "./permission";`，导入这个文件就会执行里面的路由守卫注册（§九）。它不是组件，只是「一进来就跑」的副作用。

---

## 三、路由表：URL 和组件的对应关系

### 3.1 一条路由长什么样

最小版本：

```js
{
  path: '/login',
  component: () => import('@/views/login'),
  hidden: true
}
```

| 字段 | 作用 |
|---|---|
| `path` | URL 路径 |
| `component` | 匹配后渲染哪个组件 |
| `name` | 路由名，用于命名跳转、`keep-alive` 匹配、调试 |
| `hidden` | 不在侧边栏显示（登录页、404 等） |
| `redirect` | 访问这个路径时重定向到别处 |
| `children` | 子路由 |
| `meta` | 自定义信息：标题、图标、是否缓存等 |
| `permissions` / `roles` | 项目自定义字段，用于动态路由权限过滤 |

`src/router/index.js:9-29` 的注释把每个字段都解释了一遍，其中有一句很关键：

```js
* name:'router-name'               // 设定路由的名字，一定要填写不然使用<keep-alive>时会出现各种问题
```

第 3 章讲过：`keep-alive` 的 `include` 按组件 `name` 匹配，而标签页里存的是路由 `name`，两边要对上。

### 3.2 公共路由 `constantRoutes`

`src/router/index.js:32-132`。所有用户都能访问的页面放在这里，不需要等登录后从后端拿菜单：

| 路径 | 组件 | 说明 |
|---|---|---|
| `/login` | `views/login` | 登录页 |
| `/register` | `views/register` | 注册页 |
| `/404`、`/401` | 错误页 | 找不到页面 / 无权限 |
| `/redirect/:path(.*)` | `views/redirect` | 刷新中间页（§九） |
| `/contractFileEdit` 等 | WPS 在线编辑/预览 | 通过链接直接打开，不挂菜单 |
| `''`（根路径） | `Layout` | 后台主框架 |
| `/user/profile` | 个人中心 | 挂在 `Layout` 下 |

### 3.3 嵌套路由：为什么后台页面外面有菜单栏

看首页这段 `src/router/index.js:99-111`：

```js
{
  path: '',
  component: Layout,
  redirect: "/index",
  children: [
    {
      path: 'index',
      component: () => import('@/views/index'),
      name: 'Index',
      meta: { title: '首页', icon: 'dashboard', affix: true }
    },
  ]
}
```

结构是：

```
URL: /index
  → 外层 path ''  匹配 Layout 组件
      Layout 里有侧边栏、顶栏、标签页（layout/index.vue:2-15）
      Layout 的内容区 <app-main /> 里有一个 <router-view>
  → 内层 path 'index' 匹配 views/index
      渲染到 Layout 的 <router-view> 里
```

所以后台页面的「外框」（菜单 + 顶栏）来自 `Layout`，具体页面内容来自子路由。这就是**嵌套路由**。

`Layout` 的内容区 `src/layout/index.vue:10`：

```html
<app-main />
```

`src/layout/components/AppMain.vue:3-6`：

```html
<keep-alive :include="cachedViews">
  <router-view :key="key" />
</keep-alive>
```

**`<router-view>` 就是「路由出口」**：当前匹配到哪个组件，就在这个位置渲染哪个组件。

- `src/App.vue:3` 有一个顶层 `<router-view />`，渲染 `Layout` 或登录页；
- `AppMain.vue:5` 有一个二级 `<router-view />`，渲染 `Layout` 里的具体业务页面。

### 3.4 `:key="key"` 有什么用

`src/layout/components/AppMain.vue:18-20`：

```js
key() {
  return this.$route.path
}
```

`:key` 变化时，Vue 会**销毁旧组件、创建新组件**。

例如从 `/system/user-auth/role/1` 跳到 `/system/user-auth/role/2`，路径不同，`key` 不同，组件重建，页面重新初始化。如果不加 `key`，Vue 可能复用同一个组件实例，导致「换了 id 但页面数据没刷新」。

> 代价：每次路径变化都重建组件，状态不保留。项目用 `keep-alive` 缓存需要保留的页面（`cachedViews`），两者配合。

### 3.5 动态路由 `dynamicRoutes`

`src/router/index.js:135-207`：

```js
export const dynamicRoutes = [
  {
    path: '/system/dict-data',
    component: Layout,
    hidden: true,
    permissions: ['system:dict:list'],
    children: [
      {
        path: 'index/:dictId(\\d+)',
        component: () => import('@/views/system/dict/data'),
        name: 'Data',
        meta: { title: '字典数据', activeMenu: '/system/dict' }
      }
    ]
  },
  // ...
]
```

这类页面**不挂在菜单上**，但需要能从别的页面跳进去（比如从字典列表点「数据」跳到字典数据页）。它们带 `permissions`，登录后按用户权限过滤，再动态加到路由表里（§10）。

### 3.6 三种路由来源

| 来源 | 定义位置 | 什么时候生效 | 例子 |
|---|---|---|---|
| 公共路由 | `constantRoutes` | 应用启动就有 | 登录页、首页、404 |
| 动态路由（前端写死） | `dynamicRoutes` | 登录后按权限过滤加入 | 字典数据、分配角色 |
| 后台菜单路由 | 后端接口返回 | 登录后请求菜单，转换后加入 | 岗位管理、合同台账 |

---

## 四、导航：怎么跳转

### 4.1 声明式：`<router-link>`

```html
<router-link to="/system/position">岗位管理</router-link>
```

它渲染成一个 `<a>`，点击后由路由接管，不刷新页面。

项目里封装了一层 `src/layout/components/Sidebar/Link.vue:21-26`：

```js
type() {
  if (this.isExternal) {
    return 'a'
  }
  return 'router-link'
}
```

外链用普通 `<a>`，内部页面用 `<router-link>`。侧边栏菜单就是靠它跳转的。

### 4.2 编程式：`$router.push` / `replace` / `go`

```js
this.$router.push('/system/position')
this.$router.push({ path: '/system/position', query: { id: 1 } })
this.$router.replace('/login')   // 替换当前历史记录，不能后退回来
this.$router.go(-1)              // 后退一步
this.$router.back()              // 等同 go(-1)
```

项目例子：

登录成功后 `src/views/login.vue:276`：

```js
this.$router.push({ path: this.redirect || '/' }).catch(() => { })
```

`redirect` 来自登录页对 `$route` 的监听（第 3 章 §5.3）：从哪个页面被踢出来，登录后就回哪个页面。

首页快捷入口 `src/views/index.vue:241`：

```js
this.$router.push({ path: "/contract/contractAudit" }).catch(() => {});
```

**为什么后面都带 `.catch(() => {})`？**

Vue Router 3.1 起，`push` 返回 Promise。重复跳转到当前路由时，Promise 会以 `NavigationDuplicated` 拒绝，控制台报红。`.catch(() => {})` 把这个预期内的错误吞掉。这是项目里的统一处理方式。

### 4.3 `$router` 和 `$route` 的区别

| | `$router` | `$route` |
|---|---|---|
| 是什么 | 路由**实例**（全局唯一） | 当前**路由信息对象** |
| 用途 | 跳转：`push`、`replace`、`go` | 读取：`path`、`query`、`params`、`meta` |
| 例子 | `this.$router.push('/index')` | `this.$route.query.id` |

一句话：**`$router` 用来「操作」，`$route` 用来「读取」。**

---

## 五、路由传参：`query` 与 `params`

### 5.1 `query`：挂在 URL 问号后面

跳转：

`src/views/tool/gen/index.vue:324`：

```js
this.$router.push({
  path: '/tool/gen-edit/index/' + tableId,
  query: { pageNum: this.queryParams.pageNum }
})
```

URL 变成：

```
/tool/gen-edit/index/5?pageNum=2
```

读取：

`src/views/tool/gen/index.vue:238-241`（`activated` 里）：

```js
const time = this.$route.query.t
// ...
this.queryParams.pageNum = Number(this.$route.query.pageNum)
```

特点：

- 参数在 URL 里，**刷新不丢**；
- 刷新后仍是字符串，数字要自己转；
- 适合「可选参数、筛选条件、来源标记」。

### 5.2 `params`：写在路径里的动态段

路由定义 `src/router/index.js:185-190`：

```js
{
  path: 'index/:dictId(\\d+)',
  component: () => import('@/views/system/dict/data'),
  name: 'Data',
  // ...
}
```

`:\d+` 是正则约束，表示只匹配数字。

跳转（`src/views/system/dict/index.vue` 类似写法，项目里还有）：

```js
this.$router.push('/system/dict-data/index/5')
```

读取 `src/views/system/dict/data.vue:276`：

```js
const dictId = this.$route.params && this.$route.params.dictId
```

特点：

- 参数是**路径的一部分**，刷新不丢（因为 URL 里本来就有）；
- 必须先在路由表里定义 `:参数名`；
- 适合「资源 id 这类必填、有层级意义的参数」。

> 对比「命名路由 + `params` 但不写进 path」的用法：那种情况下参数只存在内存里，刷新就没了。本项目主要用「动态段 + `query`」两种，刷新都安全。

### 5.3 怎么选

| | `query` | `params`（路径动态段） |
|---|---|---|
| URL 形态 | `?id=5` | `/index/5` |
| 路由表要不要先定义 | 不用 | 要，写 `:id` |
| 刷新是否保留 | 保留 | 保留 |
| 语义 | 可选、筛选、来源 | 必填、资源标识 |
| 读取 | `$route.query.id` | `$route.params.id` |

---

## 六、两种路由模式：`hash` 与 `history`

### 6.1 区别

| | hash 模式 | history 模式 |
|---|---|---|
| URL | `http://xxx/#/system/position` | `http://xxx/system/position` |
| 原理 | 监听 `#` 后面的变化，不请求服务器 | 用 `history.pushState` 改地址 |
| 刷新 | 浏览器只请求 `/`，安全 | 浏览器会请求 `/system/position`，**服务器必须返回 index.html** |
| 配置 | 无需额外配置 | 需要服务端 fallback |

本项目 `src/router/index.js:210` 用的是：

```js
mode: 'history', // 去掉url中的#
```

### 6.2 history 模式的坑：刷新 404

开发环境：`vue.config.js` 的 devServer 默认会把找不到的路径回退到 `index.html`，所以本地刷新没问题。

生产环境：如果 Nginx 没配，直接访问 `/system/position` 会 404，因为服务器上根本没有这个文件。Nginx 需要类似：

```nginx
location / {
  try_files $uri $uri/ /index.html;
}
```

**面试常问**：「history 模式刷新 404 怎么解决？」答案就是「服务端把所有前端路由回退到 index.html」，以及「不想配就用 hash 模式」。

> 本项目前端代码里没有 Nginx 配置（部署配置在代码仓库之外），这是需要知道的部署知识，不要编造项目里有这段配置。

---

## 七、重定向、别名与 404

### 7.1 `redirect`

`src/router/index.js:99-103`：

```js
{
  path: '',
  component: Layout,
  redirect: "/index",
  children: [ ... ]
}
```

访问 `/` 自动跳到 `/index`。

`src/router/index.js:112-116`：

```js
{
  path: '/user',
  component: Layout,
  hidden: true,
  redirect: 'noredirect',
  // ...
}
```

`noRedirect` 是 RuoYi 系的约定值（不是 Vue Router 的内置值），表示「这个父路由本身不跳转，在面包屑里显示为不可点击的文字」。

面包屑组件确实在判断它，`src/components/Breadcrumb/index.vue:5`：

```html
<span v-if="item.redirect === 'noRedirect'" class="no-redirect">{{ item.meta.title }}</span>
```

`:41` 也会把 `redirect === 'noRedirect'` 的项过滤掉。

**但这里有一处真实的不一致（坦白点）**：`router/index.js:17` 的注释写的是 `noRedirect`，面包屑代码判断的也是 `'noRedirect'`，而 `/user` 这条路由 `:116` 写的是小写 `'noredirect'`。两者对不上，所以这个「不可点击」的约定对 `/user` 实际不生效。好在 `/user` 自己没有 `meta.title`，本来就不会进面包屑，问题没有暴露出来。读代码时这种大小写不一致值得留意。

### 7.2 404 兜底

动态路由生成完后，会追加一条通配路由：

`src/store/modules/permission.js:97`：

```js
rewriteRoutes.push({ path: '*', redirect: '/404', hidden: true })
```

`*` 匹配所有前面没匹配到的路径，统一跳到 404 页。

**注意顺序**：通配路由要放在最后。如果它在前，所有路径都会先被它接住。

### 7.3 `/redirect`：刷新页面的「中间站」

路由表里有一条看起来很奇怪的路由 `src/router/index.js:33-43`：

```js
{
  path: '/redirect',
  component: Layout,
  hidden: true,
  children: [
    {
      path: '/redirect/:path(.*)',
      component: () => import('@/views/redirect')
    }
  ]
}
```

它不显示在菜单里，作用是一个「中转页」。`src/views/redirect.vue:1-11`：

```js
export default {
  created() {
    const { params, query } = this.$route
    const { path } = params
    this.$router.replace({ path: '/' + path, query })
  },
  render: function(h) {
    return h() // avoid warning message
  }
}
```

进入 `/redirect/xxx` 后，它立刻 `replace` 回 `/xxx`。

**为什么要绕这一圈？**

想「刷新当前页面」时，直接 `push` 到同一个地址是无效的（路由没变化，组件不会重建）。先跳到 `/redirect/当前路径`，再回到原路径，路由确实变化了两次，组件就会重新创建、重新请求数据。

`src/layout/components/TagsView/index.vue:199-203`：

```js
if (view.name === 'Dashboard') {
  // to reload home page
  this.$router.replace({ path: '/redirect' + view.fullPath })
}
```

关闭最后一个标签页、需要重载首页时，就用了这个技巧。

> 另一种刷新方式是第 4 章提过的 `$EventBus.$emit('reloadPage')`（`Breadcrumb/index.vue:60`），由页面自己监听并重新请求。项目里两种方式并存。

### 7.4 `hidden`

`hidden: true` 只影响**侧边栏是否显示**，不影响路由是否可用。登录页、404、编辑页等都在路由表里，但不出现在菜单中。

---

## 八、路由懒加载

`src/router/index.js:46`：

```js
component: () => import('@/views/login')
```

这不是「立即导入」，而是返回一个函数。只有真正访问 `/login` 时，webpack 才去加载对应的 JS 文件（单独打成一个 chunk）。

好处：首屏只加载当前页面需要的代码，不把几十个页面一次性塞进主包。

对比静态导入：

```js
import Login from '@/views/login'          // 打进主包，首屏就加载
component: Login

component: () => import('@/views/login')   // 懒加载，访问时才加载
```

项目里几乎所有路由都用懒加载。

> 后台菜单路由的组件是字符串（后端只返回 `"system/position/index"`），项目用 `loadView` 把它转成动态 import：
> `src/store/modules/permission.js:217-224`
> ```js
> export const loadView = (view) => {
>   if (process.env.NODE_ENV === 'development') {
>     return (resolve) => require([`@/views/${view}`], resolve)
>   } else {
>     return () => import(`@/views/${view}`)
>   }
> }
> ```
> 开发环境用 `require`，生产环境用 `import()` 实现懒加载。

---

## 九、导航守卫：路由的「安检口」

### 9.1 是什么

每次路由跳转，Vue Router 允许你在**进入之前**、**进入之后**插入函数，用来做登录校验、权限校验、埋点、进度条。

| 类型 | 写法 | 位置 |
|---|---|---|
| 全局前置 | `router.beforeEach((to, from, next) => {})` | 所有跳转 |
| 全局后置 | `router.afterEach((to, from) => {})` | 跳转完成后 |
| 路由独享 | `beforeEnter` 写在路由对象里 | 单条路由 |
| 组件内 | `beforeRouteEnter` / `beforeRouteUpdate` / `beforeRouteLeave` | 组件里 |

本项目主要用全局守卫，写在 `src/permission.js`。

### 9.2 三个参数

```js
router.beforeEach((to, from, next) => {
  // to   ：要去的路由
  // from ：从哪来的路由
  // next ：放行控制器，必须调用
})
```

`next` 的几种调用方式：

| 写法 | 含义 |
|---|---|
| `next()` | 放行，进入目标路由 |
| `next(false)` | 中断这次跳转，停留在原地 |
| `next('/login')` 或 `next({ path: '/login' })` | 跳到另一个地址 |
| `next({ ...to, replace: true })` | 重新进入目标路由，替换当前历史记录 |

**`next` 必须且只能调用一次**，否则页面会卡住或报错。

### 9.3 本项目的完整流程

`src/permission.js:52-166`，按分支读：

```js
router.beforeEach((to, from, next) => {
  NProgress.start()

  if (to.path === '/sso') {
    // ① 单点登录特殊入口，处理 token 后跳转
  } else if (getToken()) {
    // ② 有 token
    if (to.path === '/login') {
      next({ path: '/' })               // 已登录还去登录页 → 回首页
    } else {
      if (store.getters.roles.length === 0) {
        // ③ 有 token 但还没拉过用户信息（刚登录 / 刚刷新）
        store.dispatch('GetInfo').then(() => {
          store.dispatch('GenerateRoutes').then(accessRoutes => {
            router.addRoutes(accessRoutes)       // 动态加路由
            next({ ...to, replace: true })       // 重新进入目标路由
          })
        })
      } else {
        next()                          // ④ 信息已就绪，直接放行
      }
    }
  } else {
    // ⑤ 没有 token
    if (whiteList.indexOf(to.path) !== -1) {
      next()                            // 白名单直接进
    } else {
      next(`/login?redirect=${to.fullPath}`)  // 其余全部踢到登录页
    }
  }
})
```

白名单 `src/permission.js:41`：

```js
const whiteList = ['/login', '/auth-redirect', '/bind', '/register', '/wxLogin', '/wxCallback']
```

### 9.4 为什么有 token 还要再判断 `roles`

`getToken()` 只说明浏览器里有 token（存在 Cookie 里）。但用户信息、权限、菜单还没拉到（比如刚刷新页面，Vuex 里的状态被清空了）。

所以：

```
有 token，但 store.getters.roles.length === 0
  → 先 GetInfo 拉用户信息
  → 再 GenerateRoutes 拉菜单、生成路由
  → addRoutes 加进去
  → 重新进入目标路由
```

`roles` 一有值，后续跳转就走 `next()`，不再重复请求。

### 9.5 `next({ ...to, replace: true })` 为什么是「hack」

当前导航开始时，动态路由还没加进路由表。如果直接 `next()`，Vue Router 可能匹配不到目标路由（尤其刷新时直接访问 `/system/position`）。

`addRoutes` 之后再 `next({ ...to, replace: true })`，等于「**取消刚才那次导航，用最新的路由表重新匹配一遍**」。`replace: true` 避免在历史记录里多出一条。

这是 Vue Router 3 时代 RuoYi 系项目的标准写法。

### 9.6 `afterEach` 关进度条

`src/permission.js:170-172`：

```js
router.afterEach(() => {
  NProgress.done()
})
```

`beforeEach` 里 `NProgress.start()` 开顶部进度条，跳转完成关掉。

### 9.7 没登录被踢走时，`redirect` 怎么起作用

`src/permission.js:162`：

```js
next(`/login?redirect=${to.fullPath}`)
```

`to.fullPath` 是「路径 + 查询参数」。于是登录页地址变成：

```
/login?redirect=%2Fsystem%2Fposition%3FpageNum%3D1
```

登录页监听 `$route` 把 `redirect` 存下来（`src/views/login.vue:181-188`，第 3 章讲过）：

```js
this.redirect = route.query && route.query.redirect
```

登录成功后：

`src/views/login.vue:276`：

```js
this.$router.push({ path: this.redirect || '/' }).catch(() => { })
```

跳回原页面。这条链路把守卫、`query`、`watch`、`router.push` 全串起来了。

### 9.8 前端守卫能替代后端鉴权吗

**不能。**

前端守卫只控制「界面让不让你进」。用户完全可以绕过界面，直接调接口。所以：

- 前端：`beforeEach` + `v-hasPermi` 控制页面和按钮的可见性；
- 后端：`@SaCheckPermission` 等项目里的权限注解做真正的拦截。

第 2 章 §9.2 已经强调过一遍，这里再从路由角度强调一次。

---

## 十、动态路由：后端菜单怎么变成前端页面

这是本项目权限体系的核心，也是面试高频。

### 10.1 完整链路

```
① 用户登录成功，token 存进 Cookie
   （第 6 章讲 Login action）

② 刷新任意页面，进入 router.beforeEach
   → getToken() 为真
   → roles 为空（Vuex 被清空过）

③ store.dispatch('GetInfo')
   → 请求用户信息、角色、权限码
   → 存进 Vuex

④ store.dispatch('GenerateRoutes')
   → src/store/modules/permission.js:33-51
   → getRouters() 请求后端菜单接口
   → 拿到一个 JSON 数组，每个节点长这样：
      {
        name: 'Position',
        path: '/system/position',
        component: 'system/position/index',
        meta: { title: '岗位管理', icon: '...' }
      }

⑤ filterAsyncRouter 递归转换
   → 'Layout'     → 真正的 Layout 组件
   → 'ParentView' → 目录占位组件
   → 'system/position/index' → loadView('system/position/index')
                                  → () => import('@/views/system/position/index')
   → 子路由递归处理

⑥ 追加 404 通配路由
   → { path: '*', redirect: '/404', hidden: true }

⑦ commit('SET_ROUTES', rewriteRoutes)
   → 存进 Vuex，侧边栏菜单从这里渲染

⑧ router.addRoutes(accessRoutes)
   → 把动态路由真正加进路由表

⑨ next({ ...to, replace: true })
   → 用新路由表重新匹配当前地址
```

对应代码：

`src/store/modules/permission.js:75-111`：

```js
GenerateRoutes({ commit }) {
  return new Promise(resolve => {
    getRouters().then(res => {
      const sdata = JSON.parse(JSON.stringify(res.data))
      const rdata = JSON.parse(JSON.stringify(res.data))
      const sidebarRoutes = filterAsyncRouter(sdata)
      const rewriteRoutes = filterAsyncRouter(rdata, false, true)
      const asyncRoutes = filterDynamicRoutes(dynamicRoutes);
      rewriteRoutes.push({ path: '*', redirect: '/404', hidden: true })
      router.addRoutes(asyncRoutes);
      commit('SET_ROUTES', rewriteRoutes)
      commit('SET_SIDEBAR_ROUTERS', constantRoutes.concat(sidebarRoutes))
      // ...
      resolve(rewriteRoutes)
    })
  })
}
```

为什么把 `res.data` 深拷贝两份（`JSON.parse(JSON.stringify(...))`）？因为一份要转成侧边栏用的结构，一份要转成路由用的结构，处理方式不同，不能互相污染。

### 10.2 `filterAsyncRouter` 在转什么

`src/store/modules/permission.js:124-161`：

```js
function filterAsyncRouter(asyncRouterMap, lastRouter = false, type = false) {
  return asyncRouterMap.filter(route => {
    if (type && route.children) {
      route.children = filterChildren(route.children)
    }
    if (route.component) {
      if (route.component === 'Layout') {
        route.component = Layout
      } else if (route.component === 'ParentView') {
        route.component = ParentView
      } else if (route.component === 'InnerLink') {
        route.component = InnerLink
      } else {
        route.component = loadView(route.component)
      }
    }
    if (route.children != null && route.children && route.children.length) {
      route.children = filterAsyncRouter(route.children, route, type)
    } else {
      delete route['children']
      delete route['redirect']
    }
    return true
  })
}
```

关键点：后端返回的是**字符串**（`"Layout"`、`"system/position/index"`），而 Vue Router 要的是**组件对象或返回 Promise 的函数**。这一步就是「字符串 → 组件」的翻译。

### 10.3 `filterDynamicRoutes` 在做什么

`src/store/modules/permission.js:191-205`：

```js
export function filterDynamicRoutes(routes) {
  const res = []
  routes.forEach(route => {
    if (route.permissions) {
      if (auth.hasPermiOr(route.permissions)) {
        res.push(route)
      }
    } else if (route.roles) {
      if (auth.hasRoleOr(route.roles)) {
        res.push(route)
      }
    }
  })
  return res
}
```

遍历前端写死的 `dynamicRoutes`，用当前用户的权限码/角色过滤，有权限的才加入路由表。没权限的页面，地址栏直接访问也匹配不到。

> 同样要记住：这是体验层拦截，后端接口仍然要自己校验。

### 10.4 侧边栏和数据的关系

`SET_SIDEBAR_ROUTERS`（`src/store/modules/permission.js:104`）：

```js
commit('SET_SIDEBAR_ROUTERS', constantRoutes.concat(sidebarRoutes))
```

侧边栏从 Vuex 的 `sidebarRouters` 读取（第 6 章），遍历渲染菜单。所以「后端菜单 → 路由表 → 侧边栏」三件事用的是同一份数据的不同加工结果。

### 10.5 完整案例：登录 → 拿菜单 → 动态注册 → 守卫放行

> 目标：用 100 多行代码复现项目的核心链路——**前端只写死登录页和 404，业务页面的路由全部由「后端菜单」在登录后动态注册**。  
> 用法：新建 `router-demo.html`，整段复制后用浏览器打开（用 `file://` 打开也能跑，因为用的是 hash 模式）。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>动态路由 + 守卫完整案例</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 24px; max-width: 760px; }
    button { padding: 6px 12px; margin-right: 8px; cursor: pointer; }
    nav { margin: 12px 0; padding: 10px; background: #f5f5f5; border-radius: 6px; }
    nav a { margin-right: 12px; }
    ol { background: #fafafa; padding: 12px 12px 12px 36px; border-radius: 6px; }
    li { font-family: Consolas, monospace; font-size: 13px; line-height: 1.7; }
  </style>
</head>
<body>
  <div id="app">
    <h2>动态路由 + 导航守卫</h2>

    <p>
      <button @click="login">登录（写入 token）</button>
      <button @click="logout">退出（清 token）</button>
    </p>

    <nav>
      <router-link to="/">首页</router-link>
      <router-link to="/contract/draft">合同草拟（动态）</router-link>
      <router-link to="/system/role">角色管理（动态）</router-link>
    </nav>

    <router-view></router-view>

    <h3>执行日志</h3>
    <ol>
      <li v-for="(item, index) in logs" :key="index">{{ item }}</li>
    </ol>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/vue@2.6.12/dist/vue.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/vue-router@3.4.9/dist/vue-router.js"></script>
  <script>
    // ---------- 1. 页面组件 ----------
    const Home = { template: '<div><h3>首页</h3><p>公共路由，登录与否都能看。</p></div>' }
    const Login = { template: '<div><h3>登录页</h3><p>没有 token 时会被守卫踢到这里。</p></div>' }
    const NotFound = { template: '<div><h3>404</h3><p>路由表里没有这个地址。</p></div>' }
    const Layout = { template: '<div><h3>Layout 外壳</h3><router-view /></div>' }
    const Draft = { template: '<div><h3>合同草拟</h3><p>这个路由是登录后从“后端菜单”加进来的。</p></div>' }
    const Role = { template: '<div><h3>角色管理</h3><p>同样是动态加的。</p></div>' }

    // 后端菜单里的 component 是字符串，前端要映射成真正的组件
    const componentMap = {
      Layout: Layout,
      'contract/draft': Draft,
      'system/role': Role
    }

    // ---------- 2. 前端只写死这几条（对应 constantRoutes） ----------
    const router = new VueRouter({
      mode: 'hash',
      routes: [
        { path: '/', component: Home },
        { path: '/login', component: Login },
        { path: '/404', component: NotFound }
      ]
    })

    // ---------- 3. 模拟后端 /getRouters 返回的菜单树 ----------
    const fakeMenus = [
      {
        path: '/contract',
        component: 'Layout',
        meta: { title: '合同管理' },
        children: [
          { path: 'draft', component: 'contract/draft', meta: { title: '合同草拟' } }
        ]
      },
      {
        path: '/system',
        component: 'Layout',
        meta: { title: '系统管理' },
        children: [
          { path: 'role', component: 'system/role', meta: { title: '角色管理' } }
        ]
      }
    ]

    function getRouters() {
      // 用 300ms 模拟一次接口请求
      return new Promise(resolve => {
        setTimeout(() => resolve(JSON.parse(JSON.stringify(fakeMenus))), 300)
      })
    }

    // ---------- 4. 把菜单树转成路由表（对应 filterAsyncRouter） ----------
    function filterAsyncRouter(menus) {
      return menus.map(menu => {
        const route = {
          path: menu.path,
          component: componentMap[menu.component],
          meta: menu.meta
        }
        if (menu.children && menu.children.length) {
          route.children = filterAsyncRouter(menu.children)
        }
        return route
      })
    }

    // ---------- 5. 极简 store：token + 是否已加过路由 ----------
    let token = ''
    let routesAdded = false
    const logs = []

    function log(message) {
      logs.push(message)
      console.log(message)
    }

    // ---------- 6. 全局前置守卫 ----------
    router.beforeEach((to, from, next) => {
      log('守卫拦截：' + to.path)

      // 白名单：登录页直接放行
      if (to.path === '/login') {
        return next()
      }

      // ① 没有 token：踢到登录页
      if (!token) {
        log('没有 token，改道 /login')
        return next('/login')
      }

      // ② 有 token 但还没加过动态路由
      if (!routesAdded) {
        log('开始拉取菜单…')
        return getRouters().then(menus => {
          const accessRoutes = filterAsyncRouter(menus)
          // 关键：404 兜底必须最后加，否则会抢先匹配所有路径
          router.addRoutes(accessRoutes.concat([{ path: '*', redirect: '/404' }]))
          routesAdded = true
          log('addRoutes 完成，重新走一遍 ' + to.path)
          // 关键：当前这次跳转是在路由还没加进去时匹配的，必须重新匹配一次
          return next({ ...to, replace: true })
        })
      }

      // ③ 路由已就绪，正常放行
      next()
    })

    // ---------- 7. 根实例 ----------
    new Vue({
      el: '#app',
      router,
      data: { logs: logs },
      methods: {
        login() {
          token = 'fake-token'
          routesAdded = false
          log('登录成功，写入 token')
          this.$router.push('/contract/draft').catch(() => {})
        },
        logout() {
          token = ''
          routesAdded = false
          log('退出登录，清掉 token（真实项目里会刷新页面重置路由表）')
          this.$router.push('/login').catch(() => {})
        }
      }
    })
  </script>
</body>
</html>
```

**第一次进入 `/contract/draft` 时发生了什么**：

```
① 守卫拦截 /contract/draft
② 没有 token        → next('/login')                  ← 被踢走
③ 点“登录”           → token = 'fake-token'
④ push /contract/draft
⑤ 守卫拦截           → 有 token，但 routesAdded = false
⑥ getRouters()       → 模拟请求，300ms 后返回菜单树
⑦ filterAsyncRouter  → 把 component 字符串换成组件
⑧ addRoutes(...)     → 路由表里现在才有 /contract/draft
⑨ next({...to})      → 重新匹配一次，这次匹配到了，正常进入
```

**为什么必须 `next({ ...to, replace: true })` 再走一遍？**

第 ⑤ 步守卫执行时，`/contract/draft` **还不在路由表里**，这次跳转已经按「没有这条路由」算过了。`addRoutes` 只是往表里加，不会自动重算当前跳转。所以要让守卫**带着同一个目标重新走一遍**，第二次 `routesAdded` 已经是 `true`，才会真正放行。

> `replace: true` 是为了不留下「第一次没匹配上」的历史记录，用户按后退不会回到那个中间状态。

**回到项目**：

| 案例里的东西 | 项目真实代码 |
|---|---|
| 写死的公共路由 | `src/router/index.js:32` 的 `constantRoutes` |
| 需要权限才显示的动态路由 | `src/router/index.js` 的 `dynamicRoutes`（文件后半段） |
| 路由实例 + history 模式 | `src/router/index.js:209-213`（`mode: 'history'` 在 `:210`） |
| 拉菜单、生成路由、存 sidebarRouters | `src/store/modules/permission.js:75-111` 的 `GenerateRoutes` |
| 菜单树 → 路由表 | 同文件里的 `filterAsyncRouter`（第 5 章 §10.2 逐行讲过） |
| 守卫：token → GetInfo → GenerateRoutes | `src/permission.js:88-117` |
| `addRoutes` | `src/permission.js:108` |
| `next({ ...to, replace: true })` | `src/permission.js:116` |

**动手验证**：

1. 直接打开页面（地址栏是 `.../router-demo.html#/`），点「合同草拟」——被踢到登录页；
2. 点「登录」，会自动跳到 `/contract/draft`，日志里能看到「addRoutes 完成，重新走一遍」；
3. 在地址栏手动改成 `#/system/role`——不用重新登录，直接能进（路由已经加过了）；
4. 点「退出」再点「角色管理」——又被踢回登录页；
5. 把 `next({ ...to, replace: true })` 改成 `next()`，再刷新页面重新登录，观察是不是会落到 404 —— 这就是项目里那行 hack 存在的理由。

---

## 十一、常见问题与项目里的真实处理

### 11.1 刷新后动态路由没了

刷新会清空 Vuex，`roles` 变空，守卫会重新走 `GetInfo` + `GenerateRoutes` + `addRoutes`。所以刷新后菜单会重新出现。

### 11.2 重复 `addRoutes` 会怎样

如果守卫逻辑写错，每次跳转都加一遍，路由表会越来越长，出现重复路由警告。项目用 `roles.length === 0` 做闸门，只加一次。

### 11.3 `push` 到当前路由报 `NavigationDuplicated`

项目统一在 `push` 后加 `.catch(() => {})`（`src/views/index.vue:241`、`login.vue:276`）。原因见 §4.2。

### 11.4 路由 `name` 和组件 `name` 不一致，`keep-alive` 不生效

第 3 章 §7.6 讲过。`router/index.js:18` 的注释也专门提醒了。

### 11.5 `params` 刷新后丢失

只有写进 `path` 的动态段（`:id`）刷新才保留。如果跳转时用命名路由传 `params` 而路径里没有对应段，刷新就会丢。本项目用动态段和 `query`，避开了这个问题。

### 11.6 打包后访问子路由 404

history 模式的部署问题（§6.2），需要服务端 fallback。

---

## 十二、动手练习

### 练习 1：写一个最小路由

用 CDN 写一个可以切换两个页面的例子：

```html
<div id="app">
  <router-link to="/">首页</router-link>
  <router-link to="/about">关于</router-link>
  <router-view></router-view>
</div>

<script src="https://cdn.jsdelivr.net/npm/vue@2.6.12/dist/vue.js"></script>
<script src="https://cdn.jsdelivr.net/npm/vue-router@3.4.9/dist/vue-router.js"></script>
<script>
const Home = { template: '<div>这是首页</div>' }
const About = { template: '<div>这是关于页</div>' }

const router = new VueRouter({
  routes: [
    { path: '/', component: Home },
    { path: '/about', component: About }
  ]
})

new Vue({
  el: '#app',
  router
})
</script>
```

要求：

1. 把 `mode` 改成 `history`，观察地址栏变化；
2. 加一个 `/user/:id` 路由，在页面里用 `$route.params.id` 显示 id；
3. 加一个 `beforeEach`，访问 `/about` 时在控制台打印一句话。

### 练习 2：读懂守卫

打开 `src/permission.js:52-166`，不看本章解释，自己回答：

1. 没登录访问 `/system/position`，会发生什么？
2. 已登录但 Vuex 里 `roles` 是空的，会发生什么？
3. 已登录访问 `/login`，会发生什么？
4. `next({ ...to, replace: true })` 如果换成 `next()`，可能出什么问题？

### 练习 3：加一条公共路由

在 `src/router/index.js` 的 `constantRoutes` 里加一个 `/about-me` 页面（自己随便写个最简单的 `.vue`），`hidden: true`。

验收：

1. 地址栏直接访问能打开；
2. 侧边栏不出现；
3. 做完删掉。

### 练习 4：跟踪一次动态路由

打开浏览器 DevTools 的 Network 面板，重新登录后：

1. 找到菜单接口（`getRouters` 对应的请求）；
2. 看返回的 JSON 里 `component` 字段是什么；
3. 在 `src/store/modules/permission.js:124-161` 里找到它被转换成组件的位置；
4. 打开 Vue Devtools，看 Vuex 里 `permission/routes` 和 `permission/sidebarRouters` 的区别。

### 练习 5：复现登录跳回

1. 退出登录；
2. 直接在地址栏访问 `/system/position`；
3. 观察被踢到登录页后 URL 上的 `redirect` 参数；
4. 登录，观察是否回到 `/system/position`；
5. 在 `src/permission.js:162` 和 `src/views/login.vue:276` 找到对应代码。

---

## 十三、本章自测清单

- [ ] `new Router({ mode: 'history' })` 和 Vue Router 4 的写法有什么区别？
- [ ] `constantRoutes`、`dynamicRoutes`、后端菜单路由分别在什么时候生效？
- [ ] 嵌套路由里，`Layout` 和业务页面各渲染到哪个 `<router-view>`？
- [ ] `AppMain.vue` 里 `:key="$route.path"` 有什么用？
- [ ] `$router` 和 `$route` 的区别？
- [ ] `query` 和 `params` 的区别？刷新后分别会怎样？
- [ ] history 模式刷新 404 的原因和解决办法？
- [ ] `beforeEach` 的 `to`、`from`、`next` 分别是什么？
- [ ] `next()`、`next(false)`、`next('/login')`、`next({ ...to, replace: true })` 分别是什么效果？
- [ ] 没登录时，`redirect` 参数是怎么产生、怎么被登录页用上的？
- [ ] 后端菜单的 `component: "system/position/index"` 是怎么变成组件的？
- [ ] 为什么 `addRoutes` 之后要 `next({ ...to, replace: true })`？
- [ ] 前端路由守卫能替代后端权限校验吗？为什么？
- [ ] `push` 到当前路由为什么会报 `NavigationDuplicated`？项目怎么处理的？

---

## 十四、下一章前瞻

路由解决了「页面怎么切换」，但登录用户、权限码、菜单、标签页这些**多个页面共享的状态**放哪？第 6 章讲 Vuex：

- `state`、`getters`、`mutations`、`actions`、`modules` 分别是什么；
- 登录时 `Login` action 做了什么；
- 为什么刷新后 `roles` 会空、`GetInfo` 又把它填回来；
- `GenerateRoutes` 生成的菜单怎么进入侧边栏；
- `mapState` / `mapGetters` 怎么在组件里取值；
- `commit` 和 `dispatch` 的区别。

> 这一章最重要的一句话：**URL 决定显示哪个组件；守卫决定你能不能进；动态路由决定你有哪些页面——而这三件事的数据来源，一半在路由表，一半在 Vuex。**
