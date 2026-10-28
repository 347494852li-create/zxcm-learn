# 第 6 章 · Vuex 全局状态管理

> **这一章解决什么问题**：登录用户、角色、权限码、菜单、标签页，这些数据被顶栏、侧边栏、路由守卫、各个业务页面同时使用。放在哪个组件里都不合适。Vuex 就是用来放这种「全站共享状态」的仓库。
>
> **学完标准**：能看懂 `src/store/` 下的每个文件；能说清「登录 → 存 token → 拉用户信息 → 生成菜单 → 侧边栏渲染」的全过程；能区分 `state`、`getters`、`mutations`、`actions`、`modules`；能自己往 store 里加一个状态。

---

## 一、为什么需要 Vuex

### 1.1 先看问题

登录后，下面这些地方都要用到「当前用户」：

| 位置 | 要用什么 |
|---|---|
| 顶栏 `Navbar` | 头像、昵称 |
| 侧边栏 `Sidebar` | 菜单路由、待办角标 |
| 路由守卫 `permission.js` | 有没有 token、角色拉没拉过 |
| 各个页面 | 权限码（`v-hasPermi`） |
| 首页 | 昵称、部门、统计数字 |

如果靠 `props` 一层层传，组件树会变成「传参马拉松」；靠 `$EventBus`，数据没有统一存放点，谁都能改，出问题很难查。

Vuex 的思路：

> **把所有共享状态放进一个全局仓库（store），任何组件都能读；但只能按规定的流程改。**

### 1.2 一句话理解 Vuex

```
组件 → dispatch(action) → commit(mutation) → 改 state → 组件自动更新
```

- **state**：仓库里的数据；
- **getters**：仓库数据的「计算属性」；
- **mutations**：唯一允许改 state 的地方，必须同步；
- **actions**：放业务逻辑和异步请求，最后 `commit` 一个 mutation；
- **modules**：把一个巨大的仓库拆成多个小仓库。

> Vuex 3 配套 Vue 2（`package.json:72` 是 3.6.0）。Vue 3 用的是 Vuex 4 或 Pinia，写法不同。本项目没有用 Pinia。

---

## 二、Store 是怎么创建的

### 2.1 安装插件

`src/store/index.js:1-11`：

```js
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)
```

和 `Vue.use(Router)` 一样，`Vue.use(Vuex)` 把 Vuex 装到 Vue 上。

### 2.2 创建仓库

`src/store/index.js:13-23`：

```js
const store = new Vuex.Store({
  modules: {
    app,
    user,
    notify,
    tagsView,
    permission,
    settings
  },
  getters
})
```

六个模块：

| 模块 | 文件 | 管什么 |
|---|---|---|
| `app` | `store/modules/app.js` | 侧边栏开合、设备类型、组件尺寸 |
| `user` | `store/modules/user.js` | token、用户信息、角色、权限码 |
| `notify` | `store/modules/notify.js` | 通知数量 |
| `tagsView` | `store/modules/tagsView.js` | 打开的标签页、需要缓存的页面 |
| `permission` | `store/modules/permission.js` | 路由表、侧边栏菜单 |
| `settings` | `store/modules/settings.js` | 主题、标题、公司配置 |

### 2.3 挂到根实例

`src/main.js:209-214`：

```js
new Vue({
  el: "#app",
  router,
  store,
  render: (h) => h(App),
})
```

挂上之后，所有组件里都能用：

```js
this.$store.state.xxx
this.$store.getters.xxx
this.$store.dispatch('xxx')
this.$store.commit('XXX')
```

---

## 三、五个核心概念

### 3.1 `state`：数据本身

`src/store/modules/user.js:56-66`：

```js
state: {
  token: getToken(),
  name: '',
  nickname: '',
  avatar: '',
  roles: [],
  permissions: [],
  userInfo: {},
  config: {},
  showNum: {},
}
```

注意第一行：

```js
token: getToken()
```

`getToken()` 从 Cookie 里读 token（`src/utils/auth.js`）。所以**刷新页面后 Vuex 虽然被清空，token 还能恢复**。这是「刷新不掉登录」的关键之一。

### 3.2 `getters`：store 的计算属性

`src/store/getters.js:30-82`：

```js
const getters = {
  sidebar: state => state.app.sidebar,
  device: state => state.app.device,
  visitedViews: state => state.tagsView.visitedViews,
  cachedViews: state => state.tagsView.cachedViews,
  token: state => state.user.token,
  avatar: state => state.user.avatar,
  name: state => state.user.name,
  nickname: state => state.user.nickname,
  roles: state => state.user.roles,
  permissions: state => state.user.permissions,
  sidebarRouters: state => state.permission.sidebarRouters,
  // ...
}
```

