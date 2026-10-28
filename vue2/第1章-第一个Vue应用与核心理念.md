# 第 1 章 · 第一个 Vue 应用与核心理念

> **这一章解决什么问题**：不装任何环境，用一个 HTML 文件跑起 Vue，理解「声明式渲染」「数据驱动」「MVVM」这三个词到底是什么意思。
>
> **学完标准**：你能独立写出计数器、待办列表；能解释「为什么改数据页面就变了」；能看懂 `src/main.js` 最后那 6 行。

> 📌 **路径怎么找（先看这个，不然会找不到文件）**
>
> **本卷所有前端代码路径和行号，统一使用学习版：**
>
> ```text
> E:\Develop\project\zxcm-learn\zxcm-ui\
> ```
>
> 其中 12 个文件带 `★` 教学注释，**本卷行号以这个版本为准**。
>
> 文中写 `src/main.js:209-214`，意思是：
>
> ```text
> E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js  第 209-214 行
> ```
>
> 直接把完整路径复制到编辑器打开即可。**本卷不再引用答案册路径/行号**；如果你需要旧版对照，看[附录 · 注释版行号说明](../重塑版/前端重塑版/附录-注释版行号说明.md)。

---

## 一、准备：一个能跑 Vue 的 HTML 文件

新建一个文件夹，里面放 `index.html`：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <title>第一个 Vue 应用</title>
</head>
<body>
  <div id="app">
    <h1>{{ message }}</h1>
  </div>

  <!-- 引入 Vue 2.6.12 -->
  <script src="https://cdn.jsdelivr.net/npm/vue@2.6.12/dist/vue.js"></script>
  <script>
    new Vue({
      el: '#app',
      data: {
        message: '你好，Vue'
      }
    })
  </script>
</body>
</html>
```

双击用浏览器打开，页面上显示：**你好，Vue**。

这一小段代码已经包含了 Vue 的全部核心机制，下面逐个拆。

---

## 二、拆第一段代码

### 2.1 `new Vue({ ... })`：创建一个 Vue 实例

```js
new Vue({
  el: '#app',
  data: {
    message: '你好，Vue'
  }
})
```

| 选项 | 作用 |
|---|---|
| `el: '#app'` | 告诉 Vue：接管页面上 id 为 `app` 的那个元素 |
| `data: { ... }` | 这个实例的**数据**；`message` 是其中一个字段 |

`new Vue(...)` 返回一个**实例**，你可以把它理解成「这个页面区域的大脑」。

### 2.2 `{{ message }}`：插值表达式

```html
<h1>{{ message }}</h1>
```

`{{ }}` 叫**插值（interpolation）**：把里面表达式的值，作为文本显示出来。

它不只能放变量：

```html
{{ 1 + 2 }}              <!-- 3 -->
{{ message + '！' }}      <!-- 你好，Vue！ -->
{{ message.length }}      <!-- 7 -->
```

> 只能放**表达式**，不能放语句。`{{ if (x) { ... } }}` 是错的。

### 2.3 数据变，页面自动变

打开浏览器控制台（F12 → Console），输入：

```js
// 注意：需要先让实例暴露到全局才能这样访问
```

更直观的做法是把代码改成：

```js
const vm = new Vue({
  el: '#app',
  data: { message: '你好，Vue' }
})

window.vm = vm
```

然后在控制台执行：

```js
vm.message = '数据变了'
```

页面上的文字**立刻**变成「数据变了」。

你没有写任何「找到 h1、修改它的文字」的代码。这就是 Vue 的第一条核心：**数据驱动**。

---

## 三、计数器：把「行为」接上

完整例子：

```html
<div id="app">
  <h1>计数器：{{ count }}</h1>
  <button v-on:click="increment">加 1</button>
  <button v-on:click="decrement">减 1</button>
</div>

