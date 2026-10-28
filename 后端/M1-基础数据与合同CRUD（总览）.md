# M1 · 基础数据 + 合同草拟

> **一句话**：先把「合同要用到的名词」建起来（签约主体、合作方、编号规则、合同类型），再做「合同草拟」这张主表。
>
> 上一篇：[M0 骨架](M0-骨架（总览）.md) · 下一篇：[M2 审批流](M2-审批流与事件状态机（总览）.md)
> 前置阅读：**[D2 前后端调用链路](D2-前后端调用链路.md) 第零部分**（权限三层、RBAC 五张表、超管判定）——不看这个，M1 里 `@SaCheckPermission` 那些东西会看不懂。
> 🎤 本里程碑面试题：[面试速通 · 3. M1 章节](面试通关.md#3-m1--基础数据与合同-crud)
> 🔧 逐模块施工单（照着敲）：[施工单/M1-T2](施工单/M1-施工单.md) → [T3](施工单/M1-施工单.md) → [T4](施工单/M1-施工单.md)
> 🧭 前端视角：[E1 §8 页面↔施工单映射](../前端/E1-zxcm-ui工程结构.md) · [前端代码走读](../前端/E3-前端代码走读.md) ｜ 附录：[D0 表](D0-底座表地图.md) · [D1 鉴权](D1-SaToken鉴权链路.md) · [D2 链路](D2-前后端调用链路.md) · [D3 数据权限](D3-数据权限链路.md)

---

## M1 做完是什么样

| 页面 | 能做什么 | 难度 |
|---|---|---|
| 合同基础配置 → 签约主体管理 | 增删改查 + 启停用 + Excel 导入导出 | ★★ 简单 |
| 合作方管理 → 合作方管理 | 上面那些 + 联系人/银行账号（**存 JSON 列，不是子表**）+ 启停用 | ★★★ 中等 |
| 合同基础配置 → 编号规则设置 | 增删改查 + **自动生成合同编号** | ★★★ 中等（有算法） |
| 合同基础配置 → 合同类型配置 | 增删改查（动态表单配置） | ★★ 简单 |
| 合同管理 → 合同草拟 | 新建合同（选模板、选主体/相对方、填 **85 列**的表）、列表、编辑、删除、新版本 | ★★★★★ 复杂 |

**主链位置**：

```
        【M1 就是这一段】
   签约主体 + 合作方 + 编号规则 + 合同类型
                 ↓
            合同草拟（列表能出数、能新建一条）
                 ↓
            提交审批 → ……（M2）
```

---

## ★ 先看懂这一节，M1 的一半就通了 ★

### 一个 CRUD 模块的固定套路：7 个必需文件 + 可选配件

RuoYi 的分层是**固定套路**。任何一个 CRUD 模块（不管是签约主体还是合作方）都是这 9 个文件，**位置和命名都是模板化的**：

| # | 文件 | 位置（以签约主体为例） | 职责 |
|---|---|---|---|
| 1 | **实体** | `domain/TSigningSubject.java` | 对应数据库表，`@TableName("conf_signing_subject")` |
| 2 | **BO** | `domain/bo/TSigningSubjectBo.java` | Business Object = **入参**。前端提交上来的东西 |
| 3 | **VO** | `domain/vo/TSigningSubjectVo.java` | View Object = **出参**。返回给前端的东西 |
| 4 | **Mapper** | `mapper/TSigningSubjectMapper.java` | `extends BaseMapperPlus<...>`，**基本 CRUD 一行 SQL 都不用写** |
| 5 | **Service 接口** | `service/ITSigningSubjectService.java` | 声明业务方法 |
| 6 | **Service 实现** | `service/impl/TSigningSubjectServiceImpl.java` | 写业务逻辑 |
| 7 | **Controller** | `controller/TSigningSubjectController.java` | 暴露 HTTP 接口 + 权限注解 |
| 8 | 导入 VO（可选） | `domain/vo/TSigningSubjectImportVo.java` | Excel 导入用 |
| 9 | 导入监听器（可选） | `listener/SigningSubjectImportListener.java` | EasyExcel 回调 |
| — | **前端 API** | `zxcm-ui/src/api/conf/signingSubject.js` | 每个接口一个 `export function` |
| — | **前端页面** | `zxcm-ui/src/views/conf/signingSubject/index.vue` | 列表 + 搜索栏 + 弹窗表单 |

**复制实体类之前，先核对那两个类注解**（这个项目里写法是统一的，我实测过三个模块）：

```java
@TableName("conf_signing_subject")      // ← 表名小写
public class TSigningSubject {
    @TableId(value = "id")              // ← 不带 type
    private String id;
```

`@TableId` **不写 `type` 就用全局配置**，而全局是 `idType: ASSIGN_ID`（雪花号）。这个项目里几乎所有实体都是这个写法——**列类型是 `varchar(64)` 也照样走雪花**（雪花是 19 位数字，装 varchar(64) 绰绰有余），所以**别被「varchar 主键」误导成 UUID**。

> 💡 **这就是为什么 M1 可以「照着复制」**：你不需要理解为什么这么分层，你只要知道「第 7 个文件里那个 `@RequestMapping` 的字符串，决定了前端 js 里 `url:` 写什么」。
>
> **一个 CRUD 模块的 4 条链路**（对照 [D2 第三部分](D2-前后端调用链路.md)）：
> ```
> index.vue 的「查询」按钮 → api/conf/signingSubject.js 的 listXxx()
>   → axios → GET /conf/signingSubject/list
>   → Controller.list() → Service.selectXxxList() → Mapper.selectVoList()
>   → MySQL → 原路返回 → 表格渲染
> ```

### 一个好消息：这个项目的 Mapper **几乎没有 XML**

答案册 `zxcm-contract` 模块里，**31 个 Mapper XML（全项目）中，这个模块只占 1 个**（`ContractRemarksMapper.xml`）。

原因是这个项目基于 **RuoYi-Vue-Plus**，它把 RuoYi 原来的「XML 写 SQL」风格改掉了：

```java
public interface TSigningSubjectMapper
    extends BaseMapperPlus<TSigningSubjectMapper, TSigningSubject, TSigningSubjectVo> {

    @Transactional
    @Update("update CONF_SIGNING_SUBJECT set BUSINESS_END_DATE=NULL WHERE ID=#{id}")   // ← SQL 直接写注解里
    Integer updateSetDateNull(@Param("id") String id);
}
```

**所以复制清单里通常没有 XML 要拷。** 只有当你要写「多表关联的复杂分页查询」时才需要建 XML。

### 一个坏消息：MyBatis-Plus 帮你做的事，都依赖「字段名对得上」

`BaseMapperPlus` 白送你的方法（`selectVoList` / `selectVoPage` / `insert` / `updateById` / `deleteById`）能自动跑，靠的是 `application.yml` 这几行全局配置：

```yaml
mybatis-plus:
  configuration:
    mapUnderscoreToCamelCase: true    # ← 表列 SIGNING_SUBJECT_NAME ↔ 实体 signingSubjectName
    logImpl: org.apache.ibatis.logging.stdout.StdOutImpl   # ← SQL 直接打控制台（调试神器）
  global-config:
    dbConfig:
      idType: ASSIGN_ID               # ← 主键自动雪花；本表用 varchar 主键时靠实体上自己声明
      logicDeleteValue: 2             # ← 已删除值 = 2
      logicNotDeleteValue: 0          # ← 未删除值 = 0
      insertStrategy: NOT_NULL        # ← insert 时 null 字段不进 SQL（用表默认值）
```

**三个必须记住的后果**：

1. **实体类字段名必须和表列名对得上**（靠驼峰转换）。对不上就自己加 `@TableField("列名")`。这是「复制实体类」时最容易出错的地方。
2. **逻辑删除是自动的**。`contract_draft` 表的 `DEL_FLAG` 注释写着「0 未删除 / 2 已删除」，和这里的 `logicDeleteValue: 2` 是**配对的**。所以 `DELETE` 接口实际执行的是 `UPDATE ... SET DEL_FLAG = 2`，**数据不会真被删**。这也是为什么 M1 验收时查得到「删掉」的记录。
3. **`logImpl: StdOutImpl` 意味着控制台能看到每一条真实 SQL**。调试 M1 的第一手段就是**看 IDEA 控制台**，而不是猜。

---

## 📊 M1 用到的技术 & 表（面试要背的就是这一节）

### 技术清单

| 技术 | 版本 | 在 M1 里干什么 | 关键点 |
|---|---|---|---|
| **Spring Boot** | 2.6.4 | 整个后端的容器 | Java 11 |
| **MyBatis-Plus** | 3.5.2 | 所有数据库访问 | `BaseMapperPlus` 白送 CRUD；**几乎不用 XML**，复杂 SQL 用 `@Select`/`@Update` 注解 |
| **Sa-Token** | 1.29.0 | 接口权限 | `@SaCheckPermission("conf:xxx:add")`，MVC 拦截器实现，不是 AOP |
| **EasyExcel** | RuoYi 封装 | 签约主体/合作方的导入导出 | `ImportListener` 回调 + `ExcelResult` |
| **Jackson + 自研 `JsonUtil`** | — | **JSON 列**读写（联系人、银行账号） | 转换责任在**前端**，后端只透传 `String` |
| **Hutool** | — | 小工具 | `BeanUtil.copyProperties(bo, entity)`，Bo→Entity 的常规姿势 |
| **Vue 2 + Element UI** | 2.6.12 / 2.15.6 | 前端页面 | 不由你编写，整包复制 |

### M1 的表（12 张，全部由 `04-biz-tables.sql` 建好）

| 表 | 列数 | 功能归属 | **为什么这么建** |
|---|---|---|---|
| `conf_signing_subject` | 26 | T2 签约主体 | 联系人和银行卡塞进 `CONTACTS`/`BANK_JSON` 两个 **JSON 列**（`varchar(4000)`），不建子表 |
| `conf_contract_counterpart` | 40 | T3 合作方 | 同上用 JSON 列；另有 `TYC_RESULT`（天眼查原始返回）、`BAN`/`AVERAGE_SCORE`/`CREDIT_STATUS` 等**风控字段**——说明这张表不只是通讯录，还承担**合作方风险评估** |
| `conf_bill_rule` | 15 | T4 编号规则 | 规则本体（`RULE longtext` 存整条规则表达式）+ 当前计数器 `THIS_NUMBER` + `THIS_NUMBER_DATE`（按日期重置流水号） |
| `conf_bill_rule_ext` | 15 | T4 编号规则 | **拆出来的计数器子表**——把「会频繁被 `UPDATE` 的计数字段」和「很少变的规则定义」分开，减少行锁冲突 |
| `form` | 16 | T5 合同类型 | `PARENT_ID` **自关联成树**（同 `sys_menu`、`sys_dept`）；`JSON` 列存动态表单配置；`INSTANCE_DEF_KEY` 是**指向 M2 审批流的钩子** |
| `contract_draft` | **85** | T6 合同草拟 | **宽表 + 多状态列**（详见 T6 原理）。整个系统的心脏，M2-M4 都在这张表上加字段 |
| `contract_draft_partnerids` | 8 | T6 合同相对方 | **真·关联表**：`CONTRACT_DRAFT_ID` + `PARTNER_ID`。与上面的 JSON 列形成对照（见下方） |
| `contract_draft_files` | 17 | T6 合同文件 | **文件版本表**：`FILE_GROUPS_VERSION`（`V1.0`/`V1.1`）+ `FILE_GROUPS_JSON`。一份合同多个版本 |
| `contract_his` | 80 | T6 历史版本 | 合同**快照**表——字段数和 `contract_draft` 几乎一样，因为它是「某一时刻的整条合同」的副本 |
| `contract_his_files` | 17 | T6 历史版本 | 历史版本对应的文件 |
| `contract_point_log` | 13 | 跨 M1-M4 | **事件流水表**：`CONTRACT_ID` + `POINT`（事件类型）+ `BUSINESS_JSON`（事件数据）。M2 起是「到底发生了什么」的权威来源 |
| `conf_file` | 12 | 跨模块 | **统一文件表**：所有模块的附件都存这里，靠 `MODULE` 列区分来源；`ONLYOFFICE_KEY` 是 M3 在线编辑的回调凭据 |

### ★ 两条相反的「一对多」路线（本里程碑最值钱的一张对照表）

**同一个系统里，两种「一对多」并存**——这正好说明选型是跟着需求走的：

| | 合作方 ↔ 联系人 | 合同 ↔ 相对方 |
|---|---|---|
| **怎么存** | `conf_contract_counterpart.CONTACTS` 一个 **JSON 列** | `contract_draft_partnerids` 一张**关联表** |
| **能单独查吗** | ❌ 不能。查不了「哪个合作方的联系人叫张三」 | ✅ 能。`SELECT ... WHERE PARTNER_ID=?` 反查「这个合作方参与了哪些合同」 |
| **谁负责转换** | 前端 `JSON.parse` / `JSON.stringify` | 后端正常增删行 |
| **更新方式** | 整字段覆盖写 | 增/删其中一行 |
| **上限** | `varchar(4000)`，多了就爆 | 无上限 |
| **适合** | **纯附属信息**：只跟着主表一起读、不被引用 | 要被**单独查询/关联/统计**的数据 |

> 🎯 **面试怎么答**：被问「一对多怎么设计」时，**先反问需求**，再给答案：
> 「如果子数据只是跟着主表读写的附属信息，不单独查、不被外部引用 → **JSON 列**，省一张表、更新还天然是整体替换；
> 如果要被单独查询、关联、统计，或者子数据自己有状态和权限 → **关联表**。
> 我们这个项目里两种都有：合作方的联系人用 JSON 列，合同的相对方用关联表 `contract_draft_partnerids`。」
>
> **能举出系统内两种做法并说出各自理由**，比背「一对多要建子表」高一个层级。

### 另外三个跨表规律（记住能串起 M1-M4）

1. **所有业务表都有 `DEL_FLAG`（或 `DELETE_MARK`）**：`char(1) DEFAULT '0'`，配 MyBatis-Plus 的 `logicDeleteValue: 2`。**删除永远是 `UPDATE SET DEL_FLAG=2`**。
2. **"被引用过的东西"从不就地改，一律开新版本或软删除**：`contract_draft_files`（文件版本）、`contract_his`（合同快照）、`form`/模板的版本与停用。**这条规律在 M1-M4 出现多次**，是系统的设计哲学。
3. **业务表零外键**：36 个 `FOREIGN KEY` 全是 Activiti 的 `ACT_FK_*`。表间关联（合同→相对方→合作方）**全靠应用层维护**。

---

## T1 · 让业务菜单先回到库里

🎯 **目标**：M0 把业务菜单删干净了，侧边栏里根本没有「签约主体管理」这一项。**所有 M1-M4 的任务都依赖这一步。**

🖥 **对应页面**：右侧任意业务菜单
🔌 **对应接口**：`GET /getRouters`

▶️ **操作步骤**

M0 的底座脚本只保留了 75 个框架菜单。我已经从答案册提取好了一个恢复脚本：

```bash
cd "E:/Develop/project/zxcm-learn/script/sql"
mysql --no-defaults --ssl-mode=DISABLED -h 192.168.56.10 -u root -proot zxcm \
  --default-character-set=utf8mb4 < 02-biz-menu.sql
```

✅ **验证**

```sql
SELECT COUNT(*) FROM sys_menu;                                      -- 预期 170
SELECT COUNT(*) FROM sys_menu WHERE MENU_TYPE='M' AND PARENT_ID=0;  -- 预期 8
SELECT COUNT(*) FROM sys_role_menu;                                 -- 预期 446
```

刷新浏览器（**要重新登录**，见下方原理），左侧应该出现 7 个业务目录。

> ⚠️ **菜单出现了，但点进去是 404**——因为后端还没有对应的 Controller。**这是预期的**，M1-M4 就是逐个把它点亮。
> 只有「系统管理 → 用户管理」这几个框架页面是通的。

> 📌 **如果你是从 M0 刚过来的**，T1 的手把手版本（含 M0 收尾 + 逐条起跑线检查）在
> [M0-施工单.md](施工单/M0-施工单.md)。

🧠 **原理 · 为什么菜单要重新登录才出现**

因为**权限码和菜单树是在登录时查一次、之后一直用缓存**：

- 登录时 `SysPermissionService.getMenuPermission()` 从 `sys_menu` 查出权限码集合 → 塞进 `LoginUser`
- `LoginUser` 存进 **Redis 的 token-session**（key 形如 `token:login:token-session:<JWT>`）
- `/getRouters` 和 `@SaCheckPermission` 后续都**读 Redis，不读数据库**

**所以改了菜单/权限，不重新登录不生效。** 这是面试高频题（「权限改了为什么不生效」），也是 M1 调试时最常踩的坑。

> 📌 **例外**：登录用户是超管时，`permissions` 直接返回 `["*:*:*"]`（短路分支，**不查 `sys_menu`**），所以超管改菜单立刻生效、但**权限码的校验也永远通过**——这意味着**用超管测不出权限配置对不对**。
> 想真正验证权限，得建个普通用户。**这正好是 T2 之后的加餐**，留到 T5 一起做。

> 🔧 **脚本是怎么来的**：以数据库现有的 75 个菜单为准，和答案册的 170 个做差集，得到 95 个待恢复菜单 + 313 条角色绑定。
> ⚠️ 清理语句**必须用显式 id 列表**，不能写 `MENU_ID >= 1000000000000000000`——`人员组织管理(1578230801211191297)` 也是雪花 id，但它是框架菜单，会被误删（我第一版就是这么写错的）。

---

## T2 · 签约主体管理（最简单的那个，先拿它练手）

🎯 **目标**：一个完整的 CRUD 模块，包含列表、搜索、新增、修改、删除、启停用、Excel 导入导出。**把这一套流程跑顺，后面 3 个模块都是重复动作。**

🖥 **前端入口**
- 菜单：**合同基础配置 → 签约主体管理**
- 路由：`/conf/signingSubject`
- 页面文件：`zxcm-ui/src/views/conf/signingSubject/index.vue`（+ `upload.vue` 导入弹窗）
- API 文件：`zxcm-ui/src/api/conf/signingSubject.js`

🔌 **接口入口**（`doc.html` 里找 `签约主体` 分组）

| 方法 | URL | 权限码 | 用途 |
|---|---|---|---|
| GET | `/conf/signingSubject/list` | `conf:signingSubject:list` | 分页列表 |
| GET | `/conf/signingSubject/{id}` | `conf:signingSubject:query` | 详情 |
| POST | `/conf/signingSubject` | `conf:signingSubject:add` | 新增 |
| PUT | `/conf/signingSubject` | `conf:signingSubject:edit` | 修改 |
| DELETE | `/conf/signingSubject/{ids}` | `conf:signingSubject:remove` | 删除（路径是 `{ids}`，但 **Service 里一次只允许删一条**） |
| PUT | `/conf/signingSubject/changeStatus` | `conf:signingSubject:edit` | 启停用 |
| POST | `/conf/signingSubject/importData` | `conf:signingSubject:edit` | Excel 导入 |
| POST | `/conf/signingSubject/importTemplate` | — | 下载导入模板 |
| GET | `/conf/signingSubject/searchSubject` | —（在**方法体里**判断） | **受限检索**：没有 `contract:subjectAll:search` 权限的人，必须输入完整公司名、且只允许命中 1 条 |
| GET | `/conf/signingSubject/showList/select` | — | **下拉选项**（不分页的那个是它，不是 `searchSubject`） |

> ⚠️ **两个容易看错的点**：
> - `list` 在 Controller 上**没有** `@SaCheckPermission`（登录即可调）；表里的 `conf:signingSubject:list` 只是**菜单**的权限串（`02-biz-menu.sql` 里那条 `C` 类型菜单），别把它当成接口鉴权。
> - **`searchSubject` 不是下拉接口**。它是「受限检索」（防数据爬取，见 [施工单 4.7](施工单/M1-施工单.md)）；真正给别的表单「选一个签约主体」用的是 `showList/select`。

📋 **复制清单**

**后端**（答案册 `zhaoxin-contract/zxcm-contract/src/main/java/com/ruoyi/conf/` → 本项目 `zxcm-learn/zxcm-contract/src/main/java/com/ruoyi/conf/`）

| 答案册文件 | 说明 |
|---|---|
| `domain/TSigningSubject.java` | 实体（`contacts` / `bankJson` 是 **String**，存 JSON） |
| `domain/bo/TSigningSubjectBo.java` | 入参 |
| `domain/vo/TSigningSubjectVo.java` | 出参 |
| `domain/vo/TSigningSubjectImportVo.java` | 导入出参 |
| `domain/vo/ContactInfo.java` · `BankInfo.java` | **JSON 列的结构化载体**（不是子表 VO，见下方原理）。注意 `ServiceImpl` 只用到 `ContactInfo`；`BankInfo` 要到 M2 的 `ContractAuditServiceImpl` 才被 import |
| `util/JsonUtil.java` | JSON ↔ List 工具 |
| `util/ComplexPropertyPreFilter.java` | **别漏**：`JsonUtil` 依赖它（fastjson 属性过滤器），只拷 `JsonUtil` 会编译不过 |
| `mapper/TSigningSubjectMapper.java` | Mapper（**无 XML**，唯一一个注解 SQL 是 `updateSetDateNull`） |
| `service/ITSigningSubjectService.java` | Service 接口 |
| `service/impl/TSigningSubjectServiceImpl.java` | Service 实现（**注意 `queryPageList` 里有覆盖逻辑**） |
| `controller/TSigningSubjectController.java` | Controller |
| `listener/SigningSubjectImportListener.java` | Excel 导入监听器 |

**前端**

| 答案册文件 | 目标 |
|---|---|
| `zxcm-ui/src/api/conf/signingSubject.js` | 同名 |
| `zxcm-ui/src/views/conf/signingSubject/index.vue` | 同名 |
| `zxcm-ui/src/views/conf/signingSubject/upload.vue` | 同名 |

**数据库**：`conf_signing_subject`（26 列）—— **M0 的底座脚本没有它**（只保留了 `sys_*`/`gen_*`）。

**不用手工提取建表语句了**，我把 M1-M4 要用的 25 张业务表一起提到 `04-biz-tables.sql` 里了：

```bash
cd "E:/Develop/project/zxcm-learn/script/sql"
mysql --no-defaults --ssl-mode=DISABLED -h 192.168.56.10 -u root -proot zxcm \
  --default-character-set=utf8mb4 < 04-biz-tables.sql
```

跑完 `conf_signing_subject` 里已经有 **1 条演示数据**，所以 T2 的列表一进去就能出数，方便你对照。

> ✅ **好消息：业务表一张外键都没有。**
> 全库 36 个 `FOREIGN KEY` 全是 Activiti 自己的 `ACT_FK_*`（引擎表内部约束），`conf_*` / `contract_*` / `template_*` **零外键**。所以不用排依赖顺序，也不会因为建表顺序报错——表之间的关联（比如「合同引用签约主体」）**全靠应用层维护**。
> 📌 这是「**逻辑外键 vs 物理外键**」的取舍：无物理外键 → 方便分库分表、方便数据迁移、不会被级联删除坑到；代价是**数据库不管一致性，得靠代码和测试兜**。

▶️ **操作步骤**

1. 先把建表语句补进库里，确认表存在
2. 按上表复制后端文件（**保持包路径一致**，`com.ruoyi.conf.*`）——本模块 7 个 + 依赖闭包（`ContactInfo` / `JsonUtil` / `ComplexPropertyPreFilter` / `TSigningSubjectImportVo` / `SigningSubjectImportListener`），共 12 个；再改掉 **4 处死 import + 1 处大写表名**
3. 复制 2 个前端文件
4. 重启后端（IDEA 里重新 Run）
5. **前端不用重启**（dev-server 有热更新），但菜单要重新登录才出现（T1 已讲过）

✅ **跑通验证（三条硬标准）**

| # | 怎么验 | 预期 |
|---|---|---|
| 1 **接口通** | `doc.html` 里调 `GET /conf/signingSubject/list` | `{"code":200,"data":{"total":N,...}}` |
| 2 **功能通** | 浏览器：菜单 → 签约主体管理 → 新增一条 → 列表能查到 → 编辑 → 删除 | 每步都有成功提示，列表实时刷新 |
| 3 **数据对** | `SELECT * FROM conf_signing_subject;` | 你新增的那条在，删除的那条 `DEL_FLAG=2` |

🧠 **原理 · 值得讲的三个点**

1. **`DELETE` 为什么不真删**：接口叫 `remove`，但 MyBatis-Plus 的 `deleteBatchIds()` 被 `logicDeleteValue: 2` 拦截，实际发的是 `UPDATE ... SET DEL_FLAG = 2`。**去看控制台的 SQL 就能看到**。业务上叫「软删除」，这样合同里引用的合作方被"删"了也不会让合同数据悬空。
   而且 Service 里有 `if (ids.size() > 1) throw new ServiceException("对不起，该模块不支持批量删除")`——**路径写着 `{ids}`，实际一次只能删一条**。这是「接口签名比真实能力大」的典型例子。
   > 面试点：「为什么用逻辑删除？」→「历史合同引用了这个合作方，物理删除会让合同查不到相对方信息。」
2. **`changeStatus` 为什么单独一个接口**：启停用只改一个字段，但要走独立的权限校验（有的角色能改数据、不能停用主体）。这叫**按操作粒度授权**。
3. **`searchSubject` 和 `list` 为什么要两个接口**：`list` 是**分页列表**（给表格用）；`searchSubject` 是**不分页的下拉选项**（给别的表单的「选择签约主体」用）。同一个表，两种用途，性能要求不同。
   > 这是很典型的「一个实体多种查询视图」，M1 里到处都是（`searchPartner` / `searchMerchantById` 同理）。
4. **签约主体的联系人也存 JSON 列，而且列表接口会「改写」它**。看 `TSigningSubjectServiceImpl.queryPageList`：

   ```java
   for (TSigningSubjectVo vo : result.getRecords()) {
       List<ContactInfo> list = new ArrayList<>();
       ContactInfo info = new ContactInfo();
       SysUser user = sysUserMapper.selectById(LoginHelper.getLoginUser().getUserId());
       info.setName(user.getNickName());
       info.setEmail(user.getEmail());
       info.setTel(user.getPhonenumber());
       info.setRemarks("sysUser");
       list.add(info);
       vo.setContacts(JsonUtil.getObjectToString(list));   // ← 把查出来的 contacts 覆盖成"当前登录人"
   }
   ```
   注释写着「默认返回的联系人是当前登录人」——**列表接口故意不返回真实联系人**，而是塞了当前用户的信息（后面接别的东西时会用到，比如「以我为主体重建一条记录」）。

   > 🎯 **面试点**：这是 JSON 列的一个**隐性好处**——字段级改写的成本极低，一个 `vo.setContacts(...)` 就完事。
   > 如果联系人在子表里，你要么在查询后额外查一遍子表再拼，要么在 SQL 里做 `UNION`/子查询，代码量和心智负担都更大。
   > **但要诚实地说出代价**：这种做法让「同一个 VO，列表和详情返回的 contacts 语义不同」——**读代码的人如果不知道这行，会以为数据库里的联系人就是当前登录人**。这是可维护性的坑，属于「能跑但不好」。真正的做法是拆一个列表专用 VO，把「默认联系人」这个字段换个名字（比如 `defaultContact`）。

---

## T3 · 合作方管理（一对多塞进了一列）

🎯 **目标**：在 T2 的套路基础上，看懂这个项目怎么处理「**一个合作方有多个联系人 / 多个银行账号**」。

**先说结论，免得你找错方向**：这个项目**没有联系人子表，也没有财务子表**。联系人和银行账号以 **JSON 文本**存在 `conf_contract_counterpart` 的 `CONTACTS` / `BANK_JSON` 两个列里，一列搞定。

🖥 **前端入口**
- 菜单：**合作方管理 → 合作方管理**（注意：目录名和页面名一样，都叫"合作方管理"）
- 路由：`/partner/contractCounterpart`
- 页面：`views/conf/contractCounterpart/index.vue` + `edit.vue`
- API：`api/conf/contractCounterpart.js`

🔌 **接口入口**

| 方法 | URL | 权限码 |
|---|---|---|
| GET | `/conf/contractCounterpart/list` | `conf:contractCounterpart:list` |
| GET | `/conf/contractCounterpart/{id}` | `conf:contractCounterpart:query` |
| POST | `/conf/contractCounterpart` | `conf:contractCounterpart:add` |
| PUT | `/conf/contractCounterpart` | `conf:contractCounterpart:edit` |
| DELETE | `/conf/contractCounterpart/{ids}` | `conf:contractCounterpart:remove` |
| PUT | `/conf/contractCounterpart/changeStatus` | `conf:contractCounterpart:edit` |
| POST | `/conf/contractCounterpart/importData` | `conf:contractCounterpart:import` |
| POST | `/conf/contractCounterpart/importTemplate` | — |
| GET | `/conf/contractCounterpart/hide` · `/show` | — | 展开/收起相关行（UI 用） |

📋 **复制清单**

| 答案册文件（`com/ruoyi/conf/`） | 说明 |
|---|---|
| `domain/TContractCounterpart.java` | 实体（`contacts` / `bankJson` 是 **String**） |
| `domain/bo/TContractCounterpartBo.java` | 入参 |
| `domain/vo/TContractCounterpartVo.java` | 出参 |
| `domain/vo/ContractCounterpartContactsVo.java` | **不是子表 VO**——导出 Excel 时把 JSON 反序列化用的 |
| `domain/vo/ContractCounterpartFinanceVo.java` | 同上，银行账号用 |
| `domain/vo/TContractCounterpartImportVo.java` | 导入 VO |
| `domain/vo/BankInfo.java` · `ContactInfo.java` | 值对象（**签约主体/审批模块也在用**，不是本模块专用） |
| `util/JsonUtil.java` | **JSON ↔ List 工具**（导出摊平时用） |
| `mapper/TContractCounterpartMapper.java` | Mapper（**无 XML**） |
| `service/ITContractCounterpartService.java` + `impl/` | Service |
| `controller/TContractCounterpartController.java` | Controller |
| `listener/CounterPartImportListener.java`（在 `contract/listener/`） | 导入监听器 |
| `enums/PartnerEnums.java` | 枚举（合作方类型等） |
| 前端 `api/conf/contractCounterpart.js` · `views/conf/contractCounterpart/{index,edit}.vue` | 前端 |

**数据库**：**只有 `conf_contract_counterpart` 一张表**。没有 contacts / finance 子表——别再去找了。

✅ **跑通验证**：新增一条合作方时填 2 个联系人，保存后用 SQL 看这一列：

```sql
SELECT ID, NAME, CONTACTS, BANK_JSON FROM conf_contract_counterpart WHERE ID='<刚建的ID>';
-- CONTACTS 长这样（一整个 JSON 数组存在一个字符串列里）：
-- [{"name":"张三","tel":"13698989898","email":"sss@qq.com","address":"...","remarks":"..."},
--  {"name":"李四","tel":"137...","email":"...","address":"...","remarks":"..."}]
```

🧠 **原理 · 一对多为什么可以塞进一列**

**三层的类型是不一样的，这是理解它的关键**：

| 层 | contacts 的类型 | 谁负责转换 |
|---|---|---|
| 数据库 `CONTACTS varchar(4000)` | 一个字符串（JSON 文本） | —— |
| 后端实体 `private String contacts` | **String**，原样透传 | **没人转**，Service 里搜不到任何处理 |
| 前端 `form.contactsArray` | **数组**（子表格用 `v-for` 渲染） | **前端自己转** |

前端的两个转换动作（`views/conf/contractCounterpart/edit.vue`）：

```javascript
// 提交前（:692）——数组 → JSON 字符串
this.form.contacts = JSON.stringify(this.form.contactsArray);
this.form.bankJson = JSON.stringify(this.form.bankJsonArray);

// 回显时（:597）——JSON 字符串 → 数组
response.data.contactsArray = JSON.parse(response.data.contacts);
```

**所以 `ContractCounterpartContactsVo` 确实存在，但不是用来存子表的**——它是 `TContractCounterpartController` 导出 Excel 时用的：`JsonUtil.getJsonToList(vo.getContacts(), ContractCounterpartContactsVo.class)` 把 JSON 反序列化成对象，再把「联系人1姓名 / 联系人2姓名 / …」摊平成 Excel 的列（因为 Excel 一行放不下变长的数组）。

**为什么用 JSON 列而不是子表？** 两条正好对上的理由：

1. **这些数据永远跟着主表一起读写**。没有「单独查某个联系人」「按联系人手机号搜合作方」这种需求。子表的唯一优势是**能被单独索引/关联查询**——用不上的话，它只带来成本（多一张表、多一轮 insert、多一次 join）。
2. **子表的更新很别扭**。真要建成子表，更新时得「先删光再全插」——因为前端是整表单提交的，你没法知道用户删了哪一行。**JSON 列天然没有这个问题：一个字段覆盖写，天然就是「整体替换」语义。**

**代价也要能说出来**（面试官很可能追问）：
- 不能对 JSON 内部字段建索引 → 查不了「所有联系人里有张三的合作方」
- 单列 `varchar(4000)` 有上限，联系人多到一定程度会爆（所以实体上挂了 `@Size(max=4000)` 校验，超了直接拒绝并提示「联系人信息过多」）
- 数据库里看不出结构约束，JSON 格式靠应用层自觉

> 🎯 **面试点（这道题很值钱）**：面试官问「一对多怎么设计？」——**不要只答「开子表」**。标准答法是先问需求：
> - 需要**单独查子表数据**吗？（要 → 子表）
> - 子表数据**有自己的生命周期/权限/状态**吗？（要 → 子表）
> - 只是**跟着主表一起读写的附属信息**？（→ JSON 列更省事）
>
> 然后举这个项目的例子：合作方的联系人不被单独查询、不被外部引用，所以答案册选了 JSON 列；而**合同和它的相对方**（`contract_draft_partnerids`）是独立的表，因为相对方要被单独查询和关联。**同一个系统里两种做法并存**——这说明选型是跟着需求走的，不是一刀切。

> ⚠️ **`JsonUtil` 是 `com.ruoyi.conf.util.JsonUtil`**（本模块自己的），不是 Hutool 也不是 Jackson 的工具类。复制时别漏它，否则 Controller 编译不过。

---

## T4 · 编号规则设置（有算法的一个）

🎯 **目标**：配置「合同编号怎么生成」，然后**新建合同时能自动拿到编号**。

🖥 **前端入口**：**合同基础配置 → 编号规则设置**，路由 `/conf/billRule`，页面 `views/conf/billRule/index.vue`，API `api/conf/billRule.js`

🔌 **接口入口**

| 方法 | URL | 权限码 | 用途 |
|---|---|---|---|
| GET | `/conf/billRule/list` | `conf:billRule:list` | 列表 |
| GET | `/conf/billRule/{id}` | `conf:billRule:query` | 详情 |
| POST | `/conf/billRule` | `conf:billRule:add` | 新增 |
| PUT | `/conf/billRule` | `conf:billRule:edit` | 修改 |
| DELETE | `/conf/billRule/{ids}` | `conf:billRule:remove` | 删除 |
| **GET** | **`/conf/billRule/useBillNumber/{enCode}`** | — | **占用一个编号**（真正发号） |
| **GET** | **`/conf/billRule/getBillNumber/{enCode}`** | — | **预览下一个编号**（不发号） |

📋 **复制清单**

| 答案册文件 | 说明 |
|---|---|
| `domain/ConfBillRule.java` | 主表实体 |
| `domain/ConfBillRuleExt.java` | 扩展表（规则的明细段） |
| `domain/bo/ConfBillRuleBo.java` · `ConfBillRuleExtBo.java` | 入参 |
| `domain/vo/ConfBillRuleVo.java` · `ConfBillRuleExtVo.java` | 出参 |
| `mapper/ConfBillRuleMapper.java` · `ConfBillRuleExtMapper.java` | Mapper |
| `service/IConfBillRuleService.java` · `IConfBillRuleExtService.java` + `impl/` | Service |
| `controller/ConfBillRuleController.java` | Controller |
| `util/PadUtil.java` | **补零工具**（编号生成的核心） |
| 前端 `api/conf/billRule.js` · `views/conf/billRule/index.vue` | 前端 |

**数据库**：`conf_bill_rule` + `conf_bill_rule_ext`

🧠 **原理 · 编号是怎么生成的**

一个编号规则被拆成**若干段**存在 `conf_bill_rule_ext` 里，每段有类型（固定前缀 / 日期 / 流水号）和长度。发号时：

```
useBillNumber("HT")
  → 查 ext 明细，按顺序拼各段
     固定段 → 直接取字面值（如 "HT"）
     日期段 → 取当前日期按格式（如 "20260926"）
     流水号段 → 取当前计数 +1，再 PadUtil 补零到指定长度（如 0001）
  → 更新计数（这一步要保证并发安全）
  → 返回完整编号字符串
```

**`useBillNumber` vs `getBillNumber` 的区别就是「占不占号」**——预览不占号，所以可以无限次调用；`useBillNumber` 会推进计数，所以**只应该在建单据时调一次**。
> `PadUtil` 那种「左边补零」的工具类，Java 标准库也有（`String.format("%04d", n)`），可以对比着看它为什么自己写了一个。

---

## T5 · 合同类型配置（在 zxcm-form 模块里）

> 🔧 **照着敲的施工单**：[M1-施工单.md](施工单/M1-施工单.md)
> （本节是**总览**：T5 的接口、表、前端页面、清单一站式都在施工单里，含 **M1 只能验收「列表/启停/删除」** 的原因）

🎯 **目标**：配「有哪些合同类型」（采购合同 / 销售合同 / 知识产权合同……），每个类型可以挂一个自定义表单。**这一步的真正目的是给 T6 的合同草拟提供「合同类型」下拉框**。

🖥 **前端入口**：**合同基础配置 → 合同类型配置**
- 路由：`/conf/form`
- 页面：`views/system/form/index.vue`（**注意：页面在 `system/` 目录下，但菜单挂在「合同基础配置」里**）
- API：`api/system/form.js`

🔌 **接口入口** —— ⚠️ **这里有两个不同的接口，别搞混**（这是本篇最容易踩的坑）：

**① 配置页面用的**（`FormController`，在 **`zxcm-form`** 模块，`@RequestMapping("/system/form")`）

| 方法 | URL | 权限码 |
|---|---|---|
| GET | `/system/form/list` | `system:form:list`（**代码里被注释掉了**，见下方原理） |
| GET | `/system/form/{id}` | `system:form:query` |
| POST | `/system/form` | `system:form:add` |
| PUT | `/system/form` | `system:form:edit` |
| PUT | `/system/form/changeStatus` | `system:form:edit` |
| DELETE | `/system/form/{ids}` | `system:form:remove` |
| POST | `/system/form/export` | `system:form:export` |

**② 下拉框用的**（`TreeSelectorController`，在 **`zxcm-contract`** 模块的 `conf/controller/`）

| 方法 | URL | 说明 |
|---|---|---|
| GET | `/conf/treeSelect/contractTypeSelect` | 合同类型树，带 `?status=0` 只取启用的 |

📌 **为什么合同草拟（T6）选类型时走的是 ②，不是 ①**：因为 ① 是「管理合同类型配置」的接口，要 `system:form:*` 权限；而选类型的用户（普通业务员）**没有配置权限**。所以答案册另开了一个只读的下拉接口，**不带配置权限**。
> 全项目有 **10+ 个页面**在调 `api/conf/contractType.js`（合同草拟、模板草稿箱、模板发布、审签跟踪、合同报表……）。这也是为什么它被放进 `api/conf/` 而不是 `api/system/`——它是**被业务复用的公共下拉**。

📋 **复制清单**

**后端**（`zxcm-form/src/main/java/com/ruoyi/form/` → 本项目同路径，**只有 7 个文件，整个复制**）

| 文件 | 说明 |
|---|---|
| `domain/Form.java` | 实体（`@TableName("form")`） |
| `domain/bo/FormBo.java` | 入参（**注意：`zxcm-contract` 也在用这个类**） |
| `domain/vo/FormVo.java` | 出参 |
| `mapper/FormMapper.java` | Mapper |
| `service/IFormService.java` + `impl/FormServiceImpl.java` | Service |
| `controller/FormController.java` | Controller |

**后端 · 别漏了另外两个跨模块文件**（在 `zxcm-contract` 模块里，T6 复制合同模块时会一起带过来，但**它们其实是 T5 的依赖**）：

| 文件 | 为什么算 T5 的 |
|---|---|
| `conf/controller/TreeSelectorController.java` | ②号接口在这里（`:241` 的 `contractTypeSelect`），**它 import 了 `zxcm-form` 的 `FormBo`** |
| `conf/util/TreeSelectModel.java`（或其所在包） | 下拉树的数据结构 |

**前端**

| 答案册文件 | 目标 |
|---|---|
| `zxcm-ui/src/api/system/form.js` | 配置页面用 |
| `zxcm-ui/src/views/system/form/index.vue` | 配置页面 |
| `zxcm-ui/src/api/conf/contractType.js` | 下拉框用（**被 10+ 个页面共享**） |

**数据库**：`form` 表 —— ⚠️ **M0 的底座脚本没有它，而且它有 56 条初始数据（合同类型清单），不导的话合同草拟的类型下拉是空的、T6 直接卡住**。

我已经把建表语句 + 56 条初始数据提取好了：

```bash
cd "E:/Develop/project/zxcm-learn/script/sql"
mysql --no-defaults --ssl-mode=DISABLED -h 192.168.56.10 -u root -proot zxcm \
  --default-character-set=utf8mb4 < 03-missing-base-tables.sql
# 这个脚本同时补了 m2 需要的 act_designer，M2 之前不用再跑一遍
```

✅ **跑通验证**

```sql
SELECT COUNT(*) FROM form;                    -- 预期 56
SELECT NAME, CODE, PARENT_ID, INSTANCE_DEF_KEY FROM form ORDER BY SORT LIMIT 10;
```

然后浏览器里：**合同类型配置**页面能进、树能出数 → 新增一个类型 → **回「合同草拟」新建页面，看类型下拉框里有没有它**（这是 T6 的前置，没出现就别往下走）。

🧠 **原理 · 三个值得讲的点**

**1. 为什么一个业务模块会独立成 `zxcm-form`**

`M0` 时的模块清单里 `zxcm-form` 是独立模块，说明它**不依赖合同**，是可以被别的业务复用的「动态表单引擎」。合同类型只是它的第一个使用方。**依赖方向是 `contract → form`，不是反过来**——证据就是 `zxcm-contract` 的 `TreeSelectorController` 能 import `zxcm-form` 的 `FormBo`，而 `zxcm-form` 里没有任何 import 合同的代码。
> 面试点：**模块划分的依据是「依赖方向」而不是「业务归属」**。表单引擎不 import 合同，所以能独立；合同要 import 表单，所以合同在上层。这个判断力比「我按业务分了模块」值钱。

**2. `form` 表是棵树，不是平表**

`form` 表有 `PARENT_ID` 字段——和 `sys_menu`、`sys_dept` 一样是**自关联树**。所以你会看到 56 条里有**重名的类型**（「设备采购」出现 3 次、「软件采购」3 次、「技术服务」2 次）——它们在不同的大类下面。这也解释了为什么下拉接口叫 `contractTypeSelect` 且返回的是**树**（`List<TreeSelectModel>`），不是平铺列表。

**3. `list` 接口的权限注解为什么被注释掉**

`FormController` 第 71 行：
```java
//@SaCheckPermission("system:form:list")     ← 被注释掉了
@GetMapping("/list")
public R<List<FormVo>> list(FormBo bo) {
```

因为 `/system/form/list` 这个接口**同时被配置页面和业务下拉框调用**，一旦加上 `system:form:list` 权限，**所有没有「合同类型配置」权限的业务员都会选不到合同类型**。

> 🎯 **面试点（这是权限设计的经典两难）**：一个接口被两类用户用，权限该按谁定？
> - 按**最严的**（配置权限）→ 业务功能全废
> - 按**最松的**（谁都能调）→ 配置数据暴露
>
> 答案册的解法是**拆接口**：另开了 `/conf/treeSelect/contractTypeSelect` 给业务方用（只读、带 `status` 过滤），配置页继续用 `/system/form/*`，然后**把 `/system/form/list` 的权限暂时注释掉**（属于妥协，不是干净的解法）。
> **能说出「这里其实有更好的做法」比复述代码加分**：正经做法是让 `/system/form/list` 保持严格权限，所有业务方一律改走 treeSelect 接口——**注释掉注解是技术债**。

> ⚠️ **M1 范围提醒**：动态表单引擎本身很复杂（表单元数据、渲染、校验）。**M1 只需要它「能配出合同类型下拉框」**，深挖留到 M3。

---

## T6 · 合同草拟（M1 的重头戏）

> 🔧 **照着敲的施工单**：[M1-施工单.md](施工单/M1-施工单.md)
> （本节是**总览**：T6 的接口、表、前端页面、清单一站式都在施工单里，含 **「T6 是编译汇聚点」** —— `contract` 包要连兄弟包一起拷，否则编译不过）

🎯 **目标**：合同主表的 CRUD。**这是整个系统的心脏**——`contract_draft` 表是全库字段最多的表，M2/M3/M4 都在它上面长出来。

🖥 **前端入口**
- 菜单：**合同管理 → 合同草拟**
- 路由：`/contract/contractDraft`
- 页面目录：`views/contract/contractDraft/`（**7 个文件、3854 行**）

| 文件 | 职责 |
|---|---|
| `index.vue` | 草拟列表（我的合同 / 关联合同） |
| `createContract.vue` | 新建合同主流程 |
| `edit.vue` | 编辑 |
| `apply.vue` | 提交审批申请 |
| `merchant.vue` | 相对方甲方乙方信息 |
| `flowShow.vue` | 流程展示 |
| `newVersion.vue` | 创建新版本 |
| `info/index.vue` · `info/editInfo.vue` | 合同信息详情/编辑 |

🔌 **接口入口**（`ContractDraftController`）

| 方法 | URL | 权限码 | 用途 |
|---|---|---|---|
| GET | `/contract/contractDraft/list` | `contract:contractDraft:list` | 分页列表 |
| GET | `/contract/contractDraft/templateList` | `contract:contractDraft:list` | 选模板用 |
| GET | `/contract/contractDraft/{id}` | `contract:contractDraft:query` | 详情 |
| GET | `/contract/contractDraft/getContractByCode` | `contract:contractDraft:query` | 按编号查 |
| POST | `/contract/contractDraft` | `contract:contractDraft:add` | 新建 |
| PUT | `/contract/contractDraft` | `contract:contractDraft:edit` | 修改 |
| DELETE | `/contract/contractDraft/{id}` | `contract:contractDraft:remove` | 删除（**注意是单数 `{id}`，不是 `{ids}`**） |
| POST | `/contract/contractDraft/newVersion` | `contract:contractDraft:add` | 创建新版本 |
| GET | `/contract/contractDraft/myContractList` | — | 我的合同 |
| GET | `/contract/contractDraft/relatedContractList` | — | 关联合同 |

📋 **复制清单**

**后端**（`com/ruoyi/contract/`）

| 文件 | 说明 |
|---|---|
| `domain/ContractDraft.java` | **主实体（79 个字段，对应表 85 列）** |
| `domain/ContractDraftPartnerids.java` | 合同-相对方关联表 |
| `domain/ContractDraftFiles.java` | 合同文件 |
| `domain/ContractHis.java` / `ContractHisFiles.java` | 历史版本 |
| `domain/bo/ContractDraftBo.java` 等 7 个 BO | 入参 |
| `domain/vo/ContractDraftVo.java` 等 8 个 VO | 出参 |
| `mapper/ContractDraftMapper.java`（**222 行，16 个注解 SQL**） | 主 Mapper |
| `mapper/ContractDraftPartneridsMapper.java` · `ContractDraftFilesMapper.java` · `ContractHisMapper.java` · `ContractHisFilesMapper.java` · `ContractDraftStatMapper.java` | 其余 Mapper |
| `service/IContractDraftService.java` + `IContractDraftFilesService.java` + `impl/` | Service |
| `controller/ContractDraftController.java` · `ContractDraftFilesController.java` | Controller |
| `enums/ContractEnums.java` | **合同状态枚举（M2 状态机的地基）** |
| `model/ContractCollaborators.java` · `model/DocContent.java` | 模型 |

**前端**：`api/contract/contractDraft.js` · `api/contract/contractDraftFiles.js` + `views/contract/contractDraft/**` 全套

**数据库**：`contract_draft` · `contract_draft_partnerids` · `contract_draft_files` · `contract_his` · `contract_his_files`

> ⚠️ **`contract_draft` 有 85 列**（实体类 `ContractDraft.java` 只声明了 79 个字段——**表和实体字段数不必相等**：表可以有应用不用的历史列，实体也可以有表没有的 `@TableField(exist=false)` 字段）。
> 别被吓到——**M1 只需要让「新建一条、列表能查到、能改能删」跑通**，不需要把 85 个列全部用上。先让 `CONTRACT_NAME` / `CONTRACT_TYPE` / `CONTRACT_STATUS` 这几个跑通，其余字段是 M2-M4 逐个填的。

✅ **跑通验证**

| # | 怎么验 | 预期 |
|---|---|---|
| 1 接口通 | `doc.html` 调 `GET /contract/contractDraft/list` | `code:200` |
| 2 功能通 | 浏览器新建一份合同（选类型、填名称、选签约主体和相对方）→ 保存 → 列表出现 | 保存成功，列表刷新 |
| 3 数据对 | `SELECT ID, CONTRACT_CODE, CONTRACT_NAME, CONTRACT_STATUS, DEL_FLAG FROM contract_draft;` | 新记录在，`CONTRACT_STATUS='0'`（草稿），`DEL_FLAG='0'` |

🧠 **原理 · 这张表为什么这么宽**

`contract_draft` 一个表塞了 85 列，看着不合理，但看列名能看出**它承担了整条合同生命周期的状态**：

| 字段组 | 列名样例 | 属哪个里程碑 |
|---|---|---|
| 基本信息 | `CONTRACT_NAME` `CONTRACT_TYPE` `AMOUNT` `CURRENCY` | M1 |
| 主体/相对方 | `SUBJECTS` `PARTNERS` `SUBJECT_NAMES` `PARTNER_NAMES` `MERCHANT` | M1 |
| **审批状态** | `CONTRACT_APPROVE_STATUS` `CONTRACT_APPROVE_APPLY_TIME` `CURRENT_REVIEWER` | **M2** |
| 用印状态 | `CONTRACT_STAMPED_STATUS` `SEAL_FILES_ID` `CONTRACT_STAMPED_TIME` | M4 |
| 归档 | `CONTRACT_ARCHIVES_ID` `CONTRACT_ARCHIVES_TIME` | M4 |
| 纠纷 | `CONTRACT_DISPUTE_STATUS` `CONTRACT_DISPUTES_REASON` | M4 |
| 作废/续签 | `CONTRACT_CANCEL` `RENEWAL` `renewed_status` | M4 |
| 版本 | `FILE_GROUPS_ID` `FILE_GROUPS_VERSION` | M3 |

**结论：这是一张典型的「宽表 + 状态字段」设计**——用多个状态列（而不是一张流程表和一堆 join）表达「这合同走到哪一步了」。

**它的代价**：字段一多就没人知道哪个状态是权威的（`CONTRACT_STATUS` / `CONTRACT_APPROVE_STATUS` / `CONTRACT_STAMPED_STATUS` 三个都不一样）。**这正是 M2 要解决的**——M2 会引入 `contract_point_log`（事件日志）作为「到底发生了什么」的权威来源。

> 📌 **顺带说清主键（实测过，别猜）**：`contract_draft.ID` 的列类型是 `varchar(64)`，但实体上写的是
> ```java
> @TableId(value = "id")      // ← 没有 type 属性
> private String id;
> ```
> **没写 `type` 就等于用全局的 `idType: ASSIGN_ID`**（雪花号，19 位数字）——所以这一列**存的是雪花号，只是用 varchar 装的**。
>
> 「列是 varchar，注解又不写 type」这种组合在这个项目里很常见，也容易踩坑：
> - **主键类型搞错，insert 直接失败**（比如表是自增、实体却按雪花塞值）
> - **`varchar` 装雪花号是有意的**：留出「人可以手填一个有意义的 ID」的余地，也让 `contract_draft` 的 ID 和那些天然字符串主键的表（如 `conf_signing_subject`、`template_contract`）在类型上统一，跨表关联时不用做类型转换
>
> 复制实体类时，**第一个要核对的注解就是 `@TableId`**。

---

## 🎤 怎么对面试官讲（M1 口语版）

> 「M1 我把合同系统的基础数据和一个完整 CRUD 跑通了，主要想讲三个设计点。
> 第一，**一个 CRUD 模块固定 7 个文件**：Controller 收参鉴权、Service 写业务、Mapper 访问数据，再加 Entity / Bo / Vo 三种数据形状。Bo 是入参，靠 JSR-303 的**分组校验**做到『新增不校验 id、修改必须带 id』；Vo 是出参，还能放数据库里没有的**计算字段**。
> 第二，**一对多我没急着开子表**。合作方的联系人、签约主体的联系人和银行账号，都是存在一个 `varchar(4000)` 的 JSON 列里——因为它们只跟着主表整体读写、不需要单独查询，省一张表、更新天然是整体替换；代价是不能按联系人姓名检索。而**合同的相对方我用了真关联表**（`contract_draft_partnerids`），因为它要被反查『这个合作方参与了哪些合同』。同一个系统里两种做法并存，说明选型是跟着需求走的。
> 第三，**删除是逻辑删除**：`DELETE` 被 MyBatis-Plus 改写成 `UPDATE ... SET DEL_FLAG = 2`，因为历史合同引用了主体和合作方，物理删除会让合同数据悬空。
> 另外 M1 里我印象最深的是**权限的两层**：前端 `v-hasPermi` 只是把按钮藏起来（体验），后端 `@SaCheckPermission` 才是真防线——所以我专门建了个普通用户，用超管之外的账号验证过『按钮没了、直接打接口也照样 403』。」

---

## M1 验收

三条硬标准，全过才算完：

| # | 标准 | 怎么验 |
|---|---|---|
| 1 | **接口通** | `doc.html` 里 5 个模块的 `list` 接口全部 `code:200` |
| 2 | **功能通** | 浏览器里 5 个页面各走一遍：列表出数 → 新增 → 编辑 → 删除 |
| 3 | **数据对** | 每张表 SQL 查一遍，字段值和你点的一致（含 `DEL_FLAG` 逻辑删除） |

**加餐：建一个普通用户来验证权限**

M0 遗留的那件事——**用超管永远测不出权限配置对不对**（超管走 `*:*:*` 短路）。现在菜单和权限码都有了，可以：

1. 系统管理 → 用户管理 → 新建用户
2. 系统管理 → 角色管理 → 建一个角色，只勾「签约主体管理」的**查询**，不勾新增/修改/删除
3. 用新用户登录 → 签约主体管理页面**应该看得到列表、但新增/修改/删除按钮消失**

**为什么按钮会消失**：前端 `index.vue` 上的 `v-hasPermi="['conf:signingSubject:add']"` 指令，在渲染时检查当前用户的权限码集合，没有就把元素从 DOM 里删掉。**是「前端隐藏 + 后端拦截」双层**——前端隐藏只是体验，后端 `@SaCheckPermission` 才是真防线。
> ⚠️ **验证时必须双击确认这一条**：把按钮的 `v-hasPermi` 删掉（或者直接用 curl 打 `/conf/signingSubject` POST），**没有权限的用户应该仍然被后端拒绝**。这才是「权限真的生效」。
> 详细原理见 [D1 第二部分](D1-SaToken鉴权链路.md)（`@SaCheckPermission` 是 MVC 拦截器实现的，不是 AOP 切面）。

---

## M1 踩坑预警

| 症状 | 大概率原因 |
|---|---|
| 菜单里没有新页面 | 没重新登录（权限走 Redis 缓存） |
| 菜单有、点进去 404 | 后端 Controller 没复制过来，或**没重启后端** |
| 新增接口报 500，看 SQL 发现列名不对 | 实体字段名和表列名对不上（`mapUnderscoreToCamelCase` 转换后仍不匹配）→ 加 `@TableField("真实列名")` |
| 新增成功但主键是 null / 报主键冲突 | 实体 `@TableId` 的类型和表主键定义不匹配 |
| 删除后数据还在，但列表里没了 | **正常**。逻辑删除，`DEL_FLAG` 变成 2 |
| `Table 'zxcm.conf_signing_subject' doesn't exist` | 建表语句没补进库（M0 只建了 `sys_*`/`gen_*`） |
| `Table 'zxcm.CONF_SIGNING_SUBJECT' doesn't exist`（**大写**） | 从答案册拷来的注解 SQL 写了**大写表名**，而本机 MySQL `lower_case_table_names=0`（表名大小写敏感）。改成小写即可。**只在勾「长期有效」保存时触发** |
| 编译报 `找不到符号: ContractDraftVo` / `TContractCounterpartImportVo` | 拷来的文件里有**死 import**（只在 import 行出现一次、正文没用到），删掉即可（用 IDE 的 Optimize Imports） |
| 列表里每一行的联系人都一样、还是自己的名字 | **设计如此，不是 bug**：`ServiceImpl.queryPageList` 默认把「当前登录人」塞进每行的 `contacts`（`getInfo` 详情接口才返回真实联系人） |
| 「经营状态」下拉框是空的 | `sys_dict_data` 里没有 `BUSINESS_STATUS` 字典（底座脚本 `01` 带了，若手工建库容易漏） |
| 前端热更新了但页面还是旧的 | 菜单/权限相关**必须重新登录**；纯页面改动 Ctrl+F5 强刷 |
