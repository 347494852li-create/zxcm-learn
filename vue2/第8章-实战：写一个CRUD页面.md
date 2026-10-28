# 第 8 章 · 实战：写一个 CRUD 页面

> **这一章解决什么问题**：前面七章都是「读懂」。这一章开始「写出来」。我们照着项目里标准管理页的结构，从零写一个「岗位管理（练习页）」：搜索、表格、分页、新增、修改、删除、表单校验、权限按钮，一个都不少。
>
> **学完标准**：不看代码，能独立写出一个能跑的管理页；能说清每一段模板、每一个方法为什么这么写；能把同样的结构套到任何一张业务表上。

---

## 一、这一章要做什么

### 1.1 目标页面

一个简化版岗位管理页：

```
┌──────────────────────────────────────────────┐
│ 关键字 [        ]   [搜索] [重置]            │
├──────────────────────────────────────────────┤
│ [+ 新增]                                      │
├──────────────────────────────────────────────┤
│ 岗位名称 | 岗位编码 | 流程编码 | 排序 | 操作  │
│ 项目经理 | PM       | PM001    | 1    | 修改 删除 │
│ ...                                           │
├──────────────────────────────────────────────┤
│              共 20 条  < 1 2 3 >              │
└──────────────────────────────────────────────┘

点「新增」/「修改」弹出：
┌──────────────────────────────┐
│ 新增岗位                      │
│ 所属组织  [树形下拉      ]    │
│ 岗位名称  [              ]    │
│ 岗位编码  [              ]    │
│ 流程编码  [              ]    │
│ 排序      [  1           ]    │
│ 岗位说明  [              ]    │
│              [取消] [确定]    │
└──────────────────────────────┘
```

### 1.2 用真实接口，不造假数据

我们直接复用项目里已有的岗位接口 `src/api/system/position.js`：

| 函数 | 作用 |
|---|---|
| `getPositionList(params)` | 分页查询 |
| `getPositionInfo(id)` | 查详情 |
| `createPosition(data)` | 新增 |
| `updatePosition(data)` | 修改 |
| `delPosition(id)` | 删除 |

后端接口已经存在，所以这个练习页能真正跑起来。

### 1.3 重要提醒

本项目学习版 `E:\Develop\project\zxcm-learn\zxcm-ui` 里的前端代码是**只读参考**。做练习请在你自己的副本里，或者：

1. 新建文件后，练习完删掉；
2. 临时路由也记得删。

如果你完全不想动学习版代码，也可以只读这一章，把代码抄到自己的笔记里，理解结构即可。

---

## 二、准备：两个新文件和一条临时路由

### 2.1 新建页面文件

```
zxcm-ui/src/views/demo/position/index.vue
```

### 2.2 加一条临时路由

打开 `src/router/index.js`，在 `constantRoutes` 数组的末尾（`/user` 那一段之后、`]` 之前）加：

```js
{
  path: '/demo',
  component: Layout,
  hidden: true,
  children: [
    {
      path: 'position',
      component: () => import('@/views/demo/position'),
      name: 'DemoPosition',
      meta: { title: '岗位管理（练习）' }
    }
  ]
}
```

`hidden: true` 表示不出现在侧边栏。登录后直接在地址栏访问：

```
http://localhost/demo/position
```

> `Layout` 在文件顶部已经导入（`src/router/index.js:7`），不需要额外 import。

---

## 三、第一步：搭模板骨架

先不管逻辑，把页面的五个区域写出来：

1. 搜索区；
2. 工具栏（新增按钮）；
3. 表格；
4. 分页；
5. 弹窗。

完整模板如下（先看结构，下面逐段解释）：