<script src="https://cdn.jsdelivr.net/npm/vue@2.6.12/dist/vue.js"></script>
<script>
new Vue({
  el: '#app',
  data: {
    count: 0
  },
  methods: {
    increment() {
      this.count++
    },
    decrement() {
      this.count--
    }
  }
})
</script>
```

### 3.1 `methods`：实例的方法

```js
methods: {
  increment() {
    this.count++
  }
}
```

- `methods` 里的函数可以直接在模板里绑定给事件；
- 函数里的 `this` 指向 Vue 实例，所以 `this.count` 就是 `data.count`。

### 3.2 `v-on:click`：事件绑定

```html
<button v-on:click="increment">加 1</button>
```

意思是：**点击这个按钮时，调用 `increment` 方法**。

`v-on:` 可以简写成 `@`：

```html
<button @click="increment">加 1</button>
```

项目里到处是这种写法，例如 `position/index.vue:74` 的 `@click="handleAddEdit()"`。

### 3.3 发生了什么

```
用户点击按钮
  → Vue 调用 increment()
  → this.count++ 修改数据
  → Vue 检测到 count 变了
  → 重新渲染模板里用到 count 的地方（{{ count }}）
  → 页面显示新数字
```

**你只写了「数据怎么变」，没写「界面怎么改」。** 这是和传统 DOM 操作最大的区别。

---

## 四、对比：命令式 vs 声明式

假设要实现「点击按钮，让标题变成当前计数」。

### 4.1 传统写法（命令式：一步步指挥 DOM）

```html
<h1 id="title">计数器：0</h1>
<button id="btn">加 1</button>

<script>
let count = 0
document.getElementById('btn').onclick = function() {
  count++
  document.getElementById('title').textContent = '计数器：' + count
}
</script>
```

你必须自己：

1. 找到按钮；
2. 绑定点击事件；
3. 修改计数；
4. **再找到标题，手动改它的文字**。

界面元素一多，这类「找元素 → 改元素」的代码会迅速膨胀，而且容易漏改。

### 4.2 Vue 写法（声明式：只描述结果）

```html
<h1>计数器：{{ count }}</h1>
<button @click="count++">加 1</button>
```

你只声明：**标题显示 count 的值**。count 怎么变、什么时候重新渲染，Vue 负责。

> **声明式渲染**：你描述「界面应该长什么样」，而不是「怎么一步步把界面改成那样」。

这就是 Vue 官方文档反复强调的「声明式渲染（Declarative Rendering）」。

---

## 五、三个必懂的概念

### 5.1 MVVM

| 字母 | 含义 | 在 Vue 里对应 |
|---|---|---|
| **M**odel | 数据 | `data` |
| **V**iew | 视图 | `<template>` |
| **V**iew**M**odel | 连接数据和视图的中间层 | Vue 实例本身 |

```
      数据变化
data  ──────────>  视图自动更新
  ^                     |
  |      用户输入        |
  └─────────────────────┘
```

Vue 实例就是那个「中间人」：它盯着数据，也盯着视图，两边任何一边变了，它负责同步另一边。

### 5.2 数据驱动

**不要直接操作 DOM，只操作数据。**

```js
// 不推荐（在 Vue 里）
document.querySelector('.title').textContent = '新标题'

// 推荐
this.title = '新标题'
```

项目里你几乎看不到 `document.querySelector`，原因就在这里。

### 5.3 响应式

**Vue 能感知数据变化，并自动更新用到它的界面。**

你在 `data` 里声明的属性，Vue 会把它们改造成「可观察的」：读取时记录「谁用了我」，修改时通知「用我的人更新」。

这就是第 10 章要讲的响应式原理。现在只需要知道：

- 在 `data` 里声明过的属性，修改会触发更新；
- 后来**直接新增**的属性、以及**直接用下标修改数组**，有时不会触发更新（第 10 章讲原因和解法）。

---

## 六、回到本项目：根实例是怎么创建的

### 6.1 `src/main.js` 的最后 6 行

```js
new Vue({
  el: "#app",
  router,
  store,
  render: (h) => h(App),
})
```

| 选项 | 作用 |
|---|---|
| `el: "#app"` | **指定挂载点**（Vue 2 会用它替换 `public/index.html:182` 的 `<div id="app">`） |
| `router` | 装上 Vue Router（第 5 章） |
| `store` | 装上 Vuex（第 6 章） |
| `render: (h) => h(App)` | **指定根组件** `App`，用 render 函数生成内容 |

> 📌 **`el` 和 `render` 不是二选一**：前者管「挂在哪」，后者管「渲染什么」。为什么两个都要、两个 `#app` 为什么不冲突，见 §6.3。

### 6.2 为什么用 `render` 而不是 `template`

根实例这里没有写 `template: '<App />'`，而是用了：

```js
render: (h) => h(App)
```

`h` 是 Vue 传进来的**创建元素函数**（createElement）。`h(App)` 的意思是「创建一个 App 组件实例」。