getters 的特点：

- 写法像 `computed`，依赖的 state 变了会自动重算；
- 组件里用 `this.$store.getters.roles` 读；
- 主要作用是「**给 state 起一个短名字**」或「对 state 做派生计算」。

例如 `roles` 本身在 `state.user.roles`，getter 把它提到顶层，组件里写 `this.$store.getters.roles` 更短，也能配合 `mapGetters`。

### 3.3 `mutations`：唯一能改 state 的地方

`src/store/modules/user.js:73-101`：

```js
mutations: {
  SET_TOKEN: (state, token) => {
    state.token = token
  },
  SET_NAME: (state, name) => {
    state.name = name
  },
  SET_ROLES: (state, roles) => {
    state.roles = roles
  },
  SET_PERMISSIONS: (state, permissions) => {
    state.permissions = permissions
  },
  // ...
}
```

调用方式：

```js
this.$store.commit('SET_TOKEN', 'xxx')
```

规则：

1. **只能通过 mutation 改 state**，不要在组件里 `this.$store.state.user.name = 'x'`；
2. mutation 必须**同步**。异步逻辑放 action；
3. 命名习惯：**全大写 + 下划线**（`SET_ROLES`、`ADD_VISITED_VIEW`）。

**为什么必须同步？** Vuex Devtools 会记录每次 mutation 前后的 state 快照，实现「时间旅行调试」。如果 mutation 里有异步回调，快照就对不上变化发生的时刻。

### 3.4 `actions`：业务逻辑和异步

`src/store/modules/user.js:109-162`：

```js
actions: {
  // 登录
  Login({ commit }, userInfo) {
    const username = userInfo.username.trim()
    const password = userInfo.password
    const code = userInfo.code
    const uuid = userInfo.uuid
    const tenantId = userInfo.tenantId
    return new Promise((resolve, reject) => {
      login(username, password, code, uuid, tenantId).then(res => {
        setToken(res.data.token)
        commit('SET_TOKEN', res.data.token)
        resolve()
      }).catch(error => {
        reject(error)
      })
    })
  }
}
```

拆解：

1. action 的第一个参数是 `context`，里面常用 `commit`、`dispatch`、`state`；
2. 这里调用了登录接口 `login(...)`（异步）；
3. 成功后先 `setToken(...)` 把 token 写进 Cookie，再 `commit('SET_TOKEN', ...)` 写进 Vuex；
4. 返回 Promise，组件里可以 `.then()` 等它完成。

调用方式：

```js
this.$store.dispatch('Login', this.loginForm).then(() => {
  this.$router.push({ path: this.redirect || '/' })
})
```

`src/views/login.vue:272-284` 就是这么写的。

### 3.5 `modules`：把仓库拆开

六个模块分别放在 `src/store/modules/` 下。根 store 只负责组装（`store/index.js:14-21`）。

不拆的话，`store/index.js` 会变成一个几百行的巨型文件，state、mutations、actions 全挤在一起。拆模块后：

- `user.js` 只管用户；
- `permission.js` 只管路由和菜单；
- `settings.js` 只管外观和公司配置。

### 3.6 `commit` 和 `dispatch` 的区别

| | `commit` | `dispatch` |
|---|---|---|
| 触发什么 | mutation | action |
| 能否异步 | 不能 | 能 |
| 组件里写法 | `this.$store.commit('SET_ROLES', roles)` | `this.$store.dispatch('GetInfo')` |
| 项目里组件常用哪个 | 很少直接用 | 大量使用 |

**一句话**：组件一般 `dispatch` 一个 action，action 内部再 `commit` 一个 mutation。只有特别简单的同步修改，才直接在组件里 `commit`。

---

## 四、组件里怎么使用 store

### 4.1 直接读取 `state`

`src/views/login.vue:170-179`：

```js
computed: {
  copyright() {
    return this.$store.state.settings.copyright
  },
  companyName() {
    return this.$store.state.settings.companyName
  },
  systemName() {
    return this.$store.state.settings.systemName
  }
}
```

模板里就能用 `{{ companyName }}`。state 变了，computed 自动更新，模板跟着更新。

### 4.2 直接读取 `getters`

`src/views/index.vue:158-174`：

```js
computed: {
  avatar() {
    return this.$store.getters.avatar
  },
  name() {
    return this.$store.getters.name
  },
  nickname() {
    return this.$store.getters.nickname
  },
  deptName() {
    const userInfo = this.$store.state.user.userInfo || {}
    return (userInfo.dept && userInfo.dept.deptName) || userInfo.deptName || ''
  }
}
```

同一个页面里，既有 `getters` 也有 `state` 的写法，都合法。

### 4.3 `mapState` / `mapGetters`：批量映射