```html
<template>
  <div class="app-container">
    <!-- ① 搜索区 -->
    <el-form :model="params" :inline="true" @submit.native.prevent>
      <el-form-item label="关键字">
        <el-input
          v-model="params.keyword"
          placeholder="请输入岗位名称"
          clearable
          @keyup.enter.native="search"
        />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="el-icon-search" @click="search">
          搜索
        </el-button>
        <el-button icon="el-icon-refresh" @click="reset">
          重置
        </el-button>
      </el-form-item>
    </el-form>

    <!-- ② 工具栏 -->
    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="el-icon-plus"
          size="mini"
          v-hasPermi="['system:position:add']"
          @click="handleAdd"
        >
          新增
        </el-button>
      </el-col>
    </el-row>

    <!-- ③ 表格 -->
    <el-table v-loading="loading" :data="tableData" border>
      <el-table-column prop="fullName" label="岗位名称" />
      <el-table-column prop="enCode" label="岗位编码" />
      <el-table-column prop="flowCode" label="流程编码" />
      <el-table-column prop="sortCode" label="排序" width="80" align="center" />
      <el-table-column label="操作" width="150" align="center">
        <template slot-scope="scope">
          <el-button
            type="text"
            size="mini"
            v-hasPermi="['system:position:edit']"
            @click="handleUpdate(scope.row)"
          >
            修改
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-hasPermi="['system:position:remove']"
            @click="handleDelete(scope.row)"
          >
            删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- ④ 分页 -->
    <pagination
      :total="total"
      :page.sync="params.pageNum"
      :limit.sync="params.pageSize"
      @pagination="getList"
    />

    <!-- ⑤ 新增/修改弹窗 -->
    <el-dialog
      :title="title"
      :visible.sync="open"
      width="600px"
      append-to-body
      :close-on-click-modal="false"
    >
      <el-form ref="form" :model="form" :rules="rules" label-width="90px">
        <el-form-item label="所属组织" prop="organizeId">
          <treeselect
            v-model="form.organizeId"
            :options="treeData"
            placeholder="请选择归属组织"
          />
        </el-form-item>
        <el-form-item label="岗位名称" prop="fullName">
          <el-input v-model="form.fullName" placeholder="请输入岗位名称" />
        </el-form-item>
        <el-form-item label="岗位编码" prop="enCode">
          <el-input v-model="form.enCode" placeholder="请输入岗位编码" />
        </el-form-item>
        <el-form-item label="流程编码" prop="flowCode">
          <el-input v-model="form.flowCode" placeholder="请输入流程编码" />
        </el-form-item>
        <el-form-item label="排序" prop="sortCode">
          <el-input-number v-model="form.sortCode" :min="0" :max="9999" />
        </el-form-item>
        <el-form-item label="岗位说明" prop="description">
          <el-input
            v-model="form.description"
            type="textarea"
            :rows="4"
            placeholder="请输入岗位说明"
          />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="cancel">取 消</el-button>
        <el-button type="primary" :loading="submitLoading" @click="submitForm">
          确 定
        </el-button>
      </div>
    </el-dialog>
  </div>
</template>
```

### 3.1 逐段说明

**① 搜索区**

- `:model="params"`：告诉 `el-form` 数据对象是谁（方便校验和重置）；
- `:inline="true"`：表单项横向排列；
- `@submit.native.prevent`：阻止表单默认提交（第 2 章 §4.3 讲过）；
- `v-model="params.keyword"`：搜索关键词双向绑定；
- `@keyup.enter.native="search"`：回车触发搜索。

**② 工具栏**

- `v-hasPermi="['system:position:add']"`：没有该权限码时，按钮会被指令从 DOM 里删掉（第 2 章 §9.2）；
- `size="mini"`、`plain`：Element UI 的样式属性。

> 如果登录账号没有 `system:position:add` 权限，按钮会消失。这是正常现象。练习时可以暂时去掉 `v-hasPermi`，或者用管理员账号。

**③ 表格**

- `v-loading="loading"`：加载时显示遮罩；
- `:data="tableData"`：数据源；
- `border`：显示边框；
- `slot-scope="scope"`：操作列拿到当前行 `scope.row`（第 4 章 §8.3）；
- `align="center"`：列内容居中。

**④ 分页**

- `:total`：总条数；
- `:page.sync` / `:limit.sync`：页码和每页条数与 `params` 双向同步（第 4 章 §6）；
- `@pagination="getList"`：页码或每页条数变化后重新请求。

**⑤ 弹窗**

- `:visible.sync="open"`：弹窗显隐双向绑定；
- `:close-on-click-modal="false"`：点遮罩不关闭，防止误触；
- `ref="form"`：给 `el-form` 起引用名，后面用 `this.$refs.form.validate` 校验；
- `slot="footer"`：底部按钮插槽（第 4 章 §8.2）。