两种写法效果相同：

```js
// 写法一：render 函数
render: (h) => h(App)

// 写法二：模板（需要完整版 Vue，含编译器）
components: { App },
template: '<App />'
```

本项目用写法一，因为它不依赖模板编译器，运行时更小，也是 Vue CLI 项目的常见写法。

> 你现在不需要会写 render 函数，只要认得 `h(App)` 是「渲染根组件」。

### 6.3 `el` 和 `render` 为什么都要？两个 `#app` 不冲突吗？

这是初学者最容易卡住的一组问题。先把结论说清：

> **`el` 管「挂到哪个 DOM」，`render` 管「渲染哪个根组件」。它们不是二选一，而是「位置」和「内容」的关系。**

#### 6.3.1 用「挂画」来理解

```js
new Vue({
  el: "#app",                  // 挂载点：画挂在哪面墙
  render: (h) => h(App),       // 根组件：画什么内容
})
```

| 选项 | 回答的问题 | 本项目 |
|---|---|---|
| `el: "#app"` | 挂到哪个 DOM 元素上 | `public/index.html:182` 的 `<div id="app"></div>` |
| `render: (h) => h(App)` | 渲染哪个根组件 | `App.vue` |
| `router` / `store` | 给根实例注入路由和全局状态 | 子组件都能用 `this.$router` / `this.$store` |

Vue 2 挂载时大致做四步：

1. 用 `el` 找到挂载点 `<div id="app"></div>`；
2. 用 `render` 生成根组件 `App` 的 VNode；
3. 把 VNode patch 成真实 DOM；
4. 用这个真实 DOM **替换**掉挂载点。

#### 6.3.2 只写 `el` 会怎样？

`public/index.html` 里的 `#app` 是一个**空壳**：

```html
<div id="app"></div>
```

`el: "#app"` 只告诉 Vue「挂到这里」，但没告诉它「挂什么」。本项目用的又是 Vue 的 **runtime-only 构建**（`.vue` 的 `<template>` 在构建时已经被 vue-loader 编译成 render 函数，运行时没有模板编译器），所以不能靠 `el` 里的 HTML 当模板。

结果就是：**页面空白**，控制台可能提示 `template or render function not defined`。

所以 `render` 不是可选装饰，它是根实例的「内容来源」。

#### 6.3.3 两个 `<div id="app">` 为什么不冲突？

因为它们在**不同阶段**出现，不会同时存在于最终 DOM。直接看「挂载前」和「挂载后」的 DOM 就明白了。

**挂载前**（浏览器刚加载 `index.html`）：

```html
<body>
  <div id="app"></div>   <!-- public/index.html 里的占位空壳 -->
</body>
```

此时 DOM 里只有 **1 个** `#app`。

**执行 `new Vue({ el: '#app', render: h => h(App) })` 之后**：

Vue 2 的做法是：**用 App 组件的根元素替换掉 `el` 匹配到的那个元素**。

```html
<body>
  <div id="app">          <!-- App.vue 的根元素，替换了原来的占位 -->
    <router-view>...</router-view>
  </div>
</body>
```

最终 DOM 里仍然只有 **1 个** `#app`，只是它的来源从 `public/index.html` 变成了 `App.vue`。

| 阶段 | DOM 中的 `#app` | 来源 |
|---|---|---|
| 挂载前 | 1 个 | `public/index.html:182` 的占位 |
| 挂载后 | 1 个 | `App.vue:1-5` 的根元素 |
| 源码里 | 2 处 | 不同文件、不同阶段，不会共存 |

可以把它想成一个**空座位**：

- `public/index.html` 的 `<div id="app"></div>` 是空座位；
- `App.vue` 的根 `<div id="app">` 是坐下来的人；
- 人坐下后，空座位就被替换了，不会出现「两个座位」。

**为什么 `App.vue` 还要写 `id="app"`？** 因为 Vue 2 是**替换**挂载点。如果 `App.vue` 的根不写 `id="app"`，挂载后原来的 `#app` 就消失了，依赖 `#app` 的 CSS / JS 会找不到元素。所以让 `App.vue` 的根继续用同一个 id，挂载后选择器仍然有效。

#### 6.3.4 Vue 3 在这里不一样（容易踩的坑）

Vue 2 的 `el` 是「替换目标」，Vue 3 的 `mount` 是「挂载到容器内部」：