如果一个个写 `computed` 太啰嗦，可以用辅助函数。

`src/layout/index.vue:22`：

```js
import { mapState } from 'vuex'
```

`:36-44`：

```js
computed: {
  ...mapState({
    theme: state => state.settings.theme,
    sideTheme: state => state.settings.sideTheme,
    sidebar: state => state.app.sidebar,
    device: state => state.app.device,
    needTagsView: state => state.settings.tagsView,
    fixedHeader: state => state.settings.fixedHeader
  }),
  classObj() { ... }
}
```

`...` 是对象展开（第 0 章 §5.9）：把 `mapState` 返回的一组 computed 合并进当前 `computed`。

`src/layout/components/Sidebar/index.vue:43-55`：

```js
import { mapGetters, mapState } from "vuex";

export default {
  computed: {
    ...mapState(["settings"]),
    ...mapGetters(["userInfo"]),
    ...mapGetters(["sidebarRouters", "sidebar"]),
    // ...
  }
}
```

- `mapState(["settings"])` 会生一个 computed `settings`，等价于 `this.$store.state.settings`；
- `mapGetters(["sidebarRouters"])` 会生一个 computed `sidebarRouters`，等价于 `this.$store.getters.sidebarRouters`。

模板里直接用：

`src/layout/components/Sidebar/index.vue:31-36`：

```html
<sidebar-item
  v-for="(route, index) in sidebarRouters"
  :key="route.path + index"
  :item="route"
  :base-path="route.path"
/>
```

**侧边栏菜单就是遍历 `sidebarRouters` 渲染出来的。**

### 4.4 `mapActions` / `mapMutations`

项目里没怎么用，但要知道：

```js
import { mapActions } from 'vuex'

methods: {
  ...mapActions(['Login', 'LogOut']),
  // 之后可以直接 this.Login(form)
}
```

### 4.5 四种写法怎么选

| 写法 | 适合 |
|---|---|
| `this.$store.state.xxx` | 临时读一个值 |
| `this.$store.getters.xxx` | 读派生值或短名字 |
| 手写 computed 包一层 | 需要在组件里加工 |
| `mapState` / `mapGetters` | 一次要拿很多个 |

---

## 五、一次登录的完整 Vuex 链路

这是本章最重要的部分，把 login 页面、user 模块、permission 模块、路由守卫串起来。

### 5.1 第 1 步：用户点登录

`src/views/login.vue:272-284`：

```js
login() {
  this.$store
    .dispatch('Login', this.loginForm)
    .then(() => {
      this.$router.push({ path: this.redirect || '/' }).catch(() => { })
    })
    .catch(() => {
      this.loading = false
      if (this.captchaOnOff) {
        this.getCode()
      }
    })
}
```

### 5.2 第 2 步：`Login` action 拿 token

`src/store/modules/user.js:117-162`：

```js
Login({ commit }, userInfo) {
  return new Promise((resolve, reject) => {
    login(username, password, code, uuid, tenantId).then(res => {
      setToken(res.data.token)          // ① 写进 Cookie
      commit('SET_TOKEN', res.data.token) // ② 写进 Vuex
      resolve()
    }).catch(error => {
      reject(error)
    })
  })
}
```

此时：

- Cookie 里有 token（刷新还在）；
- Vuex 的 `user.token` 有值；
- 但**用户信息、角色、权限、菜单都还没有**。

### 5.3 第 3 步：跳转触发路由守卫

`login.vue:276` 执行 `this.$router.push('/')`，进入 `router.beforeEach`（第 5 章 §9）。

`src/permission.js:78-117`：

```js
} else if (getToken()) {
  // 有 token
  if (store.getters.roles.length === 0) {
    store.dispatch('GetInfo').then(() => {
      store.dispatch('GenerateRoutes').then(accessRoutes => {
        router.addRoutes(accessRoutes)
        next({ ...to, replace: true })
      })
    })
  }
}
```

刚登录时 `roles` 是空数组，所以会走 `GetInfo`。

### 5.4 第 4 步：`GetInfo` 拉用户信息

`src/store/modules/user.js:170-227`：

```js
GetInfo({ commit, state }) {
  return new Promise((resolve, reject) => {
    getInfo().then(res => {
      const user = res.data.user
      // 头像兜底
      if (res.data.roles && res.data.roles.length > 0) {
        commit('SET_ROLES', res.data.roles)
        commit('SET_PERMISSIONS', res.data.permissions)
      } else {
        commit('SET_ROLES', ['ROLE_DEFAULT'])
      }
      commit('SET_USERINFO', user)
      commit('SET_NAME', user.userName)
      commit('SET_NICKNAME', user.nickName)
      commit('SET_AVATAR', avatar)
      if (user.config) {
        commit('SET_CONFIG', JSON.parse(user.config))
      }
      resolve(res)
    })
  })
}
```