---

## 四、第二步：写 `<script>` 的骨架

```js
<script>
import {
  getPositionList,
  getPositionInfo,
  createPosition,
  updatePosition,
  delPosition
} from '@/api/system/position'
import { treeselect } from '@/api/system/dept'
import Treeselect from '@riophae/vue-treeselect'
import '@riophae/vue-treeselect/dist/vue-treeselect.css'

export default {
  name: 'DemoPosition',
  components: { Treeselect },
  data() {
    return {
      // 表格
      loading: false,
      tableData: [],
      total: 0,

      // 查询条件
      params: {
        keyword: '',
        pageNum: 1,
        pageSize: 10
      },

      // 弹窗
      open: false,
      title: '',
      submitLoading: false,
      treeData: [],
      form: {},

      // 表单校验规则
      rules: {
        organizeId: [
          { required: true, message: '请选择所属组织', trigger: 'change' }
        ],
        fullName: [
          { required: true, message: '请输入岗位名称', trigger: 'blur' },
          { max: 50, message: '岗位名称最多 50 个字符', trigger: 'blur' }
        ],
        enCode: [
          { required: true, message: '请输入岗位编码', trigger: 'blur' },
          { max: 50, message: '岗位编码最多 50 个字符', trigger: 'blur' }
        ],
        flowCode: [
          { required: true, message: '请输入流程编码', trigger: 'blur' },
          { max: 50, message: '流程编码最多 50 个字符', trigger: 'blur' }
        ]
      }
    }
  },
  created() {
    this.getList()
    this.getTreeselect()
  },
  methods: {
    // ...
  }
}
</script>
```

### 4.1 为什么这些 import

| import | 用途 |
|---|---|
| 五个接口函数 | 查询、详情、新增、修改、删除 |
| `treeselect` | 取组织树数据（`@/api/system/dept`） |
| `Treeselect` + CSS | 树形下拉组件 |

`Treeselect` 需要**局部注册**（`components: { Treeselect }`），因为项目没有全局注册它。

### 4.2 `data` 的分区

按用途分组，读起来清楚：

```
表格：loading、tableData、total
查询：params
弹窗：open、title、submitLoading、treeData、form
校验：rules
```

### 4.3 `created` 里做两件事

- `getList()`：拉第一页数据；
- `getTreeselect()`：拉组织树，供弹窗使用。

第 3 章讲过：拉数据放 `created`，不需要 DOM。

---

## 五、第三步：写方法

### 5.1 查询列表

```js
getList() {
  this.loading = true
  getPositionList(this.params)
    .then((res) => {
      this.tableData = res.rows
      this.total = res.total
    })
    .finally(() => {
      this.loading = false
    })
}
```

三个关键点：

1. **`res.rows` / `res.total`**：响应拦截器已经 `return res.data`，这里拿到的是后端 JSON 本身（第 7 章 §7.4）；
2. **`.finally`**：不管成功失败都关 loading。项目里常见 `.then` 里关，`.catch` 里再关一次；`.finally` 更简洁；
3. **整体替换 `tableData`**：第 3 章讲过，整体替换最省心地触发响应式更新。

### 5.2 搜索与重置

```js
search() {
  this.params.pageNum = 1
  this.getList()
},
reset() {
  this.params.keyword = ''
  this.search()
}
```

**搜索时为什么把页码重置为 1？**

假设现在在第 5 页，搜一个只有 2 条结果的词。如果页码还是 5，后端会返回空列表，用户以为没数据。重置到第 1 页才符合预期。

### 5.3 取组织树

```js
getTreeselect() {
  treeselect({ status: '0' })
    .then((res) => {
      this.treeData = res.data
    })
}
```

`status: '0'` 表示只取启用的组织。返回的是树形数组，直接交给 `Treeselect` 的 `:options`。

### 5.4 新增

```js
handleAdd() {
  this.resetFormData()
  this.title = '新增岗位'
  this.open = true
  this.$nextTick(() => {
    this.$refs.form && this.$refs.form.clearValidate()
  })
}
```