```js
// Vue 3
createApp(App).mount('#app')
```

Vue 3 会把 `App` 的内容**塞进** `#app` 容器里，而不是替换容器。如果 Vue 3 的 `App.vue` 根元素也写 `id="app"`，就会出现：

```html
<div id="app">
  <div id="app">...</div>  <!-- 嵌套重复 id -->
</div>
```

所以：

- **Vue 2**：`App.vue` 根写 `id="app"` 没问题（替换占位）；
- **Vue 3**：`App.vue` 根通常**不要**再写 `id="app"`（否则嵌套重复）。

本项目是 Vue 2，所以现在这样写是对的。

#### 6.3.5 自己验证一下

登录后打开浏览器 Console：

```js
document.querySelectorAll('#app').length
// 1
```

或者：

```js
document.getElementById('app').outerHTML
```

你会看到它就是 `App.vue` 渲染出来的根元素，里面是 `<router-view>` 渲染的页面。

> 对应本项目文件（**学习版行号**）：
> - `E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js:209-214`：`new Vue({ el, router, store, render })`
> - `E:\Develop\project\zxcm-learn\zxcm-ui\public\index.html:182`：占位 `<div id="app"></div>`
> - `E:\Develop\project\zxcm-learn\zxcm-ui\src\App.vue:1-5`：App 的根 `<div id="app">`
>
> 如果你读的是注释版，对应位置：
> - `E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js` 第 **209-214** 行；
> - `E:\Develop\project\zxcm-learn\zxcm-ui\public\index.html` 第 **182** 行。
>
> 更多行号换算见[附录 · 注释版行号说明](../重塑版/前端重塑版/附录-注释版行号说明.md)。

#### 6.3.6 三句话记住

1. **`el` 找位置，`render` 给内容；只给 `el` = 找到画框但没画。**
2. **两个 `#app` 是「占位 → 替换」的关系，最终 DOM 只有一个。**
3. **Vue 2 是替换，Vue 3 是塞进去；Vue 3 的 App 根别再写 `id="app"`。**

### 6.4 `App.vue` 是根组件

`src/App.vue:1-5`：

```vue
<template>
  <div id="app">
    <router-view />
  </div>
</template>
```

`<router-view />` 是一个占位符：**当前路由匹配到哪个页面组件，就在这里显示哪个**。

#### 6.4.1 `<script>` 里的 `export default` 是什么？

`App.vue` 的完整内容其实有两段：

```vue
<template>
  <div id="app">
    <router-view />
  </div>
</template>

<script>
export default {
  name: 'App',
  metaInfo() { ... }
}
</script>
```

- `<template>` 是**页面长什么样**；
- `<script>` 里的 `export default { ... }` 是**组件的配置对象**，告诉 Vue：这个组件叫什么、有哪些数据/方法/生命周期/额外选项。

vue-loader 会把 `<template>` 编译成 render 函数，再和 `export default` 这个对象合并，最终得到一个完整的组件。**它不是“又渲染了一次”，而是组件的定义。**

#### 6.4.2 `name: 'App'` 有什么用？

给组件起个名字，主要用在：

- Vue Devtools 的组件树里显示 `App`；
- 报错/警告里能看出是哪个组件；
- `<keep-alive include="App">` 这类按名字缓存；
- 组件递归引用自己时按名字找。

它**不是路由的 name**，和 `router/index.js` 里的路由名是两回事。

#### 6.4.3 `metaInfo()` 是干什么的？

`metaInfo` **不是 Vue 自带的选项**，它来自 `vue-meta` 插件。本项目在 `main.js` 里装了它：

```js
import VueMeta from "vue-meta";
Vue.use(VueMeta);
```

具体位置（直接按路径打开）：

- **学习版**：`E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js`
  - 第 **100** 行：`import VueMeta from "vue-meta";`
  - 第 **177** 行：`Vue.use(VueMeta);`

装了之后，组件里的 `metaInfo()` 会被 vue-meta 读取，用来**修改浏览器 `<head>` 里的标题和 meta 标签**。它管的是**浏览器标签页标题**，不是页面正文。

本项目的 `App.vue` 里：

```js
metaInfo() {
  return {
    title: this.$store.state.settings.dynamicTitle && this.$store.state.settings.title,
    titleTemplate: title => {
      return title ? `${title} - ${process.env.VUE_APP_TITLE}` : process.env.VUE_APP_TITLE
    }
  }
}
```

