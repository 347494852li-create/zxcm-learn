# 精读 2 · 前端代码导读 · 逐行读懂 `.vue`

> **本篇定位**：属于《前端重塑版》的精读五篇（返回[五篇导航](00-这五篇怎么读.md)）；主线章节见[前端重塑版 README](../README.md)。
>
> **路径约定**：文中 `main.js` / `permission.js` / `utils/request.js` 等短名默认指**答案册** `zhaoxin-contract/zxcm-ui/src/` 下的文件；带完整路径的以完整路径为准。
>
> ⚠️ **行号说明（2026-10-19）**：`zxcm-learn/zxcm-ui` 的 12 个前端文件已加入 560 处 `★` 教学注释，**代码逻辑没变，但行号整体后移**。本文行号默认以答案册为准；读注释版请搜 `★`，或看[附录 · 注释版行号说明](../附录-注释版行号说明.md)。


> **这份文档给谁看**：一打开 `zxcm-ui/src/views/conf/signingSubject/index.vue` 这类页面文件就发懵的人 ——
> 满屏 `<el-xxx>`、`v-model`、`:prop`、`@click`、`@click.native`、`scope.row`，不知道每行在说什么，也不知道一个页面从哪读起。
>
> **读完你会得到**：拿到任意一个 `.vue` 文件，能**逐行说出它在干什么**，能顺着它找到它调的接口。
>
> **讲法**：每个概念先用**前端自己的话**讲清"它是什么、怎么用"，再配**本项目真实代码**（就是你在 `zxcm-learn/zxcm-ui` 里能打开的那些文件）当例子。
>
> **和 [精读 1 · zxcm-ui 工程结构](01-工程结构速查.md) 的分工**：
> - **精读 1 回答「文件在哪」** —— 目录结构、菜单怎么变成页面、按钮怎么找到接口。
> - **本文回答「文件里写了什么」** —— 语法是什么、组件是什么、每一行在做什么。
>
> 两者一起看：先用 精读 1 定位到文件，再用本文读懂文件。
>
> 🧭 **本文只负责「文件里写了什么」（语法 / 组件 / 逐行读法）**。**不重复讲**：一次请求的完整链路（→ [精读 4](04-一次点击的完整旅程.md)）、前端环境类坑（→ [精读 1 §12](01-工程结构速查.md#12-踩坑速查前端)）。
>
> **和主线章节怎么对上**：`.vue` 骨架 ↔ [第 0 章 §4.2](../第0章-前端全景与运行机制.md)；模板语法/指令 ↔ [第 2 章](../第2章-路由·菜单·按钮权限.md)；`JTable`/字典/CRUD 组件 ↔ [第 3 章](../第3章-通用组件与前端的约定.md)、[第 4 章](../第4章-基础数据管理页面.md)。
>

---

## 0. 前端是怎么"动"起来的

先说清前端**最根本的一个特点**：它不是你请求一次、返回一次、就结束的一条线，
而是**常驻在浏览器里、数据一变界面就跟着变**的东西。三条要点先立住：

| # | 要点 | 说人话 |
|---|---|---|
| 1 | **数据驱动界面，不是你手动刷** | 你只管改 `this.xxx`，界面**自己**跟着变。几乎不写"刷新一下页面"这种代码——改数据，界面就动 |
| 2 | **一个 `.vue` 文件 = 一个页面** | 一个文件里装齐三样：长什么样（`<template>`）+ 什么逻辑（`<script>`）+ 什么样式（`<style>`） |
| 3 | **你只声明"发生什么时执行谁"** | `@click="handleAdd"` = "点了就执行 `handleAdd`"；`:data="postList"` = "表格的数据来自 `postList`"。**谁触发、什么时候触发、怎么画到屏幕上，都不用你写** |

一句话记住整个前端的运行方式：

```
数据 ←→ 界面（双向绑定）：数据在内存里活着，改哪哪动 —— 你改数据，界面自己就变
```

---

## 1. 一个 `.vue` 文件长什么样（骨架）

**每个 `.vue` 文件都是同一个三段式结构**。认识这三段，你就不会再"不知道从哪读起"：

```vue
<template>
  <!-- ① 长什么样：HTML + Vue 指令。表格、按钮、弹窗都在这 -->
</template>

<script>
export default {
  // ② 什么逻辑：数据、方法、生命周期。一个"类"的字面量写法
}
</script>

<style scoped>
/* ③ 什么样式：CSS。scoped = 只作用于本文件，不污染别的页面 */
</style>
```

这三段各自是什么：

| 段 | 它是什么 | 一句话 |
|---|---|---|
| `<template>` | **页面的"长相"**：写 HTML + Vue 指令，描述表格、按钮、弹窗长什么样 | 里面的数据是**活的**——绑定的变量一变，这块界面自动重画 |
| `<script>` | **页面的"逻辑"**：`export default { ... }` 导出一个对象，里面装着这个页面的状态、方法、生命周期 | 这个对象就是整个页面的定义 |
| `<style scoped>` | **页面的"样式"**：CSS | `scoped` = 只作用于本文件、不污染别的页面；一般不用管 |

> 🔑 **读文件的固定顺序**：先扫 `<script>` 的 `data()`（**这个页面有哪些状态**）→ 再扫 `methods`（**能做哪些事**）→
> 最后回到 `<template>`（**这些状态和方法是怎么显示的**）。
> **不要从 `<template>` 往下读** —— 模板里全是引用 `data`/`methods` 的名字，你还没见过它们，自然看不懂。

---

## 2. Vue 概念速查（`.vue` 里天天见到的词）★核心

这一节把 `.vue` 里反复出现的词集中列出来，每个都用**前端自己的话**说清"它是什么、怎么用"，
再配一个**本项目真实代码**里的用法。

### 2.1 `<script>` 里的字段

```js
export default {
  name: 'Post',              // 组件名
  dicts: ['sys_normal_disable'],  // 声明要用这个字典，见 §6.1
  data() { return { ... } },      // 状态
  computed: { ... },              // 派生值
  watch: { ... },                 // 监听
  created() { ... },              // 生命周期
  mounted() { ... },              // 生命周期
  methods: { ... }                // 方法
}
```

| JS 字段 | 它是什么 | 本项目里的例子 / 注意点 |
|---|---|---|
| `name: 'Post'` | 组件名，调试时在 Vue DevTools 里显示 | 一般和文件名一致 |
| **`data()`** | 返回一个对象，里面的键就是**这个页面的状态**（数据）。**只有在这里声明的字段才是"响应式"的**——改了界面才会变 | `postList` / `queryParams` / `open` 都在这 |
| **`methods`** | 一堆函数，是页面的全部"行为" | 模板里 `@click="handleAdd"` 就是调这里的 `handleAdd` |
| **`computed`** | 由别的数据**算出来**的值，有缓存、依赖变了才重算 | `computed: { values() { return this.value.length } }` |
| `watch` | 盯着某个字段，它一变就执行一段逻辑 | `watch: { open(val) { ... } }`：弹窗开合时想做点别的 |
| **`created()`** | 页面**创建出来、还没画到屏幕上**时执行一次 | 初始化放这，比如 `this.getList()` 拉列表 |
| `mounted()` | 比 `created` 晚，此时 DOM 已经渲染出来 | 要操作真实 DOM（如初始化图表）才用 |
| `props` | **父组件传进来的输入**，见 §7 | 子组件用 `props` 接父组件传的值 |
| `$refs` | 按名字取到模板里的元素/组件 | `this.$refs.form` 拿到 `<el-form ref="form">` |
| `$emit` | 子组件**往上喊一声**通知父组件，见 §7 | `this.$emit('update:page', 2)` |

> ⚠️ **最容易踩的坑**：`data()` 里的字段**必须事先声明**。
> 你写 `this.newField = 1` 但 `newField` 不在 `data()` 里，页面**不会更新**（因为新加的字段不是响应式的）。
> 这是新手最常见的"我改了数据，界面却不动"的原因——**先声明，再使用**。

### 2.2 模板里的符号

| 模板里的写法 | 读作 | 效果 |
|---|---|---|
| `{{ postName }}` | 把变量的值显示在这 | 变量一变，这里跟着变 |
| `v-bind:data="list"` 或简写 `:data="list"` | 把 `list` **传进去**当属性 | **冒号 `:` = 后面是 JS 表达式/变量**，见 §3.2 |
| `v-on:click="fn"` 或简写 `@click="fn"` | 点击时执行 `fn` | **`@` = 绑定事件**，见 §3.7 |
| `v-model="queryParams.name"` | **双向绑定**：输入框的值 ↔ 变量 | 两边自动同步，见 §3.3 |
| `v-if="open"` | `open` 为真才渲染这块 | 为假时**整块不渲染**，DOM 里没有 |
| `v-show="open"` | `open` 为真才**显示**这块 | 为假时**渲染了但用 CSS 藏起来**，DOM 里有 |
| `v-for="d in list"` | 遍历 `list` 生成多个 | 每项生成一个元素，见 §3.5 |
| `v-loading="loading"` | 显示转圈遮罩 | `loading` 为 true 时盖一层遮罩 |

---

## 3. 模板语法：`<template>` 里每行在说什么

下面按"你会在真实页面里遇到的顺序"讲。示例全部取自本项目的真实代码（`views/system/post/index.vue`）。

### 3.1 插值 `{{ }}`：把值印到页面上

```html
<span>{{ parseTime(scope.row.createTime) }}</span>
```

读作：**把 `parseTime(scope.row.createTime)` 的返回值显示在这个 `<span>` 里**。
`parseTime` 不是模板自带的函数 —— 它是挂在全局的一个方法（见 §6.3），作用是**把后端返回的时间字符串**（比如 `2026-10-11T08:00:00`）**格式化成人看的样子**（`2026-10-11 08:00:00`）。

### 3.2 `:prop`（绑定）vs 普通 `prop`（写死）

这是新手第一懵点，**只差一个冒号**：

```html
<el-input placeholder="请输入岗位编码" />        <!-- 纯字符串，写死 -->
<el-input :placeholder="tipText" />              <!-- 取变量 tipText 的值 -->
<el-table :data="postList">                      <!-- 绑数组 -->
```

| 写法 | 含义 |
|---|---|
| `label="岗位名称"` | 直接写字符串 `"岗位名称"` |
| `:label="item.label"` | **冒号 = 里面的东西是 JS 表达式**，取 `item.label` 的值（**双引号只是"表达式的外壳"，不是字符串引号**） |

> 记法：**加冒号 `:` = 里面是 JS 表达式（变量、运算、调用都行）；不加 = 原样一个字符串**。
> 所以 `label="岗位名称"` 显示的就是「岗位名称」四个字；`:label="item.label"` 显示的是 `item.label` 算出来的值。

### 3.3 `v-model`：表单输入的双向绑定

```html
<el-input v-model="queryParams.postCode" placeholder="请输入岗位编码" />
```

读作：**这个输入框的值和 `queryParams.postCode` 绑死了**。
用户打字 → `queryParams.postCode` 自动跟着变；代码里改 `queryParams.postCode` → 输入框自动跟着变。**两个方向都通**，所以叫"双向绑定"。

好处是：你**不用在提交时再去"读"输入框的值** —— 用户一边打字，变量里就已经是那个值了，提交时直接把 `queryParams` 发出去即可。

> ⚠️ **`v-model` 和 `:value` 的区别**（面试常问）：
> - `:value="x"` 是**单向**，只把 `x` 显示到输入框；用户改了输入框，`x` 不变。
> - `v-model="x"` 是**双向**，输入框改了 `x` 也变。
> 表单里几乎永远用 `v-model`，因为你要拿到用户输入的值。

### 3.4 `v-if` / `v-show`：显示和隐藏

```html
<el-form v-show="showSearch" ...>          <!-- 控制搜索栏显隐 -->
<el-button v-if="columns" ...>显隐列</el-button>   <!-- 没列就不渲染这个按钮 -->
```

| | `v-if` | `v-show` |
|---|---|---|
| 怎么实现 | 条件假时**整块不渲染**，DOM 里没有 | 条件假时加 CSS `display:none`，**DOM 里有、只是看不见** |
| 适用 | 很少切换的场景（省渲染开销） | 频繁切换的场景（省重新渲染的开销） |

> 你在本项目会看到 `v-show="showSearch"`：搜索栏老是点开点关，用 `v-show` 更省。

### 3.5 `v-for`：循环，以及那个必须写的 `:key`

```html
<el-option
  v-for="dict in dict.type.sys_normal_disable"
  :key="dict.value"
  :label="dict.label"
  :value="dict.value"
/>
```

读作：**把 `dict.type.sys_normal_disable` 这个数组，每个元素生成一个 `<el-option>`**。

- `v-for="dict in 数组"` —— 遍历数组，每轮把当前元素叫 `dict`（名字随便起），下面就能用它。
- `v-for="(item, index) in 数组"` —— 想拿下标就写成 `(元素, 下标)`。
- **`:key` 必须写**，且要是这行数据的唯一标识（这里是 `dict.value`）。

> 🧠 **`:key` 是什么、为什么必须写**：Vue 靠 `:key` 认"这一行还是上一行那一行"。
> 你不写，Vue 在列表增删时会**按位置猜**，可能把 A 行的状态错安到 B 行上。
> 说人话：`:key` 就是给这一行一个"身份"。列表增删时 Vue 靠它认出"这条还是原来那条"；
> 不写就按位置去猜，可能把某一行的状态错安到另一行头上。

### 3.6 `scope.row`：表格里"当前这一行"从哪来

表格里你会反复看到这个：

```html
<el-table-column label="状态" prop="status">
  <template slot-scope="scope">
    <dict-tag :options="dict.type.sys_normal_disable" :value="scope.row.status" />
  </template>
</el-table-column>
```

- `slot-scope="scope"`：**这个插槽能拿到"当前行"的信息**，叫 `scope`（名字随便起，约定叫 scope）。
- **`scope.row` = 当前这一行的数据对象**（就是 `postList` 数组里的某一个对象）。
- `scope.$index` = 当前行的下标。

所以 `scope.row.status` = **这一行的 `status` 字段**。
> 说人话：`postList` 是一个数组，`scope.row` 就是**当前正在画的那一个元素**。

**为什么有的列不用 `slot-scope`**？看这句：

```html
<el-table-column label="岗位编码" prop="postCode" />
```

只写 `prop="postCode"`，element-ui 就自动把每行的 `postCode` 显示出来 —— 相当于"最简单的一列"。只有当你要**自定义这列长什么样**（加颜色标签、格式化时间、放操作按钮）时，才需要 `<template slot-scope="scope">` 包起来自己写。

### 3.7 修饰符：`.native` / `.sync` / `.enter`

跟在指令后面、用点号连的那一小串，是"修饰符"：

| 修饰符 | 出现在 | 含义 |
|---|---|---|
| `@keyup.enter.native="handleQuery"` | 输入框 | **按回车键**时触发（`.enter` 限键，`.native` 表示监听原生 DOM 事件） |
| `@click.native` | 组件上 | 给**组件**绑原生点击（不加 `.native`，组件可能不把它当点击事件） |
| `:page.sync="queryParams.pageNum"` | 分页组件 | **双向同步**：子组件改了 `page`，父组件这个变量跟着变（见 §7.3） |
| `@click.stop` / `.prevent` | 按钮/链接 | `.stop` 阻止事件向上冒泡；`.prevent` 阻止默认行为（如链接跳转）|

`.native` 为什么需要：`@click` 默认是指"组件自己 `$emit` 出来的 `click` 事件"；而 element-ui 的 `<el-input>` 这种组件不一定往外抛 `click`，所以加 `.native` 说"**我要的是原生 DOM 的点击**"。

### 3.8 `v-hasPermi`：按权限隐藏按钮

```html
<el-button v-hasPermi="['system:post:add']" @click="handleAdd">新增</el-button>
```

这是**本项目自定义的指令**（源码在 `src/directive/permission/hasPermi.js`）：
**当前登录用户的权限码里有 `system:post:add` 就显示这个按钮，没有就把这个元素从 DOM 里删掉。**

- 权限码从哪来？后端 `getInfo` 接口返回的 `permissions` 数组（精读 1 §6 讲过）。
- ⚠️ **它只是"藏按钮"，不是安全措施**。真正拦你的是后端 `@SaCheckPermission`。手改前端或直接 `curl` 接口，照样能调通 —— 所以后端每个接口都要有权限注解。

---

## 4. Element UI 组件速查（`el-` 开头的都是它）

**element-ui 是一套现成的 UI 组件库**（版本 `2.15.6`）。规律很简单：**所有组件都以 `el-` 开头**，你看名字基本能猜出它是干嘛的。

> 🎯 一句话：**element-ui 就是一套现成的界面零件**。
> 你不用手写表格的 `<table>`、不用手写弹窗的定位和遮罩 —— 传几个属性进去，它自己渲染出来。

下面按"一个 CRUD 页会用到什么"分组。**每个都给你：长什么样、关键属性、用在哪**。

### 4.1 表单类

| 组件 | 长什么样 | 关键属性 | 说明 |
|---|---|---|---|
| `<el-form>` | 表单容器 | `:model` 绑数据对象、`:rules` 绑校验规则、`:inline` 行内排列、`ref` 用于取它 | 整个表单的"外壳"，把数据对象和校验规则绑上去 |
| `<el-form-item>` | 表单一"行" | `label` 标签、`prop` 对应 `model` 里的哪个字段 | `prop` 决定这条校验规则用哪条 |
| `<el-input>` | 单行输入框 | `v-model`、`placeholder`、`clearable`（带清空×） | 文本框 |
| `<el-input type="textarea">` | 多行文本域 | 同上 | 备注字段常用 |
| `<el-input-number>` | 数字输入（带步进） | `:min` `:max` | 排序号、金额可用 |
| `<el-select>` + `<el-option>` | 下拉框 | `v-model` 在下拉框上，`el-option` 用 `v-for` 生成选项 | 下拉选择 |
| `<el-radio-group>` + `<el-radio>` | 单选按钮组 | `v-model` 在组上 | 状态这种二选一常用 |
| `<el-date-picker>` | 日期选择 | `v-model`、`type="date/date range"` | 时间范围查询 |
| `<el-checkbox>` / `<el-switch>` | 复选 / 开关 | `v-model` | 布尔值 |

### 4.2 表格类（一个列表页的核心）

```html
<el-table v-loading="loading" :data="postList" @selection-change="handleSelectionChange">
  <el-table-column type="selection" width="55" align="center" />
  <el-table-column label="岗位编码" align="center" prop="postCode" />
  ...
</el-table>
```

| 组件/属性 | 说明 |
|---|---|
| `<el-table :data="postList">` | **表格，数据源是 `postList` 数组**。这是最关键的一句：`postList` 一变，表格自动重画 |
| `v-loading="loading"` | `loading` 为 true 时盖一层转圈遮罩 |
| `@selection-change="fn"` | 勾选行变化时触发 `fn`，`fn` 的参数是**被选中的行数组** |
| `<el-table-column>` | 一列。`label` 列头文字，`prop` 取行对象里的哪个字段 |
| `type="selection"` | 特殊列：勾选框。`type="index"` 是序号列 |
| `align="center"` | 居中对齐 |
| `<template slot-scope="scope">` | 列内容自定义（见 §3.6） |

### 4.3 弹窗 / 分页 / 其它

| 组件 | 说明 |
|---|---|
| `<el-dialog :visible.sync="open">` | **弹窗**。`:visible.sync="open"` 表示"`open` 为 true 就弹出，点×关掉时 `open` 自动变 false" |
| `<div slot="footer">` | dialog 的"底部插槽"，放确定/取消按钮 |
| `<el-row>` / `<el-col :span="n">` | 栅格布局，一行分 24 格，`:span="1.5"` 占 1.5 格。工具按钮那行用它横排 |
| `<el-button>` | 按钮。`type="primary/success/danger"` 决定颜色，`icon="el-icon-plus"` 加图标，`size="mini/small"` 尺寸 |
| `<el-tag>` | 彩色小标签（状态显示常用） |
| `<el-tooltip>` | 鼠标悬停提示 |
| `<el-transfer>` | 左右穿梭框（"显隐列"用的就是它） |

### 4.4 本项目的自研组件（不是 element-ui）

`main.js` 里**全局注册**了这些（任何页面直接用、不用 import），源码在 `src/components/`：

| 组件 | 作用 |
|---|---|
| `<pagination>` | 分页条（见 §7.1 逐行） |
| `<right-toolbar>` | 表格右上角"刷新/显隐搜索/显隐列"三个圆按钮（见 §7.2） |
| `<dict-tag>` | 把字典值渲染成带颜色的标签 |
| `<file-upload-contract>` / `<image-upload>` 等 | 文件/图片上传 |
| `<money-input>` | 金额输入 |
| `<zn-modal>` / `<zn-info-page>` | 业务弹窗 / 信息页（`Zn` 前缀 = 本项目自研） |

> 💡 **看到 `Zn` 开头就知道是二次开发**，不是 RuoYi 框架自带的。

---

## 5. 逐行拆一个真实页面 ★核心

光看语法表还是虚的。我们**拿一个最典型的 CRUD 页面从头读到尾**：
`zxcm-ui/src/views/system/post/index.vue`（311 行）。

> ⚠️ **先讲清一件事**：这个文件在**本项目里没有被任何菜单挂载**（`sys_menu` 里查 `COMPONENT LIKE '%post%'` 是 0 条）。
> 它是 RuoYi 框架自带的"岗位管理"页；本项目左侧「**岗位管理**」菜单实际指向的是 `views/system/position/index.vue`（风格偏定制，见 §5.7）。
> **但它仍是最好的学习范本** —— 它是 RuoYi 最标准的"**搜索栏 + 表格 + 弹窗表单**"CRUD 模板。
> 后端第 1~5 章的签约主体 / 合作方 / 编号规则 / 合同草拟，**套路和它完全一样**，只是字段更多、多几个特殊区域。
>
> 所以：**先吃透这一个（最干净），再去看真实挂载的页面（更花哨）** —— 你会发现自己一眼就能找到 `data`、`methods`、`el-table` 在哪。

先记住这个页面的**全貌地图**（读任何 `views/**/index.vue` 都先找这五块）：

```
┌─────────────────────────────────────────────┐
│ ① 搜索表单（v-show="showSearch"）             │  ← 输入条件 → 点搜索 → getList()
├─────────────────────────────────────────────┤
│ ② 工具按钮行（新增/修改/删除/导出 + 小工具）    │  ← v-hasPermi 控制显隐
├─────────────────────────────────────────────┤
│ ③ 表格 el-table :data="postList"             │  ← 数据源，最重要的一句
│    勾选 | 岗位编号 | 编码 | 名称 | 状态 | 操作  │
├─────────────────────────────────────────────┤
│ ④ 分页 <pagination>                          │  ← 翻页 → getList()
└─────────────────────────────────────────────┘
        ↑ 点"新增/修改"弹出 ↓
┌─────────────────────────────────────────────┐
│ ⑤ 弹窗 el-dialog（表单 + 校验 + 确定/取消）    │  ← 确定 → addPost/updatePost
└─────────────────────────────────────────────┘
```

### 5.1 ① 搜索表单

```html
<el-form v-show="showSearch" ref="queryForm" :model="queryParams" :inline="true" label-width="68px">
  <el-form-item label="岗位编码" prop="postCode">
    <el-input
      v-model="queryParams.postCode"
      placeholder="请输入岗位编码"
      clearable
      size="small"
      @keyup.enter.native="handleQuery"
    />
  </el-form-item>
  ...
  <el-form-item>
    <el-button type="primary" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
    <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
  </el-form-item>
</el-form>
```

逐行读：

| 代码 | 说什么 |
|---|---|
| `v-show="showSearch"` | 右上角小工具能把这整块搜索栏藏起来 |
| `ref="queryForm"` | 给这个表单起个名，`this.$refs.queryForm` 能拿到它（重置时要用） |
| `:model="queryParams"` | **这个表单的数据对象是 `queryParams`** —— 输入框的值都存它里面 |
| `:inline="true"` | 表单项横着排一行 |
| `label-width="68px"` | 标签宽度，为了对齐 |
| `v-model="queryParams.postCode"` | 输入框 ↔ `queryParams.postCode` 双向绑定 |
| `@keyup.enter.native="handleQuery"` | 按回车也触发搜索 |
| `@click="handleQuery"` | 点"搜索"调 `handleQuery`（见下） |

> ➡️ **这个 `queryParams` 最后会原样发给后端**，后端有一个对应的**查询入参对象**（`PostBo`/`QueryBo`）来接它。
> 所以表单里 `prop="postCode"` 的字段名，要和后端那个对象的字段名**对上**，后端才收得到。
> 你这边不用手写"从请求里取参数"——`v-model` 已经把用户输入填进 `queryParams` 了。

### 5.2 ② 工具按钮行

```html
<el-row :gutter="10" class="mb8">
  <el-col :span="1.5">
    <el-button
      v-hasPermi="['system:post:add']"
      type="primary" plain icon="el-icon-plus" size="mini"
      @click="handleAdd"
    >新增</el-button>
  </el-col>
  <el-col :span="1.5">
    <el-button
      v-hasPermi="['system:post:edit']"
      type="success" plain icon="el-icon-edit" size="mini"
      :disabled="single"
      @click="handleUpdate"
    >修改</el-button>
  </el-col>
  ...
  <right-toolbar :show-search.sync="showSearch" @queryTable="getList" />
</el-row>
```

要点：

- `v-hasPermi="['system:post:add']"`：**没这个权限码，这个按钮直接不存在**。
- **`:disabled="single"`**：`single` 是 data 里那个布尔字段。它的含义是「**当前选中行不等于 1 行**」。
  所以按钮的禁用状态 = `single` 的值 —— **选中恰好 1 行时 `single` 变 false，修改按钮才可用**（见 §5.6 的 `handleSelectionChange`）。
  这就是"必须选中一行才能点修改"是怎么实现的。
- **`<right-toolbar>`**：本项目全局组件，右上角三个圆按钮。`:show-search.sync="showSearch"` 让子组件能反过来改父组件的 `showSearch`（§7.3 讲 `.sync`）；`@queryTable="getList"` 表示子组件点"刷新"时调父组件的 `getList`。

### 5.3 ③ 表格

```html
<el-table v-loading="loading" :data="postList" @selection-change="handleSelectionChange">
  <el-table-column type="selection" width="55" align="center" />
  <el-table-column label="岗位编号" align="center" prop="postId" />
  <el-table-column label="岗位编码" align="center" prop="postCode" />
  <el-table-column label="岗位名称" align="center" prop="postName" />
  <el-table-column label="岗位排序" align="center" prop="postSort" />

  <el-table-column label="状态" align="center" prop="status">
    <template slot-scope="scope">
      <dict-tag :options="dict.type.sys_normal_disable" :value="scope.row.status" />
    </template>
  </el-table-column>

  <el-table-column label="创建时间" align="center" prop="createTime" width="180">
    <template slot-scope="scope">
      <span>{{ parseTime(scope.row.createTime) }}</span>
    </template>
  </el-table-column>

  <el-table-column label="操作" align="center" class-name="small-padding fixed-width">
    <template slot-scope="scope">
      <el-button v-hasPermi="['system:post:edit']" size="mini" type="text"
        icon="el-icon-edit" @click="handleUpdate(scope.row)">修改</el-button>
      <el-button v-hasPermi="['system:post:remove']" size="mini" type="text"
        icon="el-icon-delete" @click="handleDelete(scope.row)">删除</el-button>
    </template>
  </el-table-column>
</el-table>
```

三种列，按复杂度递增：

1. **纯 prop 列**：`prop="postCode"` —— 直接把每行的 `postCode` 显示出来。
2. **`slot-scope` 自定义列**：状态列用 `<dict-tag>` 把 `'0'` 渲染成带颜色的"正常"标签；时间列用 `parseTime()` 格式化。
3. **操作列**：`@click="handleUpdate(scope.row)"` —— **注意括号里传了 `scope.row`**，意思是"修改**这一行**"。

> 🔑 **操作列传不传 `scope.row` 是理解本页面的关键**：
> - 表格里的"修改"传了 `scope.row` → 改的是**这一行**。
> - 上方工具条的"修改"没传 → `handleUpdate()` 里 `row` 是 `undefined`，于是去取 `this.ids`（勾选的多行）→ 改的是**勾中的那行**。
> 同一个方法，两种入口。看懂这层，你就看懂了 `handleUpdate(row)` 里那句 `row.postId || this.ids`。

### 5.4 ④ 分页

```html
<pagination
  v-show="total>0"
  :total="total"
  :page.sync="queryParams.pageNum"
  :limit.sync="queryParams.pageSize"
  @pagination="getList"
/>
```

- `v-show="total>0"`：没数据就不显示分页条。
- `:total="total"`：总条数（来自后端返回的 `total`）。
- `:page.sync="queryParams.pageNum"` / `:limit.sync="queryParams.pageSize"`：**当前页码和每页条数，和 `queryParams` 双向同步**。
- `@pagination="getList"`：**用户翻页时，子组件喊一声 `pagination`，父组件重新拉数据**。

这个组件内部怎么写的，§7.1 逐行拆。

### 5.5 ⑤ 弹窗（新增/修改共用）

```html
<el-dialog :title="title" :visible.sync="open" width="500px" append-to-body>
  <el-form ref="form" :model="form" :rules="rules" label-width="80px">
    <el-form-item label="岗位名称" prop="postName">
      <el-input v-model="form.postName" placeholder="请输入岗位名称" />
    </el-form-item>
    <el-form-item label="岗位编码" prop="postCode">
      <el-input v-model="form.postCode" placeholder="请输入编码名称" />
    </el-form-item>
    <el-form-item label="岗位顺序" prop="postSort">
      <el-input-number v-model="form.postSort" controls-position="right" :min="0" />
    </el-form-item>
    <el-form-item label="岗位状态" prop="status">
      <el-radio-group v-model="form.status">
        <el-radio v-for="dict in dict.type.sys_normal_disable" :key="dict.value" :label="dict.value">
          {{ dict.label }}
        </el-radio>
      </el-radio-group>
    </el-form-item>
    <el-form-item label="备注" prop="remark">
      <el-input v-model="form.remark" type="textarea" placeholder="请输入内容" />
    </el-form-item>
  </el-form>
  <div slot="footer" class="dialog-footer">
    <el-button type="primary" @click="submitForm">确 定</el-button>
    <el-button @click="cancel">取 消</el-button>
  </div>
</el-dialog>
```

要点：

- `:title="title"`：标题是变量 —— 新增时是"添加岗位"，修改时是"修改岗位"（见 `handleAdd`/`handleUpdate`）。
- `:visible.sync="open"`：**`open` 为 true 就弹出**。点弹窗的 × 或"取消"时，`open` 自动变 false，弹窗自己关。**新增和修改共用这一个弹窗**，靠 `title` 和 `form` 区分。
- `:rules="rules"`：表单校验规则（在 `data` 里，见下）。
- `slot="footer"`：把这两个按钮塞进弹窗底部区域。
- `v-for` 那一行：**状态选项也是从字典拉的**（`sys_normal_disable`），不是写死的。

### 5.6 ⑥ `<script>` 部分：这个页面的"状态 + 方法"

```js
import { listPost, getPost, delPost, addPost, updatePost } from '@/api/system/post'

export default {
  name: 'Post',
  dicts: ['sys_normal_disable'],
  data() {
    return {
      loading: true,          // 遮罩层
      ids: [],                // 勾选的行 id 数组
      single: true,           // "选中行 ≠ 1" → 修改按钮禁用
      multiple: true,         // "选中行 = 0" → 删除按钮禁用
      showSearch: true,       // 是否显示搜索栏
      total: 0,               // 总条数
      postList: [],           // ★ 表格数据源
      title: '',              // 弹窗标题
      open: false,            // 弹窗是否显示
      queryParams: {          // ★ 查询参数（上送后端的 DTO）
        pageNum: 1,
        pageSize: 10,
        postCode: undefined,
        postName: undefined,
        status: undefined
      },
      form: {},               // ★ 弹窗表单数据
      rules: {                // 表单校验规则
        postName: [{ required: true, message: '岗位名称不能为空', trigger: 'blur' }],
        postCode: [{ required: true, message: '岗位编码不能为空', trigger: 'blur' }],
        postSort: [{ required: true, message: '岗位顺序不能为空', trigger: 'blur' }]
      }
    }
  },
  created() {
    this.getList()            // 页面一创建就拉列表
  },
  methods: { ... }
}
```

**先看 `data`（状态清单）—— 这 6 个字段是每个 CRUD 页的"标配"**：

| 字段 | 作用 | 后端对应（对照用）|
|---|---|---|
| `postList` | 表格数据源 | 列表查询返回的 `List<PostVo>` |
| `queryParams` | 查询条件 + 分页 | `PostQueryBo`（含 `pageNum/pageSize`） |
| `form` | 弹窗表单 | `PostBo`（新增/修改的入参） |
| `open` / `title` | 弹窗显隐 / 标题 | 界面上自己的状态，后端没有 |
| `total` | 总条数 | `Page.getTotal()` |
| `ids` / `single` / `multiple` | 勾选状态 | 界面上自己的状态，后端没有 |
| `loading` / `showSearch` | 遮罩 / 搜索栏显隐 | 界面上自己的状态，后端没有 |

**再看 `created()`**：整个页面只有一句 `this.getList()`。
> 说人话：**页面一被创建出来，`created()` 就自动跑一次**。所以这里写 `this.getList()`，
> 等于"每次打开这个页面，都自动拉一遍列表"——你不需要在任何地方手动触发它。

**`methods` 逐个读**（这是页面的全部"行为"）：

```js
/** 查询岗位列表 —— 最核心的方法，被搜索/重置/翻页/增删改后反复调用 */
getList() {
  this.loading = true                        // 打开遮罩
  listPost(this.queryParams).then(response => {   // ← 调后端接口（api/system/post.js）
    this.postList = response.rows            // 把返回的行填进表格数据源
    this.total = response.total              // 填总条数
    this.loading = false                     // 关遮罩
  })
},
```

> 🧠 **`listPost(...).then(response => {...})` 是什么**：`listPost` 发了个 HTTP 请求，**它不会立刻返回结果**（网络有延迟），
> 所以返回的是一个"**承诺**"（`Promise`）。`.then(回调)` 的意思是"**等结果回来了，执行这个回调**"——它是异步的，不是同步拿到结果。
>
> `response` 就是后端返的 `R<TableDataInfo>`：`response.rows` = 数据行，`response.total` = 总数。
> **注意这里没有 `if (code === 200)` 判断** —— 因为 `utils/request.js` 的响应拦截器已经统一判过了（精读 1 §约定3）。

```js
// 取消按钮：关弹窗 + 清表单
cancel() { this.open = false; this.reset() },

// 表单重置：把 form 清成初始值
reset() {
  this.form = { postId: undefined, postCode: undefined, postName: undefined,
                postSort: 0, status: '0', remark: undefined }
  this.resetForm('form')      // ← 全局方法，清掉校验的红字提示
},

// 搜索：页码归 1，再拉列表
handleQuery() { this.queryParams.pageNum = 1; this.getList() },

// 重置搜索：清空查询表单 + 重新搜
resetQuery() { this.resetForm('queryForm'); this.handleQuery() },

// 勾选变化：更新 ids / single / multiple
handleSelectionChange(selection) {
  this.ids = selection.map(item => item.postId)   // 取每行的 postId
  this.single = selection.length != 1             // 不是 1 行 → 修改禁用
  this.multiple = !selection.length               // 0 行 → 删除禁用
},

// 新增：清表单 → 开弹窗 → 标题"添加岗位"
handleAdd() { this.reset(); this.open = true; this.title = '添加岗位' },

// 修改：清表单 → 拿 id → 查详情 → 回填 form → 开弹窗
handleUpdate(row) {
  this.reset()
  const postId = row.postId || this.ids     // ★ 表格里点了传 row；工具条点了取勾选的 ids
  getPost(postId).then(response => {        // 先查详情
    this.form = response.data               // 回填表单
    this.open = true
    this.title = '修改岗位'
  })
},

// 提交：先校验 → 有 id 走修改、没 id 走新增
submitForm: function() {
  this.$refs['form'].validate(valid => {    // ← 触发 el-form 的校验
    if (valid) {                            // 校验通过才提交
      if (this.form.postId != undefined) {
        updatePost(this.form).then(response => {
          this.$modal.msgSuccess('修改成功')   // 绿条提示
          this.open = false                   // 关弹窗
          this.getList()                      // 重新拉列表
        })
      } else {
        addPost(this.form).then(response => { ... })
      }
    }
  })
},

// 删除：先弹确认框 → 确认后调删除接口
handleDelete(row) {
  const postIds = row.postId || this.ids
  this.$modal.confirm('是否确认删除岗位编号为"' + postIds + '"的数据项？').then(function() {
    return delPost(postIds)                  // 点"确定"才发请求
  }).then(() => { this.getList(); this.$modal.msgSuccess('删除成功') })
    .catch(() => {})                         // 点"取消"走这，什么都不做
},

// 导出：直接下载文件
handleExport() {
  this.download('system/post/export', { ...this.queryParams }, `post_${new Date().getTime()}.xlsx`)
}
```

> 🧠 **`this.$refs['form'].validate(valid => {...})`**：
> `<el-form>` 组件自带校验能力。你调它的 `validate`，它检查所有 `prop` + `rules`，
> 通过就回调 `valid=true`，不通过就 `valid=false`，并把红字显示到对应输入框下面。
> 它发生在**你点"确定"的那一刻**：不通过就直接拦住，请求根本不发出去。
>
> `$refs['form']` 对应模板里 `<el-form ref="form">` —— **按 `ref` 名字取到这个组件对象**，才能调它的 `validate` 方法。

> 💡 **新增/修改共用一个 `submitForm`** 的写法：靠 `form.postId` 有没有值来判断。
> 逻辑是"两段式"：**先 `validate` 校验通过，再决定是新增还是修改**。

### 5.7 对照：真正挂载的「岗位管理」页 `views/system/position/index.vue`

上面那个是"教学版"，下面这个是本项目**左侧菜单「岗位管理」实际打开的页面**（`sys_menu.COMPONENT = system/position/index`）。
同一个 CRUD 思路，但有四处变体 —— 认出来你就不慌：

| 变体 | 表现 | 说明 |
|---|---|---|
| **左树右表** | 左边一棵 `el-tree` 组织机构，右边表格 | 点树节点 → `handleNodeClick` → 带上 `organizeId` 重新查列表。比 §5 多了"按部门过滤" |
| **表格换成 `<JTable>`** | `<JTable v-loading="listLoading" :data="tableData">` | 本项目自研组件，包了一层 `el-table`（多了列显隐等能力），用起来一样 |
| **弹窗抽成子组件** | `<Form v-if="formVisible" ref="Form" @refreshDataList="initData" />` | 表单没内联，而是单独一个 `Form.vue`。父组件 `this.$refs.Form.init(id, organizeId)` 打开它 |
| **确认框直接用 element-ui** | `this.$confirm('确认要删除当前岗位么？', ...)` | §5 用的是本项目封装的 `this.$modal.confirm()`，这里直接用 element-ui 原生的 `$confirm`。**两者功能等价** |

**同一个接口套路照旧**：`created` → `getTreeselect()` → 拿到树后再 `initData()` 拉列表；
`@pagination="initData"` 翻页；`handleAddEdit(id)` 打开表单；`handleDel(id)` 先确认再删。

> 🎯 **练法**：拿这一节当"考试" —— 打开 `views/system/position/index.vue`，自己找：① `data` 里哪些是表格数据源？② `initData` 调了哪个 api？③ 翻页时谁被触发？
> 三个都能答上来，说明 §5 你真读懂了。下一页进入 `views/conf/signingSubject/index.vue`（对应后端第 1 章的签约主体），只是字段更多而已。

---

## 6. 三类"代码里看不见"的魔法

下面这些你在页面里**用了，但找不到定义** —— 它们由框架在启动时挂上去的。不知道来源会看得很懵，这里一次说清。

### 6.1 `dicts: [...]` 和 `dict.type.xxx`：字典

页面 `data` 外面写了一行：

```js
export default {
  dicts: ['sys_normal_disable'],   // ← 声明"我要用这个字典"
  data() { ... },
}
```

然后模板里就能写：

```html
<el-option v-for="dict in dict.type.sys_normal_disable" :key="dict.value" :label="dict.label" :value="dict.value" />
```

**`dict` 和 `dict.type.sys_normal_disable` 是哪来的？** 没有任何地方 import 它。答案是**一个全局 mixin**：

`src/utils/dict/index.js` 里注册了一个 `Vue.mixin`，逻辑是：

```
任何组件创建时：
  如果这个组件声明了 dicts: [...]           ← 就像 post 页写了 dicts: ['sys_normal_disable']
     → 给它注入一个 data 字段叫 dict
     → 自动调后端 getDicts('sys_normal_disable') 把选项拉回来
     → 存进 dict.type.sys_normal_disable
```

所以 **`dict.type.sys_normal_disable` = 后端数据字典 `sys_normal_disable` 的所有选项**（`{label:'正常', value:'0'}, {label:'停用', value:'1'}`）。

> ➡️ **这些选项来自后端的"数据字典"**（`sys_dict_type` + `sys_dict_data` 两张表）。
> 前端**不把选项写死**，而是启动时按 `dicts` 声明的类型去后端拉 —— 后端改了字典，前端刷新就对。
> **你只要记住：想在模板里用 `dict.type.xxx`，就先把 `xxx` 写进 `dicts: [...]`。**

### 6.2 `this.$modal` / `this.$tab` / `this.$auth` / `this.$download`：全局工具对象

页面里到处是 `this.$modal.msgSuccess(...)`、`this.$modal.confirm(...)`、`this.download(...)`，
但 `$modal` 也不在任何页面里定义。它在 `src/plugins/index.js` 里被挂到 `Vue.prototype` 上：

```js
install(Vue) {
  Vue.prototype.$modal     = modal       // 弹窗类
  Vue.prototype.$tab       = tab         // 标签页操作
  Vue.prototype.$auth      = auth        // 权限判断
  Vue.prototype.$cache     = cache       // 缓存
  Vue.prototype.$download  = download    // 下载
}
```

`$modal` 的方法（源码 `src/plugins/modal.js`）：

| 方法 | 效果 |
|---|---|
| `$modal.msgSuccess('新增成功')` | 顶部绿色条 |
| `$modal.msgError('失败了')` | 顶部红色条 |
| `$modal.confirm('确定删除？')` | 弹"确定/取消"框，**返回 Promise**（所以能 `.then()`） |
| `$modal.alert(...)` | 弹一个只有"确定"的提示框 |
| `$modal.loading('处理中')` / `closeLoading()` | 全屏遮罩 |

> 💡 **这就是"全局工具"的做法**：把 `modal` 挂到 `Vue.prototype` 上，于是**所有页面**都能用 `this.$modal` 拿到它
> ——不用每个页面各自 import 一遍。**凡是挂到 `Vue.prototype` 上的东西，页面里 `this.$xxx` 就能直接用。**

### 6.3 挂在 `Vue.prototype` 上的全局方法

`main.js:72-90` 一口气挂了一堆，页面里直接 `this.xxx(...)` 用：

| 方法 | 作用 | 你在哪见过 |
|---|---|---|
| `this.resetForm('form')` | 清掉表单校验的红字 | `reset()`、`resetQuery()` |
| `this.parseTime(str)` | 时间格式化 | 时间列 `{{ parseTime(scope.row.createTime) }}` |
| `this.handleTree(list, 'id')` | 平铺列表 → 树结构 | 部门树、菜单树 |
| `this.addDateRange(...)` | 组装时间范围查询参数 | 带日期范围的搜索 |
| `this.selectDictLabel(...)` | 按字典值取标签文字 | 导出/详情显示 |
| `this.download(url, params, filename)` | 下载文件 | 导出按钮 |
| `this.moneyFormat(...)` | 金额格式化 | 合同金额 |
| `this.getDicts(type)` | 拉字典 | 少数手动拉字典的地方 |

> 这些方法的定义在 `src/utils/ruoyi.js` 等文件里，但**页面里看不到 import**，因为 `main.js` 统一 `Vue.prototype.xxx = ...` 挂上去了。
> **记法：凡是 `this.$xxx` / `this.xxx()` 但本文件没定义的，去 `main.js` 和 `plugins/` 找。**

---

## 7. 组件是怎么复用的：props / `$emit` / `.sync`

页面里那些 `<pagination>`、`<right-toolbar>` 是"组件"—— **可复用的页面片段**。
组件之间靠两个东西通信，理解它俩，你就理解了 `:page.sync` 和 `@queryTable` 到底在干嘛。

**核心两句话**：

```
父 → 子：通过 props（属性）往下传数据          ← 父把值"喂"给子
子 → 父：通过 $emit（触发事件）往上喊           ← 子有事"通知"父
```

### 7.1 逐行拆 `<pagination>`（父传子 + 子传父 + `.sync`）

先看**子组件自己怎么写的**（`src/components/Pagination/index.vue`）：

```js
export default {
  name: 'Pagination',
  props: {                         // ★ 父组件能传进来的"参数"
    total: { required: true, type: Number },   // 必填：总条数
    page:  { type: Number, default: 1 },       // 当前页
    limit: { type: Number, default: 20 },      // 每页条数
    pageSizes: { type: Array, default() { return [10,20,30,50] } },
    layout: { type: String, default: 'total, sizes, prev, pager, next, jumper' },
    // ...
  },
  computed: {
    currentPage: {                 // ★ 一个"可读可写"的派生属性
      get() { return this.page },              // 读 → 返回父传进来的 page
      set(val) { this.$emit('update:page', val) }  // 写 → 通知父组件改 page
    },
    pageSize: {
      get() { return this.limit },
      set(val) { this.$emit('update:limit', val) }
    }
  },
  methods: {
    handleCurrentChange(val) {
      this.$emit('pagination', { page: val, limit: this.pageSize })  // ★ 往外喊一声
      scrollTo(0, 800)
    }
  }
}
```

再看**父页面怎么用的**（回到 `post/index.vue`）：

```html
<pagination
  :total="total"                        ← 父把 total 传进去（props）
  :page.sync="queryParams.pageNum"      ← 双向：子能改父的 pageNum
  :limit.sync="queryParams.pageSize"
  @pagination="getList"                 ← 子喊 pagination 时，父执行 getList
/>
```

**把两边对起来看**：

| 父组件写的 | 子组件里对应 | 发生什么 |
|---|---|---|
| `:total="total"` | `props.total` | 父的总条数传进子，子用来算"共几页" |
| `:page.sync="queryParams.pageNum"` | `props.page` + `$emit('update:page', val)` | 用户翻到第 2 页 → 子 `emit('update:page', 2)` → **父的 `queryParams.pageNum` 自动变 2** |
| `@pagination="getList"` | `$emit('pagination', {...})` | 子喊一声 → **父执行 `getList()` 重新拉数据** |

> 🔑 **`.sync` 是语法糖**：`:page.sync="X"` 等价于
> `:page="X"` **加上** `@update:page="X = $event"`。
> 也就是："把 `X` 传给你（`:page`），你要是改了它，就把它改回来（`@update:page`）"。
> **子组件 `$emit('update:page', 新值)` 的那一刻，父组件的 `queryParams.pageNum` 就被改成新值了。**
> 这就是为什么你会看到子组件里写 `this.currentPage = 1`（看着像改自己的字段，其实是往父组件发了个"更新"信号）。

**完整翻页时序**：

```
用户点"第 2 页"
   → 子组件 handleCurrentChange(2)
       → this.$emit('pagination', {page:2, limit:10})     [通知父：我翻页了]
       → computed setter 触发 $emit('update:page', 2)      [同步父的 pageNum]
   → 父组件 @pagination 监听器执行 getList()
       → 此时 queryParams.pageNum 已经是 2 了
       → listPost(queryParams) 发出第 2 页的请求
```

### 7.2 拆 `<right-toolbar>`（拆解"刷新/显隐列"）

子组件（`src/components/RightToolbar/index.vue`）：

```js
export default {
  props: {
    showSearch: { type: Boolean, default: true },
    columns: { type: Array }
  },
  methods: {
    toggleSearch() { this.$emit('update:showSearch', !this.showSearch) },  // 切换显隐
    refresh()      { this.$emit('queryTable') }                           // 喊"刷新"
  }
}
```

父组件用：

```html
<right-toolbar :show-search.sync="showSearch" @queryTable="getList" />
```

- 点搜索图标 → 子 `$emit('update:showSearch', false)` → **父的 `showSearch` 变 false** → `v-show="showSearch"` 的搜索栏收起。
- 点刷新图标 → 子 `$emit('queryTable')` → **父执行 `getList()`** → 列表重新加载。

> 💡 **`@queryTable` 而不是 `@click`**：这是**自定义事件名**。子组件说"我这边发生了 `queryTable` 这件事"，
> 父组件说"`queryTable` 发生时我执行 `getList`"。**事件名靠父子两边自己约定，写错就接不上。**

### 7.3 props 的"单向"铁律

⚠️ **子组件不能直接改 `props`**（改了会警告）。**只能 `$emit` 通知父组件去改**。
这就是"数据从谁那来、谁有权改"的约定：**子组件只收 `props`、不直接改**，
要改就 `$emit` 通知父组件去改（父组件才是这份数据的"主人"）。

`.sync` 就是给这种"我想改你的变量"提供一个**合法通道**：不是直接改，是 `$emit('update:xxx')` 请父组件改。

---

## 8. 从点按钮到后端：完整链路（→ 见 精读 4）

> ⚠️ **这一节已「归口」到 [精读 4 · 一次点击的完整旅程](04-一次点击的完整旅程.md)**。
> 那条链在 精读 4 里是**逐跳贴真实代码**的（点击 → `api` → `request.js` 加 token → 代理删前缀 → 后端 Controller → Service → Mapper → 返回 → 页面刷新），比本文详细得多。
> **本文是语法字典，不再重复画它**，只留一句最该记住的：

```
@click="submitForm"   →   methods.submitForm()   →   api/xxx.js 的 request({url, method, data})
        →   utils/request.js（统一加 token + 转发）   →   后端 Controller → Service → Mapper → MySQL
```

**三个结论**（展开与代码在 精读 4）：

1. **`api/*.js` 只是「请求声明」** —— 它只写"发到哪个 URL、用什么方法"，真正的处理在后端。
2. **token 不用你手写、`code !== 200` 不用每页判** —— 都被 `utils/request.js` 的两个拦截器统一处理了。
3. **增删改之后要手动 `this.getList()`** —— 前端不会"自动同步数据库最新数据"，得你主动重拉。

---

## 9. ES6 语法小抄（读 `.vue` 最容易卡住的几句）

| 写法 | 读作 | 它做什么 / 注意点 |
|---|---|---|
| `import { addPost } from '@/api/system/post'` | 从 `@/api/system/post.js` 里取出 `addPost` 拿来用 | 用别的文件里的东西，先得 `import` |
| `export function addPost() {}` / `export default {...}` | 把东西暴露出去，别人才能 `import` | 一个文件里 `export` 了啥，别处才能取啥 |
| `const x = 1` / `let y = 2` | 常量 / 变量 | `const` 不能重新赋值；前端写 `let`，几乎不用老的 `var` |
| `(a, b) => a + b` | 箭头函数 | 一种简写的匿名函数；**回调里要访问组件数据就用它**（见下面 `this` 的坑） |
| `.then(r => {...})` | 异步回调 | "等请求结果回来，再执行这段" |
| `async` / `await` | 用同步写法写异步 | `await` 让 `Promise` 写起来像一句一句往下走 |
| `` `post_${new Date().getTime()}.xlsx` `` | 模板字符串 | 用反引号 `` ` `` 包起来，`${}` 里能塞变量拼进字符串 |
| `const { rows, total } = response` | 解构赋值 | 从对象里一次取出 `rows`、`total` 两个字段 |
| `{ ...this.queryParams }` | 展开 / 浅拷贝 | 复制出一份新对象，改它不影响原来那份 |
| `this.ids = selection.map(item => item.postId)` | 映射 | 把数组每项换成它的 `postId`，得到一个新数组 |
| `arr.includes(x)` | 判断 | 数组里有没有 `x` |
| `a ?? b` / `a?.b` | 空值处理 | `a` 为空就用 `b`；`a?.b` 在 `a` 为空时不报错、直接 undefined |
| `this` | 当前组件实例 | 指向"当前这个页面对象"；但普通 `function(){}` 回调里的 `this` 可能不是组件，**新手常被这个坑** |

> ⚠️ **`this` 的坑**：`methods` 里的普通 `function` 写法，`this` 指向组件；
> 但你写回调时如果用了 `function() {}`（不是箭头函数），里面的 `this` 可能**不再是组件**。
> 看 `handleDelete` 里：`.then(function() { return delPost(postIds) })` —— 它没用到 `this` 所以没事；
> 后面 `.then(() => { this.getList() })` 用了**箭头函数**，才敢用 `this`。
> **记法：回调里要访问组件数据，就用箭头函数 `() => {}`。**

---

## 10. 调试与排错

### 10.1 三个必会手段

| 手段 | 怎么用 | 看什么 |
|---|---|---|
| **`console.log`** | 在 `<script>` 里写 `console.log('form=', this.form)` | F12 的 Console 面板里会把内容打印出来 |
| **Network 面板** | F12 → Network → 点按钮触发请求 | URL、请求参数、响应体、状态码。**接口问题全在这查** |
| **Vue DevTools** | 浏览器装插件 | 看组件的 `data` 实时值，比 `console.log` 方便 |

### 10.2 按现象定位

| 现象 | 先看哪 |
|---|---|
| 点按钮没反应 | 按钮的 `@click` 方法名写对没？`methods` 里有这个方法吗？`v-hasPermi` 是不是把它藏了？ |
| 输入框能打字，但提交时是空的 | `v-model` 绑的字段和提交用的字段是同一个吗？ |
| 列表有数据但页面不显示 | `:data` 绑的数组名对没对？`response.rows` 取对没对？ |
| 数据改了界面不刷新 | 这个字段在 `data()` 里声明了吗？（§2.1 的坑） |
| `undefined` / `Cannot read property 'x' of undefined` | 有个变量是空的。往往是对手数据还没回来就用了它 |
| 接口 404 | 后端没这个路径；或前端 `url` 拼错了（和后端 Controller 的 `@RequestMapping` 对一下） |
| 接口 401 | token 过期，正常重登；或 `request.js` 没带上 token |
| 参数后端收不到 | 名字对不上：`v-model`/`queryParams` 的键 = 后端 `Bo` 的字段名 |
| 样式乱了 | 跟你无关，忽略 |

### 10.3 一个实用的读码技巧

看到页面里某个方法不知道干嘛，**直接在 `src/api/` 全局搜它的 url 片段**。
比如不知道 `listPost` 调了啥，搜 `/system/post/list`，能一路找到后端 `SysPostController`。

### 10.4 环境 / 工程类坑 → 见 精读 1 §12

> 上面 10.2 讲的是「代码写错了」。**「环境没配好」那一类坑**（`node` 找不到、端口 80 被占、后端没起导致接口转圈、Vue 3 语法不生效……）**已集中归口到 [精读 1 · §12 踩坑速查（前端）](01-工程结构速查.md#12-踩坑速查前端)**，本文不重复。
> 转发的完整机制（`/dev-api` 怎么转给 8080）见 [精读 4 · 中篇「代理转发」](04-一次点击的完整旅程.md)。

---

## 11. 速查卡

#### 我要做 X → 看哪

| 我想…… | 前端在哪 | 后端在哪 |
|---|---|---|
| 找某个菜单对应的页面 | `sys_menu.COMPONENT` 的值 → `src/views/<值>.vue` | — |
| 看这个页面有哪些"状态" | `<script>` 的 `data()` | 后端接它的入参/返回对象 |
| 看这个页面能做哪些操作 | `<script>` 的 `methods` | 后端对应的接口方法 |
| 看某个按钮调了什么接口 | `@click="fn"` → `methods.fn` → `import` 的 api 函数 | Controller 对应方法 |
| 看接口的 URL / 方法 | `src/api/**.js` 里的 `request({url, method})` | `@GetMapping/@PostMapping` |
| 看界面长什么样 | `<template>` | — |
| 看某个字段怎么显示的 | 搜字段名，看它在哪个 `el-table-column` | — |
| 找到"魔法"方法的定义 | `main.js` 的 `Vue.prototype.xxx` / `plugins/` | — |

#### 一个 `.vue` 文件的阅读顺序（背下来）

```
1. <script> 顶部 import      → 这个页面依赖哪些接口/工具
2. <script> data()           → 页面有哪些状态（★ 重点）
3. <script> methods          → 页面有哪些行为（★ 重点）
4. <script> created/mounted  → 页面打开时先干什么
5. <template>                → 上面这些状态和行为是怎么显示的
6. <style>                   → 忽略
```

#### 三句话总结

1. **一个 `.vue` = 一个页面**：`<template>` 是长相、`<script>` 是状态+方法、`<style>` 是样式。
2. **`data` 存状态、`methods` 写行为、`created` 是页面创建时自动跑一次、`props` 接父组件传的值、`$emit` 往父组件喊话** —— 记住这五个，一个页面就能读通。
3. **`this.xxx` 没定义 = 去 `main.js` / `plugins/` 找；模板里不认识的组件 = 去 `src/components/` 找或查 element-ui 文档。**

---


---

## 本章答辩卡（追问级）

**卡 1：表格里 `slot-scope="scope"` 的 `scope.row` 到底是什么？**

**10 秒答案**：`el-table` 在渲染每一行时，通过作用域插槽把「这一行的数据 + 行号 + 列」作为对象传给父组件；`scope.row` 就是当前行的数据对象。它让你可以在某一列里自定义显示逻辑，而不是只能显示字段原值。

**证据**：`zxcm-ui/src/views/conf/signingSubject/index.vue` 的表格列、`zxcm-ui/src/components/JTable/index.vue`（透传 `$scopedSlots`）。

**卡 2：`:prop="form"` 和 `prop="form"` 有什么区别？**

**10 秒答案**：带冒号 `:` 是 `v-bind`，引号里是 JS 表达式，绑定的是变量/对象；不带冒号是普通字符串，传给子组件的就是字面量 `"form"`。表单里 `:model="form"` 必须带冒号，否则校验拿不到对象。

**证据**：`zxcm-ui/src/views/conf/signingSubject/index.vue` 的 `<el-form :model="form" :rules="rules">`。

**卡 3：`this.$refs.form.validate(callback)` 为什么能调到 Element 表单的方法？**

**10 秒答案**：模板里 `ref="form"` 让 Vue 把该子组件实例登记到父组件的 `$refs` 上；`this.$refs.form` 拿到的是 `el-form` 组件实例，`validate` 是它暴露的方法。前提是 DOM 已经渲染，所以常在 `$nextTick` 或事件回调里用，不能在 `created` 里直接用。

**证据**：`zxcm-ui/src/views/conf/signingSubject/index.vue` 的 `ref="form"` 与 `submitForm` 方法。