一次请求，把用户信息、角色、权限码全部写进 Vuex。

**细节**：如果后端返回的 `roles` 是空数组，会兜底成 `['ROLE_DEFAULT']`（`:78-80`），这样 `roles.length === 0` 的守卫条件就不会无限触发。但 `permissions` 没有兜底，保持空数组。

### 5.5 第 5 步：`GenerateRoutes` 生成菜单

`src/store/modules/permission.js:75-111`（第 5 章 §10 详细讲过）：

```js
getRouters().then(res => {
  // 后端菜单 JSON → 组件 → 路由
  commit('SET_ROUTES', rewriteRoutes)
  commit('SET_SIDEBAR_ROUTERS', constantRoutes.concat(sidebarRoutes))
  // ...
})
```

写入 Vuex：

- `permission.routes`：完整路由表；
- `permission.sidebarRouters`：侧边栏要渲染的菜单。

### 5.6 第 6 步：侧边栏自动更新

`Sidebar/index.vue` 的 computed `sidebarRouters` 来自 `mapGetters`，Vuex 一提交，computed 重算，模板重新渲染。

```
后端菜单 → Vuex permission.sidebarRouters → getters → Sidebar 模板
```

### 5.7 整条链路图

```
登录页 submit
  → dispatch('Login')
      → 登录接口
      → setToken(Cookie)
      → commit('SET_TOKEN')
  → router.push('/')
  → beforeEach 守卫
      → getToken() 有值
      → roles.length === 0
      → dispatch('GetInfo')
          → 用户信息接口
          → commit SET_ROLES / SET_PERMISSIONS / SET_USERINFO / ...
      → dispatch('GenerateRoutes')
          → 菜单接口
          → commit SET_ROUTES / SET_SIDEBAR_ROUTERS
      → addRoutes
      → next({ ...to, replace: true })
  → 页面渲染
      → Sidebar 读 sidebarRouters 渲染菜单
      → Navbar 读 avatar/nickname
      → 页面按钮按 permissions 显示
```

### 5.8 退出登录

`src/layout/components/Navbar.vue:127-128`：

```js
this.$store.dispatch('LogOut').then(() => {
  location.href = process.env.VUE_APP_CONTEXT_PATH + 'index'
})
```

`LogOut` action `src/store/modules/user.js:238-256`：

```js
LogOut({ commit, state }) {
  return new Promise((resolve, reject) => {
    logout(state.token).then(() => {
      commit('SET_TOKEN', '')
      commit('SET_ROLES', [])
      commit('SET_PERMISSIONS', [])
      removeToken()
      resolve()
    })
  })
}
```

三件事：

1. 调后端退出接口；
2. 清空 Vuex 里的 token、角色、权限；
3. 删除 Cookie 里的 token。

最后 `location.href = ...` 是**整页刷新**，内存里的 Vuex 和路由表全部重置，保证退干净。

> `FedLogOut`（`src/store/modules/user.js:267-275`）是「只清前端不调后端」的版本，用于单点登录切换账号（`src/permission.js:70` 的 `/sso` 分支）。

---

## 六、为什么刷新后菜单还在、角色还在

### 6.1 Vuex 是内存，刷新就没了

刷新浏览器 → JS 重新执行 → `new Vuex.Store(...)` 重新创建 → 所有 state 回到初始值。

那为什么刷新后不用重新登录、菜单也还在？

### 6.2 token 从 Cookie 恢复

`src/utils/auth.js` 用 `js-cookie` 存取 token（`package.json:52`）。`src/store/modules/user.js:57`：

```js
token: getToken()
```

模块初始化时就从 Cookie 读回来。

### 6.3 角色和菜单重新拉

刷新后 `roles` 是空数组，路由守卫发现后重新执行 `GetInfo` + `GenerateRoutes`（§5.3），再把菜单重新加进路由表。

所以刷新后的完整过程是：

```
刷新
  → Vuex 清空，但 Cookie 里 token 还在
  → 守卫：getToken() 有值
  → roles 为空 → GetInfo → GenerateRoutes → addRoutes
  → 页面正常显示
```

**这就是为什么「有 token 但 roles 为空」这个分支必须存在**。它不是给「刚登录」准备的，更多是给「刷新页面」准备的。

---

## 七、命名空间 `namespaced`

### 7.1 两种模块

项目里的六个模块分成两类：