分两种情况：

| `dynamicTitle` | `title` 的值 | 最终浏览器标题 |
|---|---|---|
| `false`（项目默认） | `false` | 只有 `VUE_APP_TITLE`，即「合同管理系统」 |
| `true` | 当前路由的 `meta.title` | 「签约主体 - 合同管理系统」 |

几个关键点：

- `dynamicTitle` 和 `title` 都在 Vuex 的 `settings` 模块里：`E:\Develop\project\zxcm-learn\zxcm-ui\src\store\modules\settings.js`（这个文件没有加注释）。
- `dynamicTitle` 默认是 `false`：`E:\Develop\project\zxcm-learn\zxcm-ui\src\settings.js`（同样没有加注释）。
- 路由切换时，`permission.js` 会把 `to.meta.title` 写进 `settings.title`：
  - `E:\Develop\project\zxcm-learn\zxcm-ui\src\permission.js` 第 **79** 行。
  - 只有 `dynamicTitle` 为 `true`，标题才会跟着路由变。
- `process.env.VUE_APP_TITLE` 来自 `.env.development` / `.env.production`，本项目值是「合同管理系统」；它是**构建时**被替换进代码的，不是运行时读环境变量。
- `titleTemplate` 里的 `title ? ... : ...`：有路由标题就拼上「- 合同管理系统」，没有就只用应用名。

##### `process` 和 `process.env` 到底是什么？

`process` 是 **Node.js 的全局对象**，不是浏览器里的对象。你在浏览器 Console 里直接敲 `process` 会报 `ReferenceError`。

那为什么前端代码里能写 `process.env.VUE_APP_TITLE`？因为 Vue CLI 在**构建/启动 dev server 时**，用 webpack 把 `process.env.VUE_APP_TITLE` 替换成了字符串常量：

```js
// 你写的
process.env.VUE_APP_TITLE
// 构建后实际变成
"合同管理系统"
```

值来自项目根目录的环境变量文件：

- `E:\Develop\project\zxcm-learn\zxcm-ui\.env.development:2`
- `E:\Develop\project\zxcm-learn\zxcm-ui\.env.production:2`

两处都是：

```ini
VUE_APP_TITLE = 合同管理系统
```

几个关键规则：

- 只有 `VUE_APP_` 开头的变量会被注入到前端代码；`ENV` 这种没有前缀的不会。
- 改 `.env` 后必须**重启 dev server / 重新 build** 才生效，因为它在构建时就被替换了。
- `vue.config.js:10` 还有兜底：`process.env.VUE_APP_TITLE || '合同管理系统'`。
- 想要「运行时配置」，不能靠 `process.env`，要用接口下发或 `public/config.js`。

##### `titleTemplate` 是什么？`return` 把数据返回给谁？

`metaInfo()` 的 `return` 不是返回给 `<template>`，而是返回给 **vue-meta 插件**：

```js
metaInfo() {
  return {
    title: ...,
    titleTemplate: title => {
      return title ? `${title} - ${VUE_APP_TITLE}` : VUE_APP_TITLE
    }
  }
}
```

- `metaInfo()` 返回一个对象 → vue-meta 读取；
- `titleTemplate` 是一个函数，vue-meta 把当前 `title` 传给它；
- `titleTemplate` 的 `return` 返回最终字符串 → vue-meta 把它写进 `document.title` / `<head><title>`。

所以：

```text
metaInfo() 返回对象  →  vue-meta
titleTemplate 返回字符串  →  vue-meta  →  浏览器标签页标题
```

`titleTemplate` 的作用是「统一标题格式」。本项目格式是：

- 有页面标题：`页面标题 - 合同管理系统`
- 没有页面标题：`合同管理系统`

##### `permission.js` 把 `to.meta.title` 写进 `settings.title` 是什么意思？

`permission.js` 里有一段路由守卫：

```js
router.beforeEach((to, from, next) => {
  ...
  to.meta.title && store.dispatch('settings/setTitle', to.meta.title)
  ...
})
```

- `to` 是**即将进入的目标路由对象**；
- `to.meta` 是这个路由的元信息；
- `to.meta.title` 就是路由上写的标题，比如 `router/index.js:108` 的 `meta: { title: '首页' }`；
- `store.dispatch('settings/setTitle', ...)` 调用 Vuex 的 `settings/setTitle` action；
- 这个 action 在 `E:\Develop\project\zxcm-learn\zxcm-ui\src\store\modules\settings.js:56`，目前写的是 `state.title = title`（直接改 state，是个技术债，但能工作）。