- `resetFormData()`：把 `form` 恢复成空对象（见 5.7）；
- `open = true`：打开弹窗；
- `$nextTick`：等弹窗里的表单渲染出来，再清校验（第 3 章 §八）。

**为什么不用 `resetFields()`？**

`resetFields()` 会把字段重置成**第一次渲染时**的值。如果先赋了新值再调，可能不是你想要的结果。这里直接重建 `form` 对象更直观，然后用 `clearValidate()` 只清红字。

### 5.5 修改

```js
handleUpdate(row) {
  this.resetFormData()
  this.title = '修改岗位'
  this.open = true
  getPositionInfo(row.id).then((res) => {
    this.form = res.data
    this.$nextTick(() => {
      this.$refs.form && this.$refs.form.clearValidate()
    })
  })
}
```

- 先打开弹窗（让表单渲染）；
- 再查详情；
- 详情回来后赋值给 `form`（`v-model` 自动把值显示到输入框）；
- `$nextTick` 清掉可能残留的校验提示。

> 也可以先查详情再开弹窗。项目里 `position/Form.vue:144-163` 是「先开弹窗，再在 `$nextTick` 里查详情」，两种都可以。

### 5.6 提交

```js
submitForm() {
  this.$refs.form.validate((valid) => {
    if (!valid) return

    this.submitLoading = true
    const save = this.form.id ? updatePosition : createPosition
    save(this.form)
      .then((res) => {
        this.$modal.msgSuccess(res.msg || '操作成功')
        this.open = false
        this.getList()
      })
      .finally(() => {
        this.submitLoading = false
      })
  })
}
```

要点：

1. `validate(callback)` 是异步的，`valid` 为 `true` 才继续；
2. `this.form.id ? updatePosition : createPosition`：有 id 是修改，没有是新增；
3. 成功后关弹窗、重新拉列表；
4. `$modal.msgSuccess` 是项目插件挂载的方法（`src/plugins/index.js:16`），等价于 `this.$message.success`；
5. `.finally` 关按钮 loading。

### 5.7 重置表单数据

```js
resetFormData() {
  this.form = {
    id: undefined,
    organizeId: undefined,
    fullName: '',
    enCode: '',
    flowCode: '',
    sortCode: 0,
    description: ''
  }
}
```

**为什么要重新赋一个完整对象，而不是逐个清空？**

1. 代码短；
2. 保证字段齐全，不会残留上一次的值；
3. 整体替换对象，响应式最可靠（第 3 章 §九）。

### 5.8 取消

```js
cancel() {
  this.open = false
  this.resetFormData()
}
```

### 5.9 删除

```js
handleDelete(row) {
  this.$confirm(`确认删除岗位「${row.fullName}」吗？`, '提示', {
    type: 'warning'
  })
    .then(() => delPosition(row.id))
    .then(() => {
      this.$modal.msgSuccess('删除成功')
      this.getList()
    })
    .catch(() => {})
}
```

要点：

1. `$confirm` 返回 Promise，点「确定」才继续；
2. 点「取消」会进入 `.catch`，用空函数吞掉，不报错；
3. 删除后重新拉列表；
4. 模板字符串 `` `...${row.fullName}...` `` 是 ES6 语法（第 0 章 §5.8）。

### 5.10 完整 `<script>`