| 模块 | `namespaced` | dispatch 写法 |
|---|---|---|
| `app` | `true`（`src/store/modules/app.js:52`） | `dispatch('app/toggleSideBar')` |
| `settings` | `true`（`src/store/modules/settings.js:74`） | `dispatch('settings/getCompanyConfig')` |
| `tagsView` | `true`（`tagsView.js:203`） | `dispatch('tagsView/addView')` |
| `user` | 没写（默认 false） | `dispatch('Login')`、`dispatch('GetInfo')` |
| `permission` | 没写 | `dispatch('GenerateRoutes')` |
| `notify` | 没写 | 直接 dispatch action 名 |

### 7.2 有命名空间时

`src/store/modules/settings.js:74`：

```js
export default {
  namespaced: true,
  state,
  mutations,
  actions
}
```

于是：

- action 要带模块前缀：`this.$store.dispatch('settings/getCompanyConfig')`（`login.vue:192`）；
- mutation 要带前缀：`commit('settings/CHANGE_SETTING', ...)`；
- **state 仍然按模块名挂在根上**：`this.$store.state.settings.copyright`（`login.vue:171`）；
- 根 getters 不受影响，因为 `getters.js` 直接写 `state.settings.xxx`。

### 7.3 没有命名空间时

`user.js` 没有 `namespaced`，所以：

```js
this.$store.dispatch('Login')
this.$store.dispatch('GetInfo')
this.$store.dispatch('LogOut')
```

不带 `user/` 前缀。如果加上反而找不到。

### 7.4 怎么判断

打开模块文件末尾：

```js
export default {
  namespaced: true,   // 有这行就要带前缀
  state,
  mutations,
  actions
}
```

或者直接看别处怎么调用的，照着写。

> **读代码时的实用规则**：看到 `dispatch('settings/...')` 就知道 settings 是命名空间模块；看到 `dispatch('GetInfo')` 就知道 user 不是。

### 7.5 完整案例：两个模块，一个开命名空间、一个不开