于是链路是：

```text
路由切换
  → permission.js 读到 to.meta.title = "签约主体"
  → dispatch('settings/setTitle', "签约主体")
  → settings.title = "签约主体"
  → App.vue 的 metaInfo() 读到 settings.title
  → titleTemplate 拼成 "签约主体 - 合同管理系统"
  → vue-meta 写入浏览器标签页标题
```

注意：**只有 `dynamicTitle` 为 `true`，这条链路才会影响最终标题**；默认 `false` 时，`settings.title` 虽然被写了，但 `metaInfo` 里的 `dynamicTitle && title` 会短路成 `false`，最终仍显示「合同管理系统」。

##### 怎么从 `this.$store.state.settings` 找到文件？

记住一个口诀：

> **`$store` 看 `store/index.js`；`settings` 看 `modules` 里的 key；`.state.settings.xxx` 看 `modules/settings.js` 的 `state`。**

按这个顺序找：

1. **`$store` 从哪来？**
   - `E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js:47`：`import store from "./store";`
   - `main.js:137`：`new Vue({ store, ... })`
   - 所以任何组件里的 `this.$store` 都是 `src/store/index.js` 导出的那个 store。

2. **`state.settings` 里的 `settings` 从哪来？**
   - `E:\Develop\project\zxcm-learn\zxcm-ui\src\store\index.js:8`：`import settings from './modules/settings'`
   - `store/index.js:14-21`：
     ```js
     modules: {
       app, user, notify, tagsView, permission, settings
     }
     ```
   - `settings` 这个 key 就是 `state.settings` 的来源。

3. **`.state.settings.title` / `.dynamicTitle` 在哪定义？**
   - 打开 `E:\Develop\project\zxcm-learn\zxcm-ui\src\store\modules\settings.js`；
   - `:8-25` 的 `const state = { ... }` 就是它：
     - `:9` `title: ''`
     - `:17` `dynamicTitle: ...`

4. **`settings/setTitle` 在哪？**
   - 同一个文件的 `actions` 里，`:56` 的 `setTitle`；
   - `:74` `namespaced: true`，所以调用时要写 `settings/setTitle`、`settings/changeSetting`。

在 IDE 里最快的找法：

- 在 `store/index.js` 里搜 `settings`，Ctrl+点击 `import settings from './modules/settings'`；
- 或全局搜 `dynamicTitle`，直接跳到 `src/store/modules/settings.js:17`；
- 浏览器里用 Vue Devtools → Vuex → state → settings，能看到运行时的值。

##### `const storageSetting = JSON.parse(localStorage.getItem('layout-setting')) || ''` 怎么读？

这一行在：

```text
E:\Develop\project\zxcm-learn\zxcm-ui\src\store\modules\settings.js:7
```

拆成四步：

1. **`localStorage`**：浏览器自带的本地存储，按「域名」隔离，关闭浏览器后还在。
2. **`localStorage.getItem('layout-setting')`**：读 key 为 `layout-setting` 的字符串；没存过时返回 `null`。
3. **`JSON.parse(...)`**：把 JSON 字符串转成对象。
   - 如果读出来是 `null`，`JSON.parse(null)` 返回 `null`；
   - 如果读出来是合法 JSON 字符串，返回对象；
   - 如果读出来是空字符串或坏 JSON，会抛异常（这里没有 try/catch）。
4. **`|| ''`**：如果左边是 `null`（没存过），就用空字符串 `''` 兜底。

所以：

```js
// 第一次打开，没存过
localStorage.getItem('layout-setting')  // null
JSON.parse(null)                        // null
null || ''                              // ''
// storageSetting = ''

// 用户保存过布局设置
localStorage.getItem('layout-setting')
// '{"topNav":false,"dynamicTitle":true,...}'
JSON.parse(...)                         // 对象
```

后面初始化 state 时：

```js
theme: storageSetting.theme || '#4361ee',
dynamicTitle: storageSetting.dynamicTitle === undefined ? dynamicTitle : storageSetting.dynamicTitle,
```

- `storageSetting` 是 `''` 时，`''.theme` 是 `undefined`，所以走默认值；
- `storageSetting` 是对象时，优先用用户保存的值。

**这个 key 是谁写进去的？**