```js
<script>
import {
  getPositionList,
  getPositionInfo,
  createPosition,
  updatePosition,
  delPosition
} from '@/api/system/position'
import { treeselect } from '@/api/system/dept'
import Treeselect from '@riophae/vue-treeselect'
import '@riophae/vue-treeselect/dist/vue-treeselect.css'

export default {
  name: 'DemoPosition',
  components: { Treeselect },
  data() {
    return {
      loading: false,
      tableData: [],
      total: 0,
      params: {
        keyword: '',
        pageNum: 1,
        pageSize: 10
      },
      open: false,
      title: '',
      submitLoading: false,
      treeData: [],
      form: {},
      rules: {
        organizeId: [
          { required: true, message: '请选择所属组织', trigger: 'change' }
        ],
        fullName: [
          { required: true, message: '请输入岗位名称', trigger: 'blur' },
          { max: 50, message: '岗位名称最多 50 个字符', trigger: 'blur' }
        ],
        enCode: [
          { required: true, message: '请输入岗位编码', trigger: 'blur' },
          { max: 50, message: '岗位编码最多 50 个字符', trigger: 'blur' }
        ],
        flowCode: [
          { required: true, message: '请输入流程编码', trigger: 'blur' },
          { max: 50, message: '流程编码最多 50 个字符', trigger: 'blur' }
        ]
      }
    }
  },
  created() {
    this.getList()
    this.getTreeselect()
  },
  methods: {
    getList() {
      this.loading = true
      getPositionList(this.params)
        .then((res) => {
          this.tableData = res.rows
          this.total = res.total
        })
        .finally(() => {
          this.loading = false
        })
    },
    search() {
      this.params.pageNum = 1
      this.getList()
    },
    reset() {
      this.params.keyword = ''
      this.search()
    },
    getTreeselect() {
      treeselect({ status: '0' }).then((res) => {
        this.treeData = res.data
      })
    },
    resetFormData() {
      this.form = {
        id: undefined,
        organizeId: undefined,
        fullName: '',
        enCode: '',
        flowCode: '',
        sortCode: 0,
        description: ''
      }
    },
    handleAdd() {
      this.resetFormData()
      this.title = '新增岗位'
      this.open = true
      this.$nextTick(() => {
        this.$refs.form && this.$refs.form.clearValidate()
      })
    },
    handleUpdate(row) {
      this.resetFormData()
      this.title = '修改岗位'
      this.open = true
      getPositionInfo(row.id).then((res) => {
        this.form = res.data
        this.$nextTick(() => {
          this.$refs.form && this.$refs.form.clearValidate()
        })
      })
    },
    submitForm() {
      this.$refs.form.validate((valid) => {
        if (!valid) return
        this.submitLoading = true
        const save = this.form.id ? updatePosition : createPosition
        save(this.form)
          .then((res) => {
            this.$modal.msgSuccess(res.msg || '操作成功')
            this.open = false
            this.getList()
          })
          .finally(() => {
            this.submitLoading = false
          })
      })
    },
    cancel() {
      this.open = false
      this.resetFormData()
    },
    handleDelete(row) {
      this.$confirm(`确认删除岗位「${row.fullName}」吗？`, '提示', {
        type: 'warning'
      })
        .then(() => delPosition(row.id))
        .then(() => {
          this.$modal.msgSuccess('删除成功')
          this.getList()
        })
        .catch(() => {})
    }
  }
}
</script>
```

### 5.11 可选：加一点样式

```html
<style scoped>
.mb8 {
  margin-bottom: 8px;
}
</style>
```

`scoped` 保证样式只作用于当前组件（第 0 章 §4.2）。

---

## 六、第四步：跑起来验证

### 6.1 启动

```bash
cd zxcm-ui
npm run dev
```

登录后访问 `http://localhost/demo/position`。

### 6.2 验收清单

- [ ] 页面能打开，搜索区、表格、分页、按钮都在；
- [ ] 表格能显示岗位数据，分页总数正确；
- [ ] 输入关键字回车或点搜索，列表变少；
- [ ] 点重置，关键字清空，列表恢复；
- [ ] 切换页码、每页条数，列表和页码正确；
- [ ] 点新增，弹窗打开，各字段为空；
- [ ] 不填必填项直接点确定，出现红色校验提示；
- [ ] 填完提交，提示成功，列表出现新数据；
- [ ] 点修改，弹窗显示当前行数据；改一个字段提交，列表更新；
- [ ] 点删除，先确认，再删除，列表刷新；
- [ ] 点取消，弹窗关闭且下次打开没有残留数据。

### 6.3 常见问题

| 现象 | 原因 | 排查 |
|---|---|---|
| 页面 404 | 临时路由没加或路径写错 | 看 `router/index.js` 的 path |
| 表格没数据 | 接口 404 / 参数不对 | Network 看请求和响应 |
| `res.rows` 是 undefined | 后端返回结构不是 `TableDataInfo` | 看 Response 的 JSON |
| 新增按钮消失 | 当前账号没权限码 | 临时去掉 `v-hasPermi` |
| 弹出「数据正在处理，请勿重复提交」 | 1 秒内重复提交 | 等 1 秒再点 |
| 校验提示不消失 | 没调 `clearValidate` | 检查 `$nextTick` 里的代码 |
| 组织下拉是空的 | `treeselect` 接口没返回数据 | Network 看 `/system/dept/treeselect` |