> 目标：把项目里最容易混的两套写法放在同一个页面里跑——`app` 模块**开**命名空间（`dispatch('app/xxx')`），`user` 模块**不开**（`dispatch('Login')`）。  
> 用法：新建 `vuex-demo.html`，整段复制后用浏览器打开。

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <title>Vuex 模块与命名空间完整案例</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 24px; max-width: 720px; }
    button { padding: 6px 12px; margin-right: 8px; cursor: pointer; }
    .box { background: #f7f7f7; padding: 12px; border-radius: 6px; margin: 12px 0; }
    ol { background: #fafafa; padding: 12px 12px 12px 36px; border-radius: 6px; }
    li { font-family: Consolas, monospace; font-size: 13px; line-height: 1.7; }
  </style>
</head>
<body>
  <div id="app">
    <h2>Vuex 模块 + 命名空间</h2>

    <div class="box">
      <p>登录状态：<b>{{ isLoggedIn ? '已登录' : '未登录' }}</b></p>
      <p>用户名：<b>{{ name || '—' }}</b>；角色：<b>{{ roles.join(', ') || '—' }}</b></p>
      <p>侧边栏：<b>{{ sidebarOpened ? '展开' : '收起' }}</b></p>
    </div>

    <p>
      <button @click="login">模拟登录（无前缀 action）</button>
      <button @click="logout">退出（无前缀 action）</button>
      <button @click="toggleSideBar">切换侧边栏（带 app/ 前缀）</button>
    </p>

    <h3>执行日志</h3>
    <ol>
      <li v-for="(item, index) in logs" :key="index">{{ item }}</li>
    </ol>
  </div>

  <script src="https://cdn.jsdelivr.net/npm/vue@2.6.12/dist/vue.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/vuex@3.6.0/dist/vuex.js"></script>
  <script>
    Vue.use(Vuex)

    // ---------- 模块 1：app，开了命名空间 ----------
    const app = {
      namespaced: true,
      state: {
        sidebarOpened: true
      },
      mutations: {
        TOGGLE_SIDEBAR(state) {
          state.sidebarOpened = !state.sidebarOpened
        }
      },
      actions: {
        toggleSideBar({ commit }) {
          // 真实项目里这里还会写 Cookie；这里用 200ms 模拟异步
          setTimeout(() => commit('TOGGLE_SIDEBAR'), 200)
        }
      }
    }

    // ---------- 模块 2：user，没开命名空间 ----------
    const user = {
      state: {
        token: '',
        name: '',
        roles: []
      },
      mutations: {
        SET_TOKEN(state, token) {
          state.token = token
        },
        SET_INFO(state, { name, roles }) {
          state.name = name
          state.roles = roles
        },
        CLEAR(state) {
          state.token = ''
          state.name = ''
          state.roles = []
        }
      },
      actions: {
        Login({ commit }, { username }) {
          return new Promise(resolve => {
            setTimeout(() => {
              commit('SET_TOKEN', 'fake-token-' + username)
              resolve()
            }, 300)
          })
        },
        GetInfo({ commit }) {
          return new Promise(resolve => {
            setTimeout(() => {
              commit('SET_INFO', { name: '张三', roles: ['admin'] })
              resolve()
            }, 300)
          })
        },
        LogOut({ commit }) {
          commit('CLEAR')
        }
      }
    }

    // ---------- 根 store：把两个模块挂上去 ----------
    const store = new Vuex.Store({
      modules: { app, user },
      // 根 getters 是手写的汇总层（对应项目 src/store/getters.js）
      getters: {
        isLoggedIn: state => !!state.user.token,
        name: state => state.user.name,
        roles: state => state.user.roles,
        sidebarOpened: state => state.app.sidebarOpened
      }
    })

    // ---------- 组件 ----------
    new Vue({
      el: '#app',
      store,
      data: {
        logs: []
      },
      computed: {
        // mapGetters 把 store.getters 映射成组件里的计算属性
        ...Vuex.mapGetters(['isLoggedIn', 'name', 'roles', 'sidebarOpened'])
      },
      methods: {
        log(message) {
          this.logs.push(message)
          console.log(message)
        },
        login() {
          this.log('dispatch("Login")        ← user 没开命名空间，不加前缀')
          this.$store.dispatch('Login', { username: 'admin' })
            .then(() => {
              this.log('token = ' + this.$store.state.user.token)
              this.log('dispatch("GetInfo")     ← 同样不加前缀')
              return this.$store.dispatch('GetInfo')
            })
            .then(() => {
              this.log('roles = ' + this.$store.state.user.roles.join(','))
            })
        },
        logout() {
          this.log('dispatch("LogOut")       ← 不加前缀')
          this.$store.dispatch('LogOut')
        },
        toggleSideBar() {
          // app 开了命名空间：必须带 app/ 前缀
          this.log('dispatch("app/toggleSideBar")')
          this.$store.dispatch('app/toggleSideBar').then(() => {
            // state 永远按模块名挂在根上，不管开没开命名空间
            this.log('state.app.sidebarOpened = ' + this.$store.state.app.sidebarOpened)
          })
        }
      }
    })
  </script>
</body>
</html>
```

**一张表记住两套写法的差别**：

| 操作 | `app`（`namespaced: true`） | `user`（没开） |
|---|---|---|
| 发 action | `dispatch('app/toggleSideBar')` | `dispatch('Login')` |
| 提交 mutation | `commit('app/TOGGLE_SIDEBAR')` | `commit('SET_TOKEN')` |
| 读 state | `state.app.sidebarOpened` | `state.user.token` |
| 在组件里读 | `this.$store.state.app.sidebarOpened` | `this.$store.state.user.token` |
| 根 getters | 照样写 `state.app.xxx` | 照样写 `state.user.xxx` |

**三个容易混的点**：

1. **`state` 的层级和命名空间无关**：不管开不开，模块的 state 都会挂在 `state.模块名` 下面。
2. **命名空间只影响 `commit` / `dispatch` / 模块内 getters**，不影响 state 的路径。
3. **根 getters 不受命名空间影响**，因为它直接写 `state.user.token`、`state.app.sidebarOpened`。

**回到项目**：

| 案例里的东西 | 项目真实代码 |
|---|---|
| 根 store 挂 6 个模块 | `src/store/index.js:13-23` |
| 根 getters 手写汇总 | `src/store/getters.js`（第 9 章 §9.3 讲过它是手写汇总） |
| `app` 开了命名空间 | `src/store/modules/app.js:52`（`namespaced: true`） |
| `settings` 开了命名空间 | `src/store/modules/settings.js:74` |
| `user` 没开 | `src/store/modules/user.js`（文件末尾没有 `namespaced`） |
| 带前缀的 action | `src/views/login.vue:192`（`dispatch('settings/getCompanyConfig')`） |
| 不带前缀的 action | `src/permission.js:96`、`:102`（`dispatch('GetInfo')`、`dispatch('GenerateRoutes')`） |
| 组件里读 settings 的 state | `src/views/login.vue:171-172`（`this.$store.state.settings.copyright`） |

**动手验证**：

1. 点「模拟登录」，日志里能看到 `Login` 和 `GetInfo` **都没有前缀**；
2. 点「切换侧边栏」，日志里的 `dispatch` **必须带 `app/`**，去掉前缀会报 `unknown action type`；
3. 把 `app` 模块的 `namespaced: true` 删掉，再点「切换侧边栏」，观察报错——这就是为什么项目里两种写法不能混；
4. 打开 Vue Devtools → Vuex，能看到 `state.app.sidebarOpened` 和 `state.user.token` 两个分支。

---

## 八、Vuex 在项目里的其它落点

### 8.1 标签页与 `keep-alive`

`src/store/modules/tagsView.js:1-20`：

```js
const state = {
  visitedViews: [],
  cachedViews: []
}

const mutations = {
  ADD_VISITED_VIEW: (state, view) => { ... },
  ADD_CACHED_VIEW: (state, view) => {
    if (state.cachedViews.includes(view.name)) return
    if (view.meta && !view.meta.noCache) {
      state.cachedViews.push(view.name)
    }
  },
  // ...
}
```

- `visitedViews`：顶部标签页显示哪些页面；
- `cachedViews`：哪些页面要缓存。

`AppMain.vue:4`：

```html
<keep-alive :include="cachedViews">
```

`cachedViews` 这个 computed 来自 getter（`AppMain.vue:14-17`），getter 来自 `state.tagsView.cachedViews`。所以「打开一个标签页 → Vuex 记下页面名 → keep-alive 缓存它」。

`noCache: true` 的页面不会被缓存（`tagsView.js:17`）。

### 8.2 侧边栏开合

`app.js` 的 state 里 `sidebar.opened` 控制侧边栏展开/收起，`toggleSideBar` action 修改它，`layout/index.vue:45-52` 用 `classObj` 把状态变成 CSS 类名。

### 8.3 公司配置

`src/store/modules/settings.js:60-70`：

```js
getCompanyConfig({ commit, state }) {
  return new Promise((resolve, reject) => {
    getConfig()
      .then((response) => {
        commit("SET_COMPANYCONFIG", response.data);
      })
      .catch((error) => {
        reject(error);
      });
  });
}
```

`SET_COMPANYCONFIG` 除了写 state，还会把配置写进 Cookie（`src/store/modules/settings.js:40-46`），下次打开时能先用缓存值，避免页面标题闪烁。

登录页 `created` 里就会调用它（`login.vue:192`），路由守卫每次初始化权限后也会刷新一次（`src/permission.js:140`）。

---

## 九、项目里的真实细节（坦白点）

### 9.1 没有开启严格模式

`store/index.js:13-23` 只传了 `modules` 和 `getters`，没有 `strict: true`。所以即使有人在组件里直接改 `state`，Vue 也不会报错。规范靠自觉，不靠框架。

### 9.2 有个 action 直接改了 state

`src/store/modules/settings.js:56-58`：

```js
setTitle({ commit }, title) {
  state.title = title
}
```

action 直接改了 `state.title`，没有走 `commit`。因为没开严格模式，代码能跑，但不符合「只能通过 mutation 改 state」的约定。这是项目里的一处小瑕疵，读到时不用困惑。

### 9.3 `getters.js` 是手写汇总

项目没有用 `mapGetters` 的模块内 getters，而是在根 `getters.js` 里手写了一遍。好处是集中、好查；坏处是新增状态时要记得同步加一行。

### 9.4 持久化是「手工」的

项目没有装 `vuex-persistedstate` 之类的持久化插件，而是：

- token：`js-cookie`；
- 侧边栏状态、尺寸：Cookie；
- 布局设置：`localStorage`（`src/store/modules/settings.js:7`）；
- 公司配置：state + Cookie 双写。

其余状态（用户信息、菜单）刷新后一律重新请求。

---

## 十、什么时候不该用 Vuex

Vuex 不是「所有数据都往里塞」。判断标准：

| 数据 | 放哪 |
|---|---|
| 只有当前组件用 | 组件 `data` |
| 父子组件用 | `props` / `$emit` |
| 多个不相关页面共享 | Vuex |
| 需要跨页面长期保留，刷新也要有 | Vuex + Cookie/localStorage，或直接 Cookie/localStorage |
| 后端数据的缓存 | 一般不用 Vuex，页面自己请求；需要共享才放 |

项目里 `position/index.vue` 的 `tableData`、`params` 都放在组件 `data`，没有进 Vuex。因为它们只有这个页面用。反过来，登录用户、菜单、权限才进 Vuex。

> **一个常见误区**：把所有接口返回都塞进 Vuex。结果是 store 越来越大，数据来源越来越乱。Vuex 只放「真正全局」的东西。

---

## 十一、动手练习

### 练习 1：最小 Vuex

用 CDN 跑一个计数器：

```html
<div id="app">
  <p>count: {{ count }}</p>
  <p>double: {{ double }}</p>
  <button @click="increment">+1</button>
  <button @click="incrementAsync">1 秒后 +1</button>
</div>

<script src="https://cdn.jsdelivr.net/npm/vue@2.6.12/dist/vue.js"></script>
<script src="https://cdn.jsdelivr.net/npm/vuex@3.6.0/dist/vuex.js"></script>
<script>
const store = new Vuex.Store({
  state: { count: 0 },
  getters: {
    double: state => state.count * 2
  },
  mutations: {
    INCREMENT(state) {
      state.count++
    }
  },
  actions: {
    incrementAsync({ commit }) {
      setTimeout(() => commit('INCREMENT'), 1000)
    }
  }
})

new Vue({
  el: '#app',
  store,
  computed: {
    count() {
      return this.$store.state.count
    },
    double() {
      return this.$store.getters.double
    }
  },
  methods: {
    increment() {
      this.$store.commit('INCREMENT')
    },
    incrementAsync() {
      this.$store.dispatch('incrementAsync')
    }
  }
})
</script>
```

要求：

1. 在 `increment` 里改成直接 `this.$store.state.count++`，观察页面是否更新、Devtools 是否记录；
2. 把 `INCREMENT` 改成异步（`setTimeout` 里改 state），观察 Devtools 的 state 快照是否还对得上；
3. 说清 `commit` 和 `dispatch` 的分工。

### 练习 2：给 store 加一个状态

给 `user` 模块加一个 `deptName`：

1. `state` 里加 `deptName: ''`；
2. 加 mutation `SET_DEPTNAME`；
3. 在 `GetInfo` 成功回调里 `commit('SET_DEPTNAME', ...)`；
4. 在某个页面用 `this.$store.state.user.deptName` 显示；
5. 做完改回去。

验收：登录后显示部门名；刷新后重新请求也能显示。

### 练习 3：追踪一次登录

打开 Vue Devtools 的 Vuex 面板，退出登录后重新登录，观察 mutation 的触发顺序：

1. `SET_TOKEN` 什么时候出现？
2. 之后跟着哪些 `SET_*`？
3. `permission/SET_ROUTES`、`permission/SET_SIDEBAR_ROUTERS` 在哪一步？
4. 退出登录时清空了哪几个？

把顺序记下来，和 §5.7 的链路图对照。

### 练习 4：读懂命名空间

在项目里找出：

1. 三个 `dispatch('模块名/action')` 的调用；
2. 三个不带前缀的 `dispatch('action')` 的调用；
3. 打开对应模块文件，确认 `namespaced` 的值。

### 练习 5：回答刷新问题

不看本章，自己画一遍：

```
刷新页面
  → Vuex 的 token 为什么还在？
  → roles 为什么是空的？
  → 守卫做了哪两件事？
  → 菜单为什么又出现了？
```

画不出来就回看 §六。

---

## 十二、本章自测清单

- [ ] Vuex 的五个核心概念分别是什么？各自负责什么？
- [ ] 为什么 mutation 必须同步？异步逻辑放哪？
- [ ] `commit` 和 `dispatch` 的区别？
- [ ] 组件里读取 state 有哪几种写法？`mapState` 展开后是什么？
- [ ] `getters` 和组件的 `computed` 有什么相似之处？
- [ ] 登录时 `Login` action 做了哪两件事？为什么 token 要同时写 Cookie 和 Vuex？
- [ ] `GetInfo` 提交了哪些 mutation？roles 为空时怎么处理？
- [ ] `GenerateRoutes` 把什么写进了 Vuex？侧边栏从哪里读？
- [ ] 刷新后 Vuex 清空了，为什么还能保持登录和菜单？
- [ ] `namespaced: true` 对 dispatch 写法有什么影响？state 的访问路径变吗？
- [ ] `LogOut` 做了哪三件事？为什么最后要整页刷新？
- [ ] `cachedViews` 是怎么从 Vuex 一路传到 `keep-alive` 的？
- [ ] 哪些数据不该放 Vuex？
- [ ] 项目里哪一处 action 直接改了 state？为什么不报错？

---

## 十三、下一章前瞻

到这里，Vue 的语法、实例、组件、路由、状态管理都讲完了。但这些代码是怎么变成浏览器能跑的文件的？为什么 `import '@/api/...'` 里的 `@` 能识别？`.env.development` 里的 `/dev-api` 怎么生效？axios 请求怎么发出去、响应怎么回来？

第 7 章讲工程化与请求层：

- `package.json` 的 scripts 和依赖；
- Vue CLI 与 webpack 做了什么；
- `vue.config.js` 的别名、端口、代理；
- `main.js` 的全局装配；
- `.env` 环境变量；
- `src/utils/request.js` 的 axios 封装、拦截器、token、错误处理；
- 完整请求链路（并指向附录 B 的逐行精读）。

> 这一章最重要的一句话：**Vuex 是共享状态的唯一仓库：组件 dispatch action，action commit mutation，mutation 改 state，所有用到 state 的地方自动更新。**