布局设置页：

```text
E:\Develop\project\zxcm-learn\zxcm-ui\src\layout\components\Settings\index.vue
```

**`saveSetting()`（`:182-196`）**：把布局选项拼成 JSON 字符串，写进 `localStorage`。

```js
saveSetting() {
  this.$modal.loading('正在保存到本地，请稍候...')
  this.$cache.local.set(
    'layout-setting',
    `{
        "topNav":${this.topNav},
        "tagsView":${this.tagsView},
        "fixedHeader":${this.fixedHeader},
        "sidebarLogo":${this.sidebarLogo},
        "dynamicTitle":${this.dynamicTitle},
        "sideTheme":"${this.sideTheme}",
        "theme":"${this.theme}"
      }`
  )
  setTimeout(this.$modal.closeLoading(), 1000)
}
```

**`resetSetting()`（`:198-201`）**：删除 key 并刷新页面。

```js
resetSetting() {
  this.$modal.loading('正在清除设置缓存并刷新，请稍候...')
  this.$cache.local.remove('layout-setting')
  setTimeout('window.location.reload()', 1000)
}
```

`$cache.local` 是对 `localStorage` 的简单封装，见：

```text
E:\Develop\project\zxcm-learn\zxcm-ui\src\plugins\cache.js
```

它的 `localCache.set/get/remove` 内部就是 `localStorage.setItem/getItem/removeItem`。

##### `this.$cache.local.set` 是怎么找到的？

你直接搜 `this.$cache.local.set` 是搜不到的，因为它是**运行时拼出来的属性链**。要按下面顺序找：

1. **`$cache` 是谁挂上去的？**
   - `E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js:50`：`import plugins from "./plugins";`
   - `main.js:175`：`Vue.use(plugins);`
   - `E:\Develop\project\zxcm-learn\zxcm-ui\src\plugins\index.js:14`：
     ```js
     Vue.prototype.$cache = cache
     ```
   - 所以任何组件里的 `this.$cache` 都是 `plugins/cache.js` 导出的那个对象。

2. **`.local` 是什么？**
   - `E:\Develop\project\zxcm-learn\zxcm-ui\src\plugins\cache.js:68-77`：
     ```js
     export default {
       session: sessionCache,
       local: localCache
     }
     ```
   - 所以 `this.$cache.local` 就是 `localCache`。

3. **`.set` 在哪？**
   - `plugins/cache.js:34-42`：
     ```js
     const localCache = {
       set (key, value) {
         if (!localStorage) {
           return
         }
         if (key != null && value != null) {
           localStorage.setItem(key, value)
         }
       },
       ...
     }
     ```
   - 所以 `this.$cache.local.set('layout-setting', jsonString)` 最终执行的就是：
     ```js
     localStorage.setItem('layout-setting', jsonString)
     ```

**完整调用链**：

```text
this.$cache
  ↑ Vue.prototype.$cache = cache（plugins/index.js:14）
  → .local
  ↑ export default { local: localCache }（src/plugins/cache.js:76）
  → .set(key, value)
  ↑ localCache.set（src/plugins/cache.js:35）
  → localStorage.setItem(key, value)（src/plugins/cache.js:40）
```

**IDE 里怎么找**：

- 全局搜 `Vue.prototype.$cache`，会直接到 `plugins/index.js:14`；
- 再打开同目录的 `cache.js`，搜 `localCache`；
- 不要搜 `this.$cache` 字面量，它是运行时才有的。

**两个小坑**：

- `|| ''` 用空字符串兜底能工作，但语义上不如 `|| {}` 清楚；
- `JSON.parse` 没有 `try/catch`，如果用户/脚本把 `layout-setting` 改成坏 JSON，store 初始化会报错。

如果去掉 `metaInfo`：页面正文照常渲染，只是 vue-meta 不再管理浏览器标签页标题。

如果去掉 `Vue.use(VueMeta)`：`metaInfo` 就只是一个普通对象选项，vue-meta 不工作，标题也不会变。

所以整条链路是：

```
index.html 的 <div id="app">
  → main.js 的 new Vue 接管
  → 渲染 App 组件
  → App 里的 <router-view /> 显示当前路由页面
  → 当前路由页面（如 position/index.vue）显示具体内容
```

第 5 章会把这条链路完整走一遍。

### 6.5 还有一堆 `Vue.use` 和 `Vue.prototype`