---

## 七、第五步：把结构套到别的表上

任何一张业务表的标准管理页，都是这五段：

```
① 搜索区      表单 + 查询条件 + 搜索/重置
② 工具栏      新增/批量删除/导出
③ 表格        el-table + 列 + 操作列
④ 分页        Pagination 组件
⑤ 弹窗        表单 + 校验 + 取消/确定
```

换一张表，只需要改四处：

| 要改的地方 | 从哪改到哪 |
|---|---|
| API 文件 | `@/api/system/position` → 新表的 API |
| 查询参数 | `params` 里的字段 |
| 表格列 | `el-table-column` 的 `prop` / `label` |
| 表单字段 | 弹窗里的 `el-form-item` + `rules` + `resetFormData` |

**其余结构完全不用动**。这就是标准管理页的价值：模式固定，复制、替换字段即可。

---

## 八、动手练习

### 练习 1：加一列

给表格加一列「创建时间」：

```html
<el-table-column prop="createTime" label="创建时间" width="160" />
```

验收：列表出现这一列，显示时间。

### 练习 2：加一个搜索条件

搜索区加一个「岗位编码」输入框，绑定 `params.enCode`，并在 `reset()` 里清空。

提示：后端是否支持这个参数，要以接口为准；如果没有效果，说明后端没接收这个字段，这也是一次真实的排查练习。

### 练习 3：加一个确认提示

删除时，如果岗位名称包含「管理员」，先弹一个额外提示或禁止删除。

提示：在 `handleDelete` 里判断 `row.fullName`。

### 练习 4：改用独立子组件

把弹窗拆成 `src/views/demo/position/Form.vue`：

- 父组件用 `ref="Form"` 调用 `init(id)`；
- 子组件提交成功后 `$emit('refreshDataList')`；
- 父组件 `@refreshDataList="getList"`。

验收：功能不变，但父页面模板变短。

参考第 4 章 §9 和项目里的 `position/Form.vue`。

### 练习 5：完整重写

关掉这一章，从空白文件开始，凭记忆再写一遍。写不出来的地方，就是你还没掌握的地方，回去看对应小节。

---

## 九、本章自测清单

- [ ] 标准 CRUD 页面的五段结构是什么？
- [ ] 搜索时为什么要把 `pageNum` 重置为 1？
- [ ] 列表接口返回后，为什么用 `res.rows` 而不是 `res.data.rows`？
- [ ] `v-loading`、`:data`、`slot-scope` 分别在这一页的哪里？
- [ ] 分页组件的 `:page.sync` 和 `@pagination` 各做什么？
- [ ] 新增和修改共用一个提交方法，怎么区分？
- [ ] `validate` 的回调参数 `valid` 是什么？
- [ ] 为什么新增时要 `$nextTick` 再操作表单？
- [ ] `clearValidate()` 和 `resetFields()` 有什么区别？
- [ ] 重置表单数据为什么整体赋一个新对象？
- [ ] `$confirm` 点取消时代码走到哪？为什么 `.catch(() => {})` 是必要的？
- [ ] `v-hasPermi` 没有权限时，按钮会怎样？
- [ ] 换成另一张表，需要改哪四处？

---

## 十、下一章前瞻

第 9 章进入真实页面精读：

- `src/views/conf/billRule/index.vue`：Tab 切换、动态表单行、`currentData`、保存与导出；
- `src/views/system/position/index.vue`：左侧组织树 + 右侧列表联动、`JTable`、`dict-tag`、独立 `Form.vue`、`Selector.vue`。

会把每一块模板、每一个方法、每个按钮对应的接口都标出来，并指出项目里的历史代码、死代码和小瑕疵。

> 这一章最重要的一句话：**管理页是模板化的：搜索 → 工具栏 → 表格 → 分页 → 弹窗，五个区域各司其职；换表只换字段，不换骨架。**