`main.js` 前半部分做了两类「全局安装」：

```js
Vue.use(Element, { size: ... })       // 安装 Element UI 组件库
Vue.component('Pagination', Pagination) // 全局注册组件
Vue.prototype.parseTime = parseTime     // 挂全局方法，任何组件里 this.parseTime 可用
```

这些在第 4、7 章分别讲。现在只需要知道：**`main.js` 是全站的公共装配车间**。

---

## 七、为什么组件里的 `data` 必须是函数

先埋一个伏笔。根实例里 `data` 是对象：

```js
new Vue({
  data: { count: 0 }
})
```

但组件里必须是函数：

```js
export default {
  data() {
    return { count: 0 }
  }
}
```

原因：组件会被复用多份。如果 `data` 是对象，所有组件实例会**共享同一个数据对象**，改一个全变；用函数则每次创建实例都返回一个**新对象**，彼此隔离。

第 4 章会结合项目代码再讲一遍。

---

## 八、动手练习

### 练习 1：计数器

不看上面的代码，从零写一个计数器，要求：

- 显示当前数字；
- 「+1」「-1」「归零」三个按钮；
- 数字小于 0 时显示红色。

提示：红色可以用 `:class`（第 2 章详解，这里可以先试）。

### 练习 2：待办列表

要求：

- 一个输入框 + 一个「添加」按钮；
- 输入内容后点添加，加入列表；
- 列表用 `v-for` 渲染（第 2 章会讲，先照猫画虎）；
- 每项后面有「删除」按钮。

```html
<div id="app">
  <input v-model="text" />
  <button @click="add">添加</button>
  <ul>
    <li v-for="(item, index) in todos" :key="index">
      {{ item }}
      <button @click="remove(index)">删除</button>
    </li>
  </ul>
</div>
```

### 练习 3：解释给外行听

用三句话向一个没学过前端的人解释：

1. 什么是 Vue？
2. 什么是数据驱动？
3. 为什么改数据页面会变？

### 练习 4：对照项目

打开 `E:\Develop\project\zxcm-learn\zxcm-ui\src\main.js:209-214` 和 `E:\Develop\project\zxcm-learn\zxcm-ui\src\App.vue`，画出：

```
index.html 的 div#app
  → new Vue
  → App 组件
  → <router-view>
  → 某个页面
```

这条链路。画不出来就回看 §6。

---

## 九、本章自测清单

- [ ] `new Vue({ el, data, methods })` 里三个选项分别是什么？
- [ ] `{{ }}` 里能放什么，不能放什么？
- [ ] `v-on:click` 和 `@click` 是什么关系？
- [ ] 什么是声明式渲染？它和直接操作 DOM 有什么区别？
- [ ] MVVM 三个字母分别对应 Vue 里的什么？
- [ ] 数据驱动是什么意思？
- [ ] 响应式是什么意思？它保证了什么？
- [ ] `el` 和 `render` 分别管什么？只写 `el` 不写 `render` 会怎样？
- [ ] 为什么 `public/index.html` 和 `App.vue` 都有 `id="app"`，却不冲突？
- [ ] Vue 2 和 Vue 3 的挂载方式有什么区别？为什么 Vue 3 的 `App.vue` 根通常不再写 `id="app"`？
- [ ] `render: (h) => h(App)` 在做什么？
- [ ] `App.vue` 里的 `<router-view />` 是什么？
- [ ] `.vue` 文件里的 `export default { ... }` 是在做什么？它和 `<template>` 是什么关系？
- [ ] `name: 'App'` 有什么用？它是路由的 name 吗？
- [ ] `metaInfo()` 来自哪个插件？它管的是页面正文还是浏览器标签页标题？
- [ ] `dynamicTitle` 为 `false` 和 `true` 时，浏览器标题分别是什么？
- [ ] 为什么组件里的 `data` 必须是函数？

---

## 十、下一章前瞻

第 2 章会把模板里能用的语法一次讲全：

- `{{ }}` 的完整规则；
- `v-bind` / `:`、`v-on` / `@`、`v-model`；
- `v-if` / `v-show`、`v-for` / `:key`；
- 事件修饰符、表单修饰符；
- 项目模板里那些 `slot-scope`、`v-loading`、`v-hasPermi` 到底是什么。

> 这一章最重要的一句话：**Vue 让你只改数据，界面自动跟着变；你描述结果，不指挥过程。**
