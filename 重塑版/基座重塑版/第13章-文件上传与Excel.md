# 第 13 章 · 文件上传与 Excel 导入导出

> **本章讲两件"每个后台系统都要有、但很少有人讲透"的事**：**文件怎么进来、怎么出去**（上传/下载/预览），**Excel 怎么批量导入、怎么批量导出**。
>
> **本章第一次讲透**：① **文件存哪**——本项目**没有 OSS，就是存本地磁盘**（`RuoYiConfig.getProfile()`），**这意味着什么、有什么坑**；② **文件名为什么不能信用户**——`FileUploadUtils` 里头文件的"三重门"（大小、后缀、路径穿越）；③ **Excel 导入的"注解 + 监听器"机制**——`@ExcelProperty` 怎么和字典翻译（`@ExcelDictFormat`）联动，**为什么 10 万行 Excel 用 POI 会 OOM 而 EasyExcel 不会**；④ 几个**必须坦白的事实**：**`/profile/**` 是免登录的（谁拿到 URL 谁能看）**、**上传大小限制配了两套且不一致**、**文件路径写死在配置里**。

---

## 一、这部分是做什么的？

### 1.1 两块内容

| 块 | 解决什么 | 入口 | 核心类 |
|---|---|---|---|
| **文件上传下载** | 用户传头像/附件/合同扫描件；下载/预览 | `CommonController` | `FileUploadUtils`、`FileUtils`、`RuoYiConfig` |
| **Excel 导入导出** | 批量导入用户；导出报表 | `SysUserController` 等业务 Controller | `ExcelUtil`、`ExcelResult`、`DefaultExcelListener` |

### 1.1.1 零基础先补概念

| 概念 | 大白话 | 本项目里的真实例子 |
|---|---|---|
| **`MultipartFile`** | Spring 把"浏览器上传的那个文件"包装成一个对象：能拿文件名、大小、内容，也能"另存到磁盘" | 上传接口的参数就是它（`ruoyi-admin/src/main/java/com/ruoyi/web/controller/common/CommonController.java:100`） |
| **文件流 / 落盘** | 文件不是"一个字符串"，而是一串字节。服务端要把它从请求里读出来，写到服务器磁盘上（落盘） | `file.transferTo(desc)`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/FileUploadUtils.java:127`） |
| **UUID 重命名** | 不信任用户给的文件名，自己生成一个随机、唯一、只含安全字符的新名字 | `日期目录 + UUID + 后缀`（`FileUploadUtils.java:138`） |
| **SAX 流式解析** | 读 Excel 时"读一行、处理一行、丢一行"，不在内存里放整张表；对应的是"一次性全读进内存"（POI 的 DOM 方式） | EasyExcel 的 `doRead()` + 监听器（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/poi/ExcelUtil.java:61-64`） |

**一句话串起来**：上传 = 接住 `MultipartFile` → 校验 → 用 UUID 改名 → 写进磁盘；导入 = 用 SAX 一行行读 → 每行校验/收集 → 最后统一判定。

### 1.2 文件到底存在哪？（先建立物理认知）

**本项目没有接 OSS / MinIO / 云存储**——**文件就是存在"服务器本机的一块磁盘目录"里**。

（`ruoyi-admin/src/main/resources/application-dev.yml:4`）

```yaml
ruoyi:
  profile: D:\java project\document\Resources     # :4  ★文件根目录
```

**围绕这个根目录，派生了几个子目录**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/config/RuoYiConfig.java`）：

```java
public static String getProfile()            { return profile + File.separator + LoginHelper.getTenantId(); }  // :79-82  ★租户根目录
public static String getImportPath()         { return getProfile() + "/import"; }    // :92-95
public static String getAvatarPath()         { return getProfile() + "/avatar"; }    // :108-111
public static String getDownloadPath()       { return getProfile() + "/download/"; } // :116-119
public static String getUploadPath()         { return getProfile() + "/upload"; }    // :124-127  ★通用上传
public static String getTempPath()           { return getProfile() + File.separator + "temp"; }  // :132-135 临时目录
```

**注意 `getProfile()` 不是"配置里那一行原样返回"**：它会再拼上当前登录人的租户 ID（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/config/RuoYiConfig.java:79-82`），所以不同租户的文件天然分目录。**所以一个上传的文件最终落在**：

```
D:\java project\document\Resources\<租户ID>\upload\2026\10\17\<uuid>.png
└──────────── profile 配置 ───────────┘ └tenant┘ └upload─┘ └datePath┘ └── UUID 文件名 ──┘
```

**这个"存本地"的选择，是本章最重要的前提**——它决定了后面所有的**坑**（多台服务器文件不共享、备份麻烦、预览接口在免登录白名单里……），§4.13 会集中讲（注意：白名单不等于匿名一定可读，见那一节）。

### 1.3 Excel 导入导出是什么场景？

**两个典型用例**：

1. **批量导入**——管理员拿到一个"用户名单.xlsx"，**一键导入 500 个用户**（`SysUserController.java:129` 的 `/system/user/importData`）；
2. **批量导出**——把"用户列表"导出成 Excel 给领导（`SysUserController.java:108` 的 `/system/user/export`）。

**为什么不用 CSV？** 因为**业务表格要"中文表头、字典翻译、数据校验、错误定位到行列"**——CSV 太简陋，**Excel（.xlsx）才是国内办公场景的事实标准**。

---

## 二、站在架构师 / 开发经理角度：为什么这么设计？

### 2.1 为什么上传的文件名要用 UUID，不能用原始文件名？

**用户上传的文件名，一个字都不能信。** 看项目的做法（`FileUploadUtils.java:135`）：

```java
public static final String extractFilename(MultipartFile file) {
    String fileName = file.getOriginalFilename();                     // :136  拿到原始名
    String extension = getExtension(file);                            // :137  只留后缀
    fileName = DateUtils.datePath() + "/" + IdUtils.fastUUID() + "." + extension;   // :138  ★彻底不要原名
    return fileName;
}
```

**注意**：**原始文件名（`fileName`）拿到了，但第 138 行重新拼的时候——原始名被完全丢弃，只用它的"后缀"。** **新名字 = `日期路径 + UUID + 后缀`。**

**为什么这么狠？** 因为原始文件名能害你八种办法：

| 风险 | 例子 | 后果 |
|---|---|---|
| **路径穿越** | 文件名 `../../etc/passwd` | **覆盖系统文件**（经典漏洞） |
| **中文/特殊字符** | `合同 2024/10.xlsx` | 存盘失败或 URL 编码错乱 |
| **重名覆盖** | 两个人传了同名 `a.png` | **后传的覆盖先传的** |
| **XSS** | 文件名含 `<script>` | 预览页执行脚本 |
| **超长文件名** | 200 个字符 | 超过文件系统限制报错 |
| **可执行后缀** | `x.jsp` / `x.sh` | **结合"解析漏洞"可能被当代码执行** |
| **信息泄漏** | 文件名暴露业务信息 | —— |
| **不可控字符** | `\0` 截断 | 经典文件上传绕过 |

**UUID 一次性解决几乎所有问题**：**随机、唯一（不重名）、只含安全字符、长度固定、不可预测。** **代价是"原始文件名丢了"**——**所以文件名要单独存数据库**（本项目 `CommonController.uploadFile` 把 `realName`（原文件名）一起返回给前端，`:112`）。

**"用 UUID 重命名 + 原始名另存"是文件上传的标准姿势。**

### 2.2 为什么按日期分目录（`datePath`）？

**如果所有文件都堆在一个目录里**：一个运行三年的系统，**`upload` 目录下可能有几十万个文件**——**`ls` 卡死、备份慢、文件系统索引性能下降。**

**按 `yyyy/MM/dd` 分目录**（`DateUtils.datePath()`，`FileUploadUtils.java:138`）：

- **每个目录文件数量可控**（按天分流）；
- **好定位**（"那天的文件"直接去那天找）；
- **好归档/清理**（按目录批量删旧数据）。

**这是"用目录结构分担文件系统压力"的经典做法。**

### 2.3 为什么"大小限制"配了两层，而且不一致？

**本项目有两道大小限制**：

| 层级 | 位置 | 值 |
|---|---|---|
| **Servlet 层** | `application.yml:72` `spring.servlet.multipart.max-file-size: 80MB` | **80MB** |
| **代码层** | `FileUploadUtils.java:51` `DEFAULT_MAX_SIZE = 100 * 1024 * 1024` | **100MB** |

**为什么要两层？**

- **`max-file-size`（Servlet 层）**：**在"请求还没进 Controller"时就拦**——**Tomcat 直接拒绝，不占内存**（大文件传一半就被切断，最省资源）；
- **`DEFAULT_MAX_SIZE`（代码层）**：**在业务代码里再校验一次**——**给"不同类型文件不同上限"留口子**（比如头像限 2MB、附件限 100MB），**`assertAllowed` 支持传 `allowedExtension` 就能配套不同大小**。

**但本项目这两层"数值不一致"（80 vs 100）**（★坦白点）——**实际效果是"Servlet 层先拦，80MB 以上的文件根本到不了代码层"，所以代码层那个 100MB 形同虚设。** **这是个典型的"配置不同步"问题**（§4.12）。

### 2.4 为什么 Excel 用 EasyExcel，而不是 Apache POI？

**这是本章最"涨知识"的一个技术点。**

**POI 的工作方式**：**把整个 Excel 一次性读进内存**（`XSSFWorkbook`）。**行数、列数越多，占用内存越大（与整张表的内容成正比）**——**并发几个人同时导入，就可能 OOM（内存溢出）**。

**EasyExcel 的工作方式**：**SAX 流式解析、"读一行、处理一行、丢一行"**——**内存占用恒定（只有"当前行 + 表头"那么大），10 万行和 100 行吃一样的内存**。

**所以本项目的选择是"内存安全优先"**：`ExcelUtil` 里所有读写都走 EasyExcel（`ExcelUtil.java:49/63/94`）。**代价是"不能随机访问单元格"**（流式读不能回头）——**但对"逐行导入"这个场景，完全够用。**

### 2.5 为什么导入要"逐行校验 + 收集错误 + 一次性返回"？

**导入 Excel 最怕什么？** **导入到一半失败了，前面导进去的怎么办？**

**本项目的策略**：**"先全量校验、收集所有错误、一条都不写入、最后统一反馈"**（看 `DefaultExcelListener` + `DefautExcelResult`）。

**流程**：

```
逐行读 Excel（以默认监听器 DefaultExcelListener 为例）
   ├─ 每行做 Bean Validation（ValidatorUtils.validate，DefaultExcelListener.java:106）
   ├─ 校验失败 → onException 记进 errorList，并抛 ExcelAnalysisException（:94）→ 本次解析中止
   └─ 校验通过 → 加进 list（:108）
读完后：
   ├─ getAnalysis()（DefautExcelResult.java:73）生成回执
   └─ "全部成功，共N条" / 有错误时由上层拼错误详情
```

**注意两种监听器的行为不同**：默认监听器 `DefaultExcelListener.onException` 在记录错误后**会抛异常终止解析**（`:93-94`），所以它并不是"把整张表的错都收集完"；而用户导入用的 `SysUserImportListener` 在 `invoke` 内部用 `try/catch`（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/listener/SysUserImportListener.java:96-180`）把行级业务异常吞掉、继续下一行，才能做到"多行错误一起收集"。**说"逐行收集错误"时，要指明是哪个监听器。**

**为什么这么设计？** **因为"用户需要知道"哪里错了"以便改"**——**如果第一行错就抛异常返回，用户改完再传，第二行又错，来回十几次**。**"一次性把所有错误攒起来告诉用户"体验好得多。**

**代价**：**设计意图是"要么全成功、要么全失败（不能只导入对的部分）"**——**这是"数据一致性优先"的选择**（宁可让用户重传，也不留下一半脏数据）。

**⚠️ 但对当前用户导入代码要加一句澄清（★ 真实性校正）**：**实际实现并不是"一条都不写入"**。`SysUserImportListener.invoke` 在每一行校验通过后立刻调 `userService.insertUser(user)`（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/listener/SysUserImportListener.java:136`），**Controller 这一层没有用一个事务把整份 Excel 包起来**；它只是把行级错误 catch 住、记进 `failureMsg`（`:175-180`），最后 `getExcelResult()` 发现有失败才抛异常（`:251-263`）。所以：**成功行可能已经落库，失败行之后才报错**——这是当前实现的真实行为。要真正做到"全成功或全失败"，得把整个导入包进一个事务，或者先全部校验、再统一入库。**答辩时请按真实行为说，别把设计意图说成已实现。**

---

## 三、需要哪些技术？

| 技术点 | 是什么 | 在本项目哪里用到 | 关键类/方法 |
|---|---|---|---|
| **`MultipartFile`** | Spring 对上传文件的抽象 | 接收上传 | `CommonController.java:100` |
| **`spring.servlet.multipart.max-file-size`** | Servlet 层上传大小限制 | 上传限制第一道 | `application.yml:72` |
| **UUID 文件名** | 用随机唯一名替换原始名 | 防穿越/重名/XSS | `FileUploadUtils.java:138` |
| **日期目录** | 按 `yyyy/MM/dd` 分层存文件 | 分摊目录压力 | `DateUtils.datePath()` |
| **后缀白名单** | 只允许指定扩展名 | 防上传可执行文件 | `MimeTypeUtils.DEFAULT_ALLOWED_EXTENSION` `:42` |
| **路径穿越防护** | 去掉 `..` 防止目录逃逸 | 预览接口 | `CommonController.java:143` |
| **`HttpServletResponse` 流式写出** | 把文件字节写进响应 | 下载/预览 | `FileUtils.writeBytes` `:54` |
| **`Content-Disposition` 附件头** | 告诉浏览器"这是下载" | 下载接口 | `FileUtils.setAttachmentResponseHeader` `:219` |
| **EasyExcel** | 流式（SAX）读写 Excel | 导入导出 | `ExcelUtil.java:49/63/94` |
| **`@ExcelProperty`** | 声明"字段对应哪一列表头" | 导入导出 VO | `SysUserImportVo.java` |
| **`@ExcelDictFormat` + `ExcelDictConvert`** | 导入导出时自动做字典翻译 | 状态/性别等码值 | `ExcelDictConvert.java` |
| **EasyExcel `AnalysisEventListener`** | 逐行回调的监听器 | 导入处理 | `DefaultExcelListener.java`、`SysUserImportListener` |
| **Bean Validation** | 逐行数据校验 | 导入校验 | `ValidatorUtils.validate` `:106` |
| **SPI（`DictService`）** | common 定义接口、system 实现 | 字典翻译解耦 | `DictService.java` |

---

## 四、功能具体怎么实现（带着你读懂）

### 4.1 上传入口：`CommonController.uploadFile`

（`ruoyi-admin/src/main/java/com/ruoyi/web/controller/common/CommonController.java`）

```java
@PostMapping("/common/upload")                                         // :99
public R<Map<String, Object>> uploadFile(MultipartFile file) throws Exception {   // :100
    try {
        String filePath = RuoYiConfig.getUploadPath();                 // :105  ★目标目录 = profile/upload
        String fileName = FileUploadUtils.upload(filePath, file);      // :107  ★真正干活的是它
        String url = serverConfig.getUrl() + fileName;                 // :108  ★拼成可访问 URL
        Map<String,Object> result = new HashMap<String,Object>();
        result.put("fileName", fileName);                              //       存盘名（UUID 那个）
        result.put("uid", IdUtil.fastSimpleUUID());                    //       给前端组件用的唯一 id
        result.put("realName", file.getOriginalFilename());            // :112  ★原始文件名也返回
        result.put("url", url);                                        //       可直接访问的地址
        return R.ok(result);
    } catch (Exception e) {
        return R.fail(e.getMessage());                                 // :118  ★注意：这里 catch 了
    }
}
```

**读懂三个点**：

1. **`serverConfig.getUrl()`（`:108`）**：**把"存盘路径"拼成"外网可访问 URL"**——比如存盘是 `upload/2026/10/17/abc.png`，返回的 `url` 是 `http://你的域名:8080/profile/upload/2026/10/17/abc.png`。**前端拿这个 url 就能显示图片。**
2. **返回四个东西**（`:110-113`）：**`fileName`（存盘名）、`uid`（随机 id，前端上传组件用）、`realName`（原文件名，给人看）、`url`（访问地址）**——**"一个上传接口，满足前端组件的所有需求"。**
3. **`catch (Exception e) { return R.fail(e.getMessage()) }`（`:116-118`）**：**这里把异常 catch 掉转成 `R.fail`**——**这跟第 11 章"全局异常处理器"的风格不同**。**好处是错误信息能透传给前端；坏处是"异常没被记为系统异常"**（对排查不利）。**这是本项目里比较少见的"自己 catch"写法。**

### 4.2 上传核心三步：`FileUploadUtils.upload`

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/FileUploadUtils.java`）

```java
public static final String upload(String baseDir, MultipartFile file, String[] allowedExtension)
        throws ... {
    int fileNamelength = file.getOriginalFilename().length();
    if (fileNamelength > FileUploadUtils.DEFAULT_FILE_NAME_LENGTH) {          // :118  ★① 文件名长度
        throw new FileNameLengthLimitExceededException(...);
    }

    assertAllowed(file, allowedExtension);                                    // :122  ★② 大小 + 后缀校验

    String fileName = extractFilename(file);                                  // :124  ★③ 生成安全文件名
    File desc = getAbsoluteFile(baseDir, fileName);                           // :126  确保目录存在
    file.transferTo(desc);                                                    // :127  ★④ 落盘
    String pathFileName = getPathFileName(baseDir, fileName);                 // :128  拼成对外返回的路径
    return pathFileName;                                                      // :129
}
```

**逐行读 `upload` 方法**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/FileUploadUtils.java:114-130`）：

| 行号 | 这一行在干什么 | 为什么这么写 |
|---|---|---|
| `:114` | 方法签名：基目录 + 文件 + 允许的后缀白名单 | 把"存哪、存什么、允许什么"全部显式传进来，工具类不自己猜 |
| `:117-118` | 取原始文件名长度，超过 `DEFAULT_FILE_NAME_LENGTH`（`:56`，值为 100）就抛异常 | 超长文件名会撞文件系统限制；**先校验再落盘，避免写到一半失败** |
| `:122` | `assertAllowed(file, allowedExtension)` | 统一做"大小 + 后缀"两道校验（`:168-195`），调用方不用重复写 |
| `:124` | `extractFilename(file)` | 真正生成安全文件名：丢弃原始名，只保留后缀（`:135-139`） |
| `:126` | `getAbsoluteFile(baseDir, fileName)` | 把"目录 + 文件名"拼成 `File`，父目录不存在就 `mkdirs()`（`:142-151`） |
| `:127` | `file.transferTo(desc)` | 把上传内容写进磁盘。**注意：这一步失败时没有清理半截文件**（§4.13 提醒） |
| `:128` | `getPathFileName(baseDir, fileName)` | 把磁盘绝对路径转成对外的 `/profile/<upload 相对路径>`（`:153-158`） |
| `:129` | 返回路径 | Controller 拿它去拼完整 URL（`CommonController.java:108`） |

**四步就是上传的全部**：**校验文件名长度 → 校验大小和后缀 → 生成安全文件名 → 确保目录存在并落盘。**

**`assertAllowed`（`:168`）——"大小 + 后缀"双校验**：

```java
public static final void assertAllowed(MultipartFile file, String[] allowedExtension) throws ... {
    long size = file.getSize();
    if (DEFAULT_MAX_SIZE != -1 && size > DEFAULT_MAX_SIZE) {                  // :171  超过 100MB 拒绝
        throw new FileSizeLimitExceededException(DEFAULT_MAX_SIZE / 1024 / 1024);
    }

    String fileName = file.getOriginalFilename();
    String extension = getExtension(file);
    if (allowedExtension != null && !isAllowedExtension(extension, allowedExtension)) {   // :177  后缀不在白名单
        // :178-192  按类型抛不同的异常（图片/Flash/媒体/视频/通用）
        throw new InvalidExtensionException...;
    }
}
```

**注意 `DEFAULT_MAX_SIZE != -1`（`:171`）**——**这是个"开关"：把 `DEFAULT_MAX_SIZE` 设成 `-1` 就"不限制大小"**。**这种"用魔法值当开关"的写法不优雅，但常见**（更好的做法是配个 boolean）。

**`isAllowedExtension`（`:204`）** 的逻辑简单直白——**遍历白名单，`equalsIgnoreCase` 匹配**：

```java
public static final boolean isAllowedExtension(String extension, String[] allowedExtension) {   // :204
    for (String str : allowedExtension) {
        if (str.equalsIgnoreCase(extension)) {                                // :206  ★忽略大小写
            return true;
        }
    }
    return false;
}
```

**白名单是什么？** 看 `MimeTypeUtils`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/MimeTypeUtils.java:21`）：

```java
public static final String[] IMAGE_EXTENSION = { "bmp", "gif", "jpg", "jpeg", "png" };     // :33
public static final String[] FLASH_EXTENSION = { "swf", "flv" };                          // :35
public static final String[] MEDIA_EXTENSION = { "swf", "flv", "mp3", "wav", ... };        // :37
public static final String[] VIDEO_EXTENSION = { "mp4", "avi", "rmvb" };                   // :40
public static final String[] DEFAULT_ALLOWED_EXTENSION = { ... };                          // :42  ★默认白名单
```

**关键安全认知**：**"白名单"（只允许列出的）永远比"黑名单"（禁止列出的）安全**——**因为黑名单永远列不全**（黑客总能找到你没禁的诡异后缀）。**本项目用的是白名单。**

### 4.3 后缀怎么取？`getExtension` 的"双保险"

```java
public static final String getExtension(MultipartFile file) {                 // :219
    String extension = FilenameUtils.getExtension(file.getOriginalFilename()); // :220  ★优先取文件名后缀
    if (StringUtils.isEmpty(extension)) {
        extension = MimeTypeUtils.getExtension(file.getContentType());         // :222  ★没有就用 MIME 类型推
    }
    return extension;
}
```

**为什么要"双保险"？** 因为**有些客户端上传的文件，`originalFilename` 可能没有后缀**（比如某些 App 上传的流）。**这时退而求其次，从 `Content-Type`（MIME 类型）反推后缀**。

**⚠️ 一个安全隐患**：**这个 `extension` 是"从文件名或 MIME 猜的"，而 MIME 类型是客户端可以随便伪造的**——**攻击者能把 `a.jsp` 的 MIME 伪造成 `image/png`**。**不过本项目还有第二道——白名单校验（`assertAllowed`），且存盘时用的是"重新拼的 UUID 名"**，**攻击者很难同时绕过"后缀白名单 + UUID 重命名"**。

### 4.4 路径穿越防护：`/profile/**` 预览接口

（`CommonController.java:126`）

```java
@GetMapping(value = "/profile/**")                                     // :125  ★通配符匹配
public void view(HttpServletRequest request, HttpServletResponse response) {
    String imgPath = extractPathFromPattern(request);                  // :128  取出 URL 里 /profile/ 之后的部分
    if (StringUtils.isEmpty(imgPath) || imgPath == "null") { return; }
    String ext = FileNameUtil.extName(imgPath);

    if (!Arrays.asList(MimeTypeUtils.IMAGE_EXTENSION).contains(ext)) {  // :135  ★只允许图片后缀
        return;
    }

    try {
        imgPath = imgPath.replace("..", "").replace("../", "");         // :143  ★★ 路径穿越防护
        ...
        String filePath = RuoYiConfig.getProfile() + File.separator + imgPath;   // :147
        response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE);
        FileUtils.writeBytes(filePath, response.getOutputStream());      // :150  ★写出去
    } catch (IOException e) {
        log.error("预览文件失败", e);
        response.setStatus(404);                                        // :153
    }
}
```

**两道防护，都很关键**：

1. **`:135` 只允许图片后缀**——**这个接口叫"图片预览"，就只放图片**。**防止有人拿它当"任意文件下载器"**（下载数据库备份、配置文件等）。
2. **`:143` `replace("..", "")`**——**去掉路径里的 `..`**。**如果不做这一步，攻击者请求 `/profile/../../../../etc/passwd` 就能读取系统文件**（经典目录穿越漏洞）。

**⚠️ 坦白点（安全性）**：**这种 `replace("..", "")` 是"黑名单式"防护，理论上不够严密**——**比如 `....//` 这种绕过**（虽然 `.replace("..","")` 对 `....//` 会得到 `//`，看起来是安全的）。**更稳妥的做法是"规范化路径后判断是否还在允许目录内"（`Path.normalize()` + `startsWith`）**。**本项目用的是"简单粗暴版"，实践中够用但不是最优解。**

### 4.5 下载：`checkAllowDownload` + 附件头

```java
@PostMapping("/common/download/resource")                              // :188
public void resourceDownload(String resource, HttpServletRequest request, HttpServletResponse response) {
    try {
        if (!FileUtils.checkAllowDownload(resource)) {                 // :194  ★★ 白名单校验
            throw new Exception(StringUtils.format("资源文件({})非法，不允许下载。", resource));
        }
        String localPath = RuoYiConfig.getProfile();
        String downloadPath = localPath + StringUtils.substringAfter(resource, Constants.RESOURCE_PREFIX);   // :201
        String downloadName = StringUtils.substringAfterLast(downloadPath, "/");   // :203
        response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE);         // :204
        FileUtils.setAttachmentResponseHeader(response, downloadName);            // :205  ★下载头
        FileUtils.writeBytes(downloadPath, response.getOutputStream());           // :206
    } catch (Exception e) {
        log.error("下载文件失败", e);
    }
}
```

**`checkAllowDownload`（`FileUtils.java:159`）** 又是白名单：

```java
public static boolean checkAllowDownload(String resource) {                     // :159
    // 禁止目录上跳级别
    ...
    if (ArrayUtils.contains(MimeTypeUtils.DEFAULT_ALLOWED_EXTENSION, FileTypeUtils.getFileType(resource))) {   // :168
        return true;
    }
    return false;
}
```

**`setAttachmentResponseHeader`（`FileUtils.java:219`）** 设置 `Content-Disposition: attachment; filename=<编码后的文件名>`——**告诉浏览器"这是个附件，下载它，别在页面里打开"**。**下载（attachment）和预览（inline）的区别就在这个头。**

### 4.6 Excel 导出：一行搞定

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/poi/ExcelUtil.java`）

```java
public static <T> void exportExcel(List<T> list, String sheetName, Class<T> clazz,
                                   HttpServletResponse response) {              // :87
    try {
        String filename = encodingFilename(sheetName);
        response.reset();
        FileUtils.setAttachmentResponseHeader(response, filename);              // ★附件头
        response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=UTF-8");
        ServletOutputStream os = response.getOutputStream();                    // ★直接写响应流
        EasyExcel.write(os, clazz)                                              // :94  ★EasyExcel
            .autoCloseStream(false)
            .registerWriteHandler(new LongestMatchColumnWidthStyleStrategy())   // :97  ★自动列宽
            .registerConverter(new ExcelBigNumberConvert())                     // :99  ★大数值防失真
            .sheet(sheetName).doWrite(list);
    } catch (IOException e) {
        throw new RuntimeException("导出Excel异常");                             // :102  ★统一异常
    }
}
```

**业务 Controller 里就一行**（`SysUserController.java:119`）：

```java
ExcelUtil.exportExcel(listVo, "用户数据", SysUserExportVo.class, response);
```

**两个"贴心"的注册**：

- **`LongestMatchColumnWidthStyleStrategy`（`:97`）**：**自动适配列宽**——**不注册的话，导出的 Excel 列宽是默认的，"用户名"可能显示不全**。**这是"让导出结果好看"的细节。**
- **`ExcelBigNumberConvert`（`:99`）**：**大数值防失真**——**Excel 对超过 15 位的数字会变成科学计数法（`1.23E+15`）**，**比如"身份证号""订单号"导出来就废了**。**这个转换器让大数字按字符串输出，不失真**——**这是国内业务场景的"血泪经验"。**

### 4.7 Excel 导入：三个重载

```java
// ① 同步导入（最简单，适合小数据量）
public static <T> List<T> importExcel(InputStream is, Class<T> clazz) {         // :48
    return EasyExcel.read(is).head(clazz).autoCloseStream(false).sheet().doReadSync();
}

// ② 带校验的导入（用默认监听器，返回 ExcelResult）
public static <T> ExcelResult<T> importExcel(InputStream is, Class<T> clazz, boolean isValidate) {  // :61
    DefaultExcelListener<T> listener = new DefaultExcelListener<>(isValidate);
    EasyExcel.read(is, clazz, listener).sheet().doRead();                       // :63  ★doRead 异步式
    return listener.getExcelResult();
}

// ③ 自定义监听器导入（业务逻辑最灵活）
public static <T> ExcelResult<T> importExcel(InputStream is, Class<T> clazz, ExcelListener<T> listener) {  // :75
    EasyExcel.read(is, clazz, listener).sheet().doRead();
    return listener.getExcelResult();
}
```

**三个重载的取舍**：

| 重载 | 返回 | 适合 |
|---|---|---|
| ① `importExcel(is, clazz)` | `List<T>` | **简单场景**（不用错误收集） |
| ② `importExcel(is, clazz, isValidate)` | `ExcelResult<T>` | **要校验 + 要错误报告** |
| ③ `importExcel(is, clazz, listener)` | `ExcelResult<T>` | **要自定义业务校验**（本项目用户导入用的这个） |

**注意 `doReadSync()` vs `doRead()`**：**前者"同步返回所有数据"（一次性拿到 List），后者"逐行回调监听器"**。**本项目用户导入走的是"监听器"路线**（因为要边读边校验、边收集错误）。

### 4.8 注解驱动：`@ExcelProperty` + `@ExcelDictFormat`

**导入导出的"字段 ↔ 表头"映射，靠注解声明。** 看导入 VO（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/domain/vo/SysUserImportVo.java`）：

```java
@Data
public class SysUserImportVo implements Serializable {

    @ExcelProperty(value = "部门名称")                              // :38  ★这一列对应"部门名称"表头
    private String deptPath;

    @ExcelProperty(value = "用户账号")                             // :44
    private String userName;

    @ExcelProperty(value = "用户性别", converter = ExcelDictConvert.class)    // :68  ★用字典转换器
    @ExcelDictFormat(dictType = "sys_user_sex")                               // :69  ★性别字典
    private String sex;

    @ExcelProperty(value = "帐号状态", converter = ExcelDictConvert.class)    // :75
    @ExcelDictFormat(dictType = "sys_normal_disable")                         // :76  ★状态字典
    private String status;
    ...
}
```

**精髓在"字典翻译"**：**Excel 里用户填的是"男/女"、"正常/停用"（人类看得懂的标签），但数据库存的是"0/1/2"（码值）。** **`ExcelDictConvert` 负责这个翻译**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/convert/ExcelDictConvert.java:41`）：

```java
// 导入：Excel 标签 → 数据库码值
@Override
public Object convertToJavaData(ReadCellData<?> cellData, ExcelContentProperty contentProperty, ...) {   // :54
    ExcelDictFormat anno = getAnnotation(contentProperty.getField());            // :56  ★读到字段上的注解
    String type = anno.dictType();                                              // :57  "sys_user_sex"
    String label = cellData.getStringValue();                                   // :58  Excel 里的 "男"
    if (StringUtils.isBlank(type)) {
        value = ExcelUtil.reverseByExp(label, anno.readConverterExp(), anno.separator());   // :61  用"表达式"翻译
    } else {
        value = SpringUtils.getBean(DictService.class).getDictValue(type, label, ...);      // :63  ★用字典翻译 "男"→"0"
    }
    return Convert.convert(contentProperty.getField().getType(), value);        // :65
}

// 导出：数据库码值 → Excel 标签
@Override
public WriteCellData<String> convertToExcelData(Object object, ExcelContentProperty contentProperty, ...) {   // :72
    ...
    label = SpringUtils.getBean(DictService.class).getDictLabel(type, value, anno.separator());  // :83  "0"→"男"
    return new WriteCellData<>(label);                                           // :85
}
```

**读懂这个设计的三个层次**：

1. **注解 `@ExcelDictFormat(dictType = "sys_user_sex")`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/annotation/ExcelDictFormat.java`）**：**声明"这个字段用哪个字典翻译"**（`dictType`）、或者**直接用表达式**（`readConverterExp = "0=男,1=女"`，就不用查字典了）、**以及分隔符**（`separator`，用于"多值字段"如"角色1,角色2"）。
2. **转换器 `ExcelDictConvert`**：**EasyExcel 的 `Converter` 接口**——**导入时 `convertToJavaData`（标签→码值），导出时 `convertToExcelData`（码值→标签）**。**注意它用 `SpringUtils.getBean(DictService.class)` 拿字典服务**（又是依赖倒置——common 不能依赖 system）。
3. **SPI `DictService`**：**common 定义接口**（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/core/service/DictService.java:21`，`getDictLabel` / `getDictValue`），**system 实现**（`SysDictTypeServiceImpl.java:48` `implements ISysDictTypeService, DictService`）。**所以 common 里的转换器能"调"到 system 里的字典逻辑，而不产生编译依赖。**

**核心理念**：**"一个注解声明 What，转换器负责 How，SPI 解决 Where。"** ——**业务 VO 上只写一句 `@ExcelDictFormat(dictType="xxx")`，翻译逻辑全自动。**

### 4.9 `DefaultExcelListener`：逐行校验 + 错误定位

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/excel/DefaultExcelListener.java`）

```java
public class DefaultExcelListener<T> extends AnalysisEventListener<T> implements ExcelListener<T> {

    private Boolean isValidate = Boolean.TRUE;
    private Map<Integer, String> headMap;                     // 表头（用来报错时定位列名）
    private ExcelResult<T> excelResult;

    @Override
    public void invoke(T data, AnalysisContext context) {                      // :104  ★每读一行回调一次
        if (isValidate) {
            ValidatorUtils.validate(data);                                     // :106  ★★ Bean Validation
        }
        excelResult.getList().add(data);                                       // :108  收集通过的数据
    }

    @Override
    public void onException(Exception exception, AnalysisContext context) throws Exception {   // :69
        String errMsg = null;
        if (exception instanceof ExcelDataConvertException) {                  // ★单元格类型转换错
            ExcelDataConvertException e = (ExcelDataConvertException) exception;
            Integer rowIndex = e.getRowIndex();
            Integer columnIndex = e.getColumnIndex();
            errMsg = StrUtil.format("第{}行-第{}列-表头{}: 解析异常<br/>",
                rowIndex + 1, columnIndex + 1, headMap.get(columnIndex));       // ★★ 定位到"行+列+表头名"
        }
        if (exception instanceof ConstraintViolationException) {                // ★Bean Validation 失败
            ...
            errMsg = StrUtil.format("第{}行数据校验异常: {}", ...);
        }
        excelResult.getErrorList().add(errMsg);                                 // 收集错误
        throw new ExcelAnalysisException(errMsg);                               // ★终止后续读（或由上层决定）
    }
}
```

**两个关键点**：

1. **`invoke`（`:104`）是"每读一行调一次"**——**这是 EasyExcel 流式的体现**：**读一行、校验一行、存一行、丢一行**，内存里永远只有当前行。**`ValidatorUtils.validate(data)`（`:106`）直接复用第 11 章讲的 Bean Validation**（`@NotBlank` 等注解）——**"校验规则写在 VO 上，导入时自动生效"。**
2. **`onException`（`:69`）能拿到"行列号"**——**`ExcelDataConvertException.getRowIndex()/getColumnIndex()`**，**再配合 `headMap`（表头）就能拼出"第 5 行-第 3 列-表头用户邮箱: 解析异常"**——**这种"精确到单元格"的错误提示，是用户体验的关键。**

### 4.10 `ExcelResult`：为什么要"结果对象"而不是"直接返回 List"？

（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/excel/ExcelResult.java`）

```java
public interface ExcelResult<T> {
    List<T> getList();           // 成功的数据
    List<String> getErrorList(); // 错误列表
    String getAnalysis();        // 导入回执（一句话总结）
}
```

**实现 `DefautExcelResult` 的 `getAnalysis`（`DefautExcelResult.java:73`）**：

```java
public String getAnalysis() {                                                  // :73
    int successCount = list.size();                                            // :74
    int errorCount = errorList.size();                                         // :75
    if (successCount == 0) {
        return "读取失败，未解析到数据";
    } else {
        if (errorCount == 0) {
            return StrUtil.format("恭喜您，全部读取成功！共{}条", successCount);
        }
        ...
    }
}
```

**为什么需要 `ExcelResult` 这个"三合一"对象？** 因为**导入不只是"成功"或"失败"两种结果**——**可能是"部分成功、部分失败"，而用户需要知道"哪些失败了、为什么"**。**所以返回"成功列表 + 错误列表 + 一句话回执"。**

**业务 Controller 里就一行**（`SysUserController.java:132-133`）：

```java
ExcelResult<SysUserImportVo> result = ExcelUtil.importExcel(
    file.getInputStream(), SysUserImportVo.class, new SysUserImportListener(updateSupport, roles, depts));
return R.ok(result.getAnalysis());        // ★只把"回执"返回给前端
```

### 4.11 自定义监听器：`SysUserImportListener`

**当"逐行 Bean Validation"不够用时，就要自定义监听器。** 用户导入的 `SysUserImportListener`（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/listener/SysUserImportListener.java:57`）**继承了 `AnalysisEventListener<SysUserImportVo>`（`:57`）**，在 `invoke`（`:94`）里做了**远超"字段非空"的业务校验**：

- **校验"角色名"是否存在**（`:104` "角色设置错误"、`:118` "角色id不能为空"）；
- **校验"部门路径"是否存在、有没有冲突**（`:187` "部门信息不能为空"、`:209` "部门名冲突"）；
- **校验通过的才真正入库**（`updateSupport` 决定"已存在的用户"是更新还是报错）；
- **最后 `getExcelResult()`（`:251`）汇总"失败清单"，有失败就整体抛出**（`:258`）。

**读懂这个设计**：**默认监听器管"通用校验"（字段级），自定义监听器管"业务校验"（跨表、跨行）**——**这是"通用能力放基座、业务规则放业务"的分层。**

**和 §2.5 的设计意图一致**：**错误"边读边收集"，读完"整体判断"——有错就整体抛 `ServiceException`（`:258`）**。**但注意（§2.5 已澄清）：成功行是在 `invoke` 里逐行 `insertUser` 的（`:136`），并没有被一个事务整体回滚**——"抛异常"不等于"已写入的也没了"。**这是回答"数据一致性"问题时必须说清的一点。**

### 4.12 分片上传：`mergeFiles`（大文件场景）

**超大文件（比如几百 MB 的合同扫描件）不适合"一次性上传"**——**网络一抖就前功尽弃**。**分片上传**：**前端把文件切成小块分别上传，后端最后合并。**

`FileUploadUtils.mergeFiles`（`:247`）：

```java
public static void mergeFiles(File[] chunks, Path finalPath) throws IOException {   // :247
    Files.createDirectories(finalPath.getParent());
    Arrays.sort(chunks, fileNameComparator);                                        // :253  ★按分片名排序（关键！）
    try (FileChannel finalChannel = FileChannel.open(finalPath, CREATE, WRITE, APPEND)) {
        Arrays.stream(chunks).forEach(chunk -> {
            try (FileChannel chunkChannel = new RandomAccessFile(chunk, "r").getChannel()) {
                finalChannel.transferFrom(chunkChannel, finalChannel.size(), chunkChannel.size());  // :265  ★零拷贝传输
                chunk.delete();                                                      // :268  ★合并后删分片
            } catch (IOException e) {
                log.error("删除分片临时文件失败", e);
            }
        });
    }
}
```

**两个技术点**：

1. **`Arrays.sort(chunks, fileNameComparator)`（`:253`）**——**分片必须"按顺序"合并**，否则拼出来的文件是乱的。**所以分片名必须带序号，排序器负责按序号排。** **这是分片上传最容易出错的地方。**
2. **`FileChannel.transferFrom`（`:265`）**——**NIO 的"零拷贝"传输**，**比"读进内存再写出"高效得多**（不经过用户态缓冲）。**大文件合并的最佳实践。**

**注意**：**这个 `mergeFiles` 方法存在，但没有被 `CommonController` 调用**——**说明分片上传的能力"备好了"，但通用上传接口没走它**（通用上传是"单文件一次性"，分片在业务模块里按需调用）。

### 4.13 ★ 坦白点（本章有四个）

**坦白点一：`/profile/**` 是免登录的——谁拿到 URL 谁就能看文件**

**事实**：`application.yml` 的 `security.excludes` 里有：

```yaml
    - /common/temp/resource     # :165
    - /profile/**               # :167  ★开放文件预览
```

**后果**：**`/profile/**` 不会被 Sa-Token 的登录拦截器拦住**（白名单在 `ruoyi-admin/src/main/resources/application.yml:167`）。**但"不被拦截"不等于"实际一定能匿名读到"**：`CommonController.view`（`ruoyi-admin/src/main/java/com/ruoyi/web/controller/common/CommonController.java:125-171`）真正读文件时用的是 `RuoYiConfig.getProfile()`（`:147`），而 `getProfile()` 会拼上 `LoginHelper.getTenantId()`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/config/RuoYiConfig.java:79-82`）——**没有登录上下文时拿不到租户，实际会抛异常（通常是 500），而不是把文件吐出来**。前端取图片时也是带 token 的（`zxcm-ui/src/utils/request.js:161-166` 用 axios 带 `token` 头请求）。

**所以准确的结论是**：**白名单 = 不做登录校验，但代码仍依赖登录上下文来决定"读哪个租户的目录"**；**不要把它简单说成"知道 URL 就能公开访问"**。风险依然存在：一旦某个部署/改造让 `getProfile` 不再依赖登录（例如固定租户目录、或 token 能从 cookie/URL 带入），UUID URL 一旦泄漏（分享、日志、截图），文件就不再安全。**答辩时按自己实测的行为说。**

**影响**：**对"公告配图"这类公开内容没问题；但对"合同扫描件、身份证照片"这类敏感文件，这是个安全问题。**

**怎么修**：**敏感文件不要走 `/profile/**` 静态预览，改用"需要鉴权的下载接口"**（校验当前用户有没有权限看这个文件），或者**接 OSS 用"带签名的临时 URL"**。

**坦白点二：两套上传大小限制不一致（80MB vs 100MB）**

**事实**：`spring.servlet.multipart.max-file-size: 80MB`（`application.yml:72`）vs `FileUploadUtils.DEFAULT_MAX_SIZE = 100MB`（`FileUploadUtils.java:51`）。

**后果**：**Servlet 层先拦（80MB），代码层那个 100MB 永远触发不了**——**代码层的"大小限制"形同虚设，而且两个数字不同会让人误以为限制是 100MB。**

**怎么修**：**让两处一致**（要么都 80MB，要么把 Servlet 层设成比代码层大一点、让代码层做"精细化限制"）。**更好的是"用同一个配置项驱动两处"。**

**坦白点三：文件路径写死在配置文件里，且 dev/prod 是同一个 Windows 绝对路径**

**事实**：

```yaml
# application-dev.yml:4
ruoyi:
  profile: D:\java project\document\Resources
# application-prod.yml:4
ruoyi:
  profile: D:\java project\document\Resources     # ★和 dev 一模一样
```

**后果**：① **硬编码 Windows 盘符路径**——**部署到 Linux 服务器上会直接找不到目录**；② **dev 和 prod 指向同一路径**——**在生产环境本身就是个隐患（开发时的测试文件和生产文件混在一起）**。

**怎么修**：**用环境变量或 `${user.home}` 之类的占位符**，**并且 dev/prod 分开配置**（生产环境应该是 `/data/zxcm/files` 这类 Linux 路径）。

**坦白点四：`CommonController.uploadFile` 自己 catch 了异常**

**事实**：`uploadFile`（`:100`）里 `try { ... } catch (Exception e) { return R.fail(e.getMessage()); }`（`:116-118`）。

**后果**：**上传失败时，异常不会流到第 11 章的 `GlobalExceptionHandler`**——**所以①不会打"系统异常"的日志堆栈（排查困难）；②返回给前端的 `e.getMessage()` 可能是原始异常信息（信息泄漏）。**

**为什么这么写？** 大概率是**想让"文件太大""后缀不允许"这些**校验异常的提示**直接透传给用户**（`FileSizeLimitExceededException` 的 message 是中文提示）。**但代价是"异常处理风格不统一"。**

**更规范的做法**：**在 `GlobalExceptionHandler` 里加 `FileSizeLimitExceededException` / `InvalidExtensionException` 的 handler**（就像第 11 章处理 `SQLException` 那样），**让上传异常也走统一出口。**

### 4.14 动手验证：上传和导入各跑一遍

**前置**：应用启动、已登录，后续请求都带请求头 `token: Bearer <你的登录 token>`（头名见 `ruoyi-admin/src/main/resources/application.yml:103`，`Bearer` 前缀见 `application.yml:119`，前端拼法见 `zxcm-ui/src/utils/request.js:33`）。

**A. 上传一张小图片**

1. 打开 Knife4j（`http://localhost:8080/doc.html`），在 `CommonController` 里调 `POST /common/upload`，选一张 `test.png`。
   **预期返回**：`code=200`，`data` 里有四个键：`fileName`、`uid`、`realName`、`url`（`ruoyi-admin/src/main/java/com/ruoyi/web/controller/common/CommonController.java:109-114`）。
2. 到磁盘找文件（PowerShell）：

   ```powershell
   Get-ChildItem -Recurse "D:\java project\document\Resources" -Filter *.png |
     Sort-Object LastWriteTime | Select-Object -Last 5 FullName
   ```

   **预期路径**：`...\Resources\<租户ID>\upload\<yyyy>\<MM>\<dd>\<uuid>.png`——日期目录和 UUID 来自 `FileUploadUtils.java:138`，租户目录来自 `RuoYiConfig.java:79-82`。
3. 访问返回的 `url`（形如 `http://localhost:8080/profile/upload/<yyyy>/<MM>/<dd>/<uuid>.png`）。**分两种请求各试一次**：不带 token 直接浏览器打开、带 `token: Bearer <token>` 用 curl 打开。`/profile/**` 虽然在免登录白名单里（`ruoyi-admin/src/main/resources/application.yml:167`），但 `view` 内部还要用 `RuoYiConfig.getProfile()` 取租户目录（`RuoYiConfig.java:79-82`）——**不带登录上下文可能直接 500**。**以你本机实测结果为准，这正是 §4.13 坦白点一说的"白名单 ≠ 匿名一定可读"。**
4. 故意上传一个 `test.jsp`（内容随便写）。**预期被拒**：后缀不在 `DEFAULT_ALLOWED_EXTENSION` 里（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/MimeTypeUtils.java:42`），`assertAllowed` 抛 `InvalidExtensionException`（`FileUploadUtils.java:177-192`），接口返回 `R.fail`。
5. 断点：`CommonController.java:107`（进入工具类前）、`FileUploadUtils.java:122`（校验）、`:127`（落盘）、`CommonController.java:150`（预览写响应）。

**B. 导入一个"故意填错"的 Excel**

1. 先调 `POST /system/user/importTemplate` 下载模板（`ruoyi-admin/src/main/java/com/ruoyi/web/controller/system/SysUserController.java:137-139`），填一行用户；**把"用户性别"填成模板里没有的值**（比如 `外星人`）。
2. 调 `POST /system/user/importData` 上传（`SysUserController.java:128-133`）。
   **预期**：请求失败，错误信息里出现"解析字典失败，请检查excel格式是否正确"——这是 `ExcelDictConvert` 在转换阶段抛出的（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/convert/ExcelDictConvert.java:66-68`）。
3. 再试一种错误：**性别填对，但"用户账号"留空**。这时 Bean Validation 在 `SysUserImportListener.invoke` 里抛异常，被 `:175-180` 捕获并记入 `failureMsg`，**其他行会继续处理**；读完 `getExcelResult`（`:251-263`）发现有失败，整体抛 `ServiceException`。
4. 断点：`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/listener/SysUserImportListener.java:94`（每行入口）、`:136`（真正入库）、`:175`（捕获行级错误）、`:251`（汇总）。
5. **重要观察**：如果第 1 行合法、第 2 行账号为空，第 1 行已经通过 `:136` 入库了——所以别把"全成功或全失败"当成必然（§4.11 末尾有澄清）。

**C. 导出一份 Excel**

调 `POST /system/user/export`（`SysUserController.java:107-120`），打开下载的 xlsx。**预期**：表头是中文，状态/性别是"男/女""正常/停用"这种标签而不是 `0/1`——转换发生在 `ExcelDictConvert.convertToExcelData`（`ExcelDictConvert.java:72-86`）。

---

## 五、一个操作的完整生命流程

**场景一：上传一张头像图片**

```
┌─ 1. 前端 POST /common/upload（multipart/form-data） ────────┐
│  XssFilter / RepeatableFilter（第 10 章）                    │
│    └─ ★注意：这两个过滤器都跳过非 JSON 请求，multipart 不受影响 │
│  Spring MultipartResolver 解析出 MultipartFile              │
│    └─ 若文件 > 80MB（application.yml:72）→ Tomcat 直接拒绝    │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 2. CommonController.uploadFile(:100) ─────────────────────┐
│  filePath = RuoYiConfig.getUploadPath()  → profile/<租户>/upload │
│  FileUploadUtils.upload(filePath, file)(:107)               │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 3. FileUploadUtils 四步 ──────────────────────────────────┐
│  ① 文件名长度 ≤ 100（:118）                                  │
│  ② assertAllowed（:122）                                    │
│     ├─ 大小 ≤ 100MB（但 Servlet 层已先拦 80MB）              │
│     └─ 后缀在白名单内（png/jpg/...）（:177）                 │
│  ③ extractFilename（:135）—— 丢弃原名，生成 UUID 名          │
│     └─ "2026/10/17/8f3a...c1.png"                           │
│  ④ getAbsoluteFile（调用 :126、定义 :142）→ transferTo(:127) │
│  返回 pathFileName = "/profile/upload/2026/10/17/xxx.png"   │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 4. 拼 URL 返回 ───────────────────────────────────────────┐
│  url = serverConfig.getUrl() + "/profile/upload/2026/10/17/<uuid>.png" │
│  R.ok({ fileName, uid, realName, url })                     │
└──────────────────────────────────────────────────────────┘
                    ↓
┌─ 5. 前端显示图片（GET /profile/upload/<日期>/<uuid>.png，免登录） ┐
│  CommonController.view(:126)                                │
│    ├─ 后缀必须是图片（:135）                                 │
│    ├─ 去 ".." 防穿越（:143）                                 │
│    └─ FileUtils.writeBytes 写进响应（:150）                  │
└──────────────────────────────────────────────────────────┘
```

**场景二：导入用户 Excel**

```
┌─ 1. 前端 POST /system/user/importData（带 .xlsx + updateSupport）┐
│  SysUserController.importData(:129)                             │
│    @SaCheckPermission("system:user:import")  ← 先鉴权（第 3 章） │
└─────────────────────────────────────────────────────────────────┘
                    ↓
┌─ 2. ExcelUtil.importExcel(is, clazz, new SysUserImportListener(...))(:132) ┐
│  EasyExcel 流式读（SAX，内存恒定）                                │
│    对每一行：                                                    │
│      ├─ DefaultExcelListener.invoke(:104) 逐字段校验             │
│      │    └─ ValidatorUtils.validate(data)(:106)                │
│      ├─ SysUserImportListener.invoke(:94) 业务校验               │
│      │    ├─ 角色名存在吗？（:104）                              │
│      │    └─ 部门路径存在吗？冲突吗？（:209）                     │
│      ├─ 出错 → onException(:69) 拼"第X行-第Y列-表头Z" 收进 errorList │
│      └─ 通过 → 收进 list                                        │
└─────────────────────────────────────────────────────────────────┘
                    ↓
┌─ 3. 读完后判定 ─────────────────────────────────────────────┐
│  SysUserImportListener.getExcelResult()(:251)               │
│    ├─ 有失败 → 抛 ServiceException(汇总失败清单)(:258)        │
│    └─ 全成功 → 返回 result                                   │
└─────────────────────────────────────────────────────────────────┘
                    ↓
┌─ 4. 返回回执 ───────────────────────────────────────────────┐
│  R.ok(result.getAnalysis())  → "恭喜您，全部读取成功！共 500 条" │
│  （若失败 → GlobalExceptionHandler 兜底返回错误清单）           │
└─────────────────────────────────────────────────────────────────┘
```

**两个场景对照的收获**：

| | 文件上传 | Excel 导入 |
|---|---|---|
| **入口** | 通用 `CommonController` | 各业务 Controller |
| **流式与否** | 一次性落盘 | **流式逐行（EasyExcel）** |
| **校验** | **后缀白名单 + 大小** | **字段校验 + 业务校验** |
| **错误反馈** | 抛异常（单条） | **收集成清单（多条）** |
| **一致性** | 单文件，无所谓 | **目标是全成功或全失败；但当前是逐行入库，失败时前面的行不会自动回滚（§2.5 澄清）** |

---

## 六、面试官高频问题 + 答案

**Q1：你们项目文件上传是怎么做的？文件存哪？**

**A**：**没有接 OSS，存本地磁盘**——根目录是配置项 `ruoyi.profile`（`application-dev.yml:4`），通用上传落在 `profile/upload`。
**上传流程**：`CommonController.uploadFile`（`:100`）接 `MultipartFile` → `FileUploadUtils.upload`（`:107`）**做四件事：校验文件名长度（`:118`）、校验大小和后缀白名单（`:122`/`:168`）、用"日期目录 + UUID"生成安全文件名（`:138`）、落盘**，最后返回 `{fileName, uid, realName, url}`。
**坦白**：**存本地意味着"多台服务器文件不共享"**（要配 NFS 或共享盘），**而且 `/profile/**` 在免登录白名单里**（`application.yml:167`）——**但注意，处理它的 `view` 方法内部还会用 `RuoYiConfig.getProfile()` 取租户目录（`RuoYiConfig.java:79-82`），没登录上下文时实际会报错**，所以"白名单"不等于"匿名一定可读"（§4.13 有完整解释）。**敏感文件仍不要依赖这个接口，应该走鉴权下载。**

**Q2：上传的文件名为什么要改成 UUID？直接存原文件名不行吗？**

**A**：**因为用户上传的文件名不可信**，能造成：**路径穿越**（`../../etc/passwd`）、**重名覆盖**、**XSS**、**超长名**、**可执行后缀**等问题。**用 `日期路径 + UUID + 后缀`（`FileUploadUtils.java:138`）一次性解决**——**随机、唯一、字符安全、不可预测**。
**代价是原始文件名丢了**，**所以我们把 `realName`（原名）随返回值一起给前端**（`CommonController.java:112`），需要显示时用它。

**Q3：为什么要做"后缀白名单"？白名单和黑名单哪个好？**

**A**：**防上传可执行文件**（`.jsp`/`.sh` 之类，结合解析漏洞可能被当代码跑）。**白名单（只允许列出的）永远比黑名单安全**——**因为黑名单永远列不全**，攻击者总能找到你没想到的后缀。**本项目用的是白名单**（`MimeTypeUtils.DEFAULT_ALLOWED_EXTENSION`，`FileUploadUtils.java:177` 校验）。

**Q4：`/profile/**` 预览接口为什么只允许图片后缀、还要去 `..`？**

**A**：**两道防护**：① **只允许图片后缀**（`CommonController.java:135`）——**防止有人拿"图片预览"当"任意文件下载器"**；② **`replace("..", "")`（`:143`）防路径穿越**——**否则 `/profile/../../../../etc/passwd` 就能读系统文件**。
**坦白**：**这个 `replace` 是"黑名单式"防护，理论上不如"规范化路径后判断是否越界"（`Path.normalize()` + `startsWith`）严密**，实践中够用但不是最优。

**Q5：Excel 导入为什么用 EasyExcel 不用 POI？**

**A**：**内存**。**POI 把整个 Excel 读进内存**（`XSSFWorkbook`），**行数一大就会占用大量内存**，并发导入容易 OOM；**EasyExcel 用 SAX 流式解析，"读一行、处理一行、丢一行"，解析阶段内存占用恒定**（`ExcelUtil.java:49/63`）。
**代价**：**不能随机访问单元格**（流式不能回头），**但"逐行导入"完全够用**。

> **追问：既然 EasyExcel 是流式的，导入 100 万行就一定不会 OOM 吗？**
> **一句话答**：不一定。**解析**是流式的，但 `DefaultExcelListener` 把每个成功行都加进 `excelResult.getList()`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/excel/DefaultExcelListener.java:108`），**结果集仍在内存里越积越大**。真正的超大导入要"边读边分批落库、不在内存里留全量 List"。

**Q6：Excel 里的"男/女"怎么变成数据库的"0/1"？**

**A**：**靠注解 + 转换器 + SPI**。VO 字段上标 `@ExcelDictFormat(dictType = "sys_user_sex")`，EasyExcel 通过 `converter = ExcelDictConvert.class` 调这个转换器：**导入时 `convertToJavaData` 把"男"翻成"0"（标签→码值），导出时 `convertToExcelData` 把"0"翻成"男"（码值→标签）**。**转换器里用 `SpringUtils.getBean(DictService.class)` 拿字典服务**——`DictService` 是 common 定义的接口、system 实现（`SysDictTypeServiceImpl.java:48`），**靠 SPI 解决"common 不能依赖 system"的问题**。

**Q7：Excel 导入校验失败怎么处理？**

**A**：**"逐行校验、收集错误、一次性反馈"**。`DefaultExcelListener.invoke`（`:104`）每读一行做 Bean Validation（`:106`），`onException`（`:69`）**能精确到"第几行第几列哪个表头"**，错误收进 `errorList`。**读完由 `getExcelResult()`（`SysUserImportListener.java:251`）判定：有错就整体抛异常**。
**但要如实补一句**：用户导入是在 `invoke` 里逐行 `insertUser` 的（`SysUserImportListener.java:136`），**没有外层事务把整份 Excel 包住**——所以前面的成功行可能已经落库，抛异常只代表"给前端一个失败回执"，不代表数据回滚了。**"要么全成功、要么全失败"是设计目标，不是当前实现的保证。**
**为什么不一错就停？** **因为用户需要知道"所有哪里错了"以便一次改完**，不然改一行传一次太痛苦。

**Q8：Excel 导出有什么坑？**

**A**：两个，本项目都处理了：① **列宽**——不处理的话列宽默认，"用户名"显示不全，我们注册了 `LongestMatchColumnWidthStyleStrategy`（`ExcelUtil.java:97`）自动适配；② **大数字失真**——**Excel 超过 15 位的数字会变科学计数法**（身份证、订单号就废了），我们注册了 `ExcelBigNumberConvert`（`:99`）让它按字符串输出。**这两个都是国内业务场景的血泪经验。**

**Q9：上传大小限制配在哪？你们配的多少？**

**A**：**两层**：**Servlet 层 `spring.servlet.multipart.max-file-size: 80MB`（`application.yml:72`）+ 代码层 `FileUploadUtils.DEFAULT_MAX_SIZE = 100MB`（`FileUploadUtils.java:51`）**。Servlet 层"请求还没进 Controller 就拦、不占内存"；代码层"给不同类型文件留不同上限的口子"。
**坦白**：**我们这两处数值不一致（80 vs 100）**——**实际是 Servlet 层先拦，代码层那个 100MB 永远触发不了**，**应该统一。**

**Q10：大文件上传怎么优化？**

**A**：**分片上传 + 合并**。本项目 `FileUploadUtils.mergeFiles`（`:247`）就是合并逻辑：**分片必须按序数排序后合并**（`:253`，否则文件是乱的）、**用 `FileChannel.transferFrom` 零拷贝**（`:265`）、**合并后删分片**（`:268`）。
**坦白**：**这个方法目前没被通用上传接口调用**——**能力备好了，但通用上传走的还是"单文件一次性"**。

### 本章答辩卡（3 组追问级问答）

**卡 1｜问：导入模板是怎么生成的？为什么不用手写一个空 Excel 文件？**

- **10 秒答案**：模板就是"只导出表头、不导出数据"的导出。`/system/user/importTemplate` 调 `ExcelUtil.exportExcel(new ArrayList<>(), "用户数据", SysUserImportVo.class, response)`（`ruoyi-admin/src/main/java/com/ruoyi/web/controller/system/SysUserController.java:137-139`），传一个空 List；EasyExcel 根据 `SysUserImportVo` 上的 `@ExcelProperty`（`ruoyi-base/ruoyi-system/src/main/java/com/ruoyi/system/domain/vo/SysUserImportVo.java:38-83`）自动生成表头。好处是**模板和导入解析共用同一套 VO 注解，永远不会"模板列名和导入字段对不上"**。
- **证据**：`SysUserController.java:137-139`、`SysUserImportVo.java:38-83`、`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/poi/ExcelUtil.java:87-100`。

**卡 2｜问：敏感合同扫描件也走 `/profile/**`，安全吗？你会怎么改？**

- **10 秒答案**：不安全。`/profile/**` 在 `security.excludes` 里（`ruoyi-admin/src/main/resources/application.yml:167`），拿到 URL 的人不登录也能读图（`CommonController.java:125-171` 只校验"是不是图片后缀"）。改法：敏感文件不进 `/profile/**`，改走"带鉴权的下载接口"（校验 `@SaCheckPermission` + 文件归属），或者接 OSS 发临时签名 URL。
- **证据**：`ruoyi-admin/src/main/resources/application.yml:167`、`ruoyi-admin/src/main/java/com/ruoyi/web/controller/common/CommonController.java:125-171`。

**卡 3｜问：上传过程中服务重启，磁盘上会留下半截文件吗？你们怎么处理？**

- **10 秒答案**：会。`file.transferTo(desc)`（`ruoyi-base/ruoyi-common/src/main/java/com/ruoyi/common/utils/file/FileUploadUtils.java:127`）一旦中途失败或进程被杀，已写出的部分不会被这段代码清理——`upload` 方法里没有 `try/finally` 删除。规范做法是"先写临时目录 → 成功后再原子移动到正式目录 → 定时清理过期临时文件"。
- **证据**：`FileUploadUtils.java:124-129`（无异常清理）、`:132-135`（已有 `getTempPath()` 可当临时目录）。

---

## 七、选型对比

### 7.1 文件存储方案

| 方案 | 多机共享 | 成本 | 扩展性 | 本项目 |
|---|---|---|---|---|
| **本地磁盘（本项目）** | **✗（要配 NFS/共享盘）** | 最低 | 差（受单机磁盘限制） | **✔** |
| **OSS / MinIO / 七牛** | **✔** | 中 | **好（近乎无限）** | ✗ |
| 数据库 Blob | ✔ | 高（撑大数据库） | 差 | ✗ |
| 分布式文件系统（HDFS） | ✔ | 高 | 好（但重） | ✗ |

**结论**：**本地磁盘是"最简单、最便宜、但最不扩展"的方案**——**适合"单机部署 + 对可靠性要求不高"的场景。** **本项目选它，代价是"多机部署要额外配共享存储、备份麻烦、URL 免登录"。** **如果需要更规范的方案，MinIO（自建、兼容 S3）是性价比之选。**

### 7.2 Excel 库选型

| 库 | 内存模型 | 大数据 | 上手 | 本项目 |
|---|---|---|---|---|
| **Apache POI** | **DOM（全量进内存）** | **差（易 OOM）** | 中 | ✗ |
| **EasyExcel（本项目）** | **SAX（流式）** | **好（内存恒定）** | 好 | **✔** |
| EasyPOI | 基于 POI | 差 | **好** | ✗（`zxcm-excel` 有但没用于导入导出） |
| hutool Excel | 封装 POI | 差 | 好 | ✗ |

**结论**：**数据量一上去，POI 家族就 OOM，EasyExcel 是"国内大数据量 Excel"的事实标准。** 本项目选了它，**而且把"字典翻译""错误定位""大数防失真"都包好了**——**这层封装是基座的价值。**

### 7.3 导入的处理模型

| 模型 | 内存 | 错误反馈 | 一致性 | 本项目 |
|---|---|---|---|---|
| **全量读进 List 再处理** | 差（大文件 OOM） | 简单 | 好控制 | ✗ |
| **流式逐行 + 监听器（本项目）** | **好** | **丰富（行/列/表头）** | **好（全成或全败）** | **✔** |
| 边读边写库 | 好 | 差（写一半失败难回滚） | **差** | ✗ |

**结论**：**"流式 + 监听器 + 收集错误 + 整体判定"是目前最优的导入模型**——**内存安全、错误可读、数据一致。** 本项目用的就是它。

---

## 延伸阅读（网上优秀资料）

- **EasyExcel 官方文档**：<https://easyexcel.opensource.alibaba.com/> —— **`@ExcelProperty`、监听器、转换器、读写大数据的完整指南**（本项目 `ExcelUtil` 的底层，务必看"读 Excel"和"写 Excel"两章）。
- **EasyExcel · 为什么比 POI 省内存**：官方文档「性能测试 / 内存占用」章节，或搜「EasyExcel POI 内存 SAX 原理」—— **理解 DOM vs SAX 的区别**，面试问"为什么用 EasyExcel"照这个答。
- **OWASP · 文件上传安全**：<https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html> —— **文件上传的权威安全清单**（白名单、重命名、存放到 webroot 之外、禁止执行权限）。
- **Spring · `MultipartFile` 与 multipart 配置**：Spring 官方文档搜「Multipart / Handling File Uploads」—— `max-file-size`、`max-request-size` 的作用。
- **`FileChannel.transferFrom` 零拷贝**：搜「Java NIO 零拷贝 transferTo transferFrom 原理」—— 理解 §4.12 合并文件为什么高效。
- **路径穿越（Path Traversal）**：搜「路径穿越漏洞 防御 normalize」—— **理解为什么 `replace("..","")` 不够、`Path.normalize()` + `startsWith` 更好**。
- **RuoYi-Vue-Plus 官方文档 · 文件与 Excel**：搜「RuoYi-Vue-Plus 文件上传 Excel 导入导出」—— 对照官方怎么讲这套设计。

---

## 本章小结 & 下一步

- **文件存储**：**存本地磁盘**，根目录 `ruoyi.profile`（`application-dev.yml:4`），上传落 `profile/upload`。
- **上传四步**（`FileUploadUtils.java:114`）：**校验文件名长度（`:118`）→ `assertAllowed` 校验大小+后缀白名单（`:122`/`:168`）→ `extractFilename` 生成 `日期路径 + UUID + 后缀`（`:138`）→ 落盘**。
- **★ 安全要点**：**用户文件名不可信**（路径穿越/重名/XSS/超长/可执行后缀）→ **UUID 重命名**；**白名单优于黑名单**；**预览接口限图片后缀（`CommonController.java:135`）+ 去 `..`（`:143`）**；**下载前 `checkAllowDownload`（`FileUtils.java:159`）**。
- **Excel 导出**：`ExcelUtil.exportExcel`（`:87`）→ EasyExcel + **自动列宽（`:97`）+ 大数防失真（`:99`）**；业务侧一行调用（`SysUserController.java:119`）。
- **Excel 导入**：三个重载（`:48` 同步 / `:61` 带校验 / `:75` 自定义监听器）→ **流式 SAX（内存恒定）**。
- **★ 注解驱动**：`@ExcelProperty`（列↔字段）+ `@ExcelDictFormat(dictType=...)` + `ExcelDictConvert`（**导入标签→码值、导出码值→标签**）→ 靠 SPI **`DictService`**（common 定义、`SysDictTypeServiceImpl.java:48` 实现）解耦。
- **错误反馈**：`DefaultExcelListener.invoke(:104)` 逐行校验 + `onException(:69)` **定位"第X行第Y列表头Z"** → `ExcelResult`（`getList` + `getErrorList` + `getAnalysis :73`）→ **设计目标是"全成功或全失败"，但用户导入是逐行入库、无整体事务（§2.5 澄清）**。
- **分片合并**：`mergeFiles(:247)`——**按序排序（`:253`）+ `transferFrom` 零拷贝（`:265`）+ 删分片（`:268`）**（但通用上传未走它）。
- **坦白点**：① **`/profile/**` 在免登录白名单里**（`application.yml:167`），但 `view` 内部仍依赖登录上下文取租户目录（`RuoYiConfig.java:79-82`）——**"不拦截"不等于"匿名一定可读"，答辩按实测说**；② **两套大小限制不一致**（80MB Servlet vs 100MB 代码）；③ **文件路径写死 Windows 绝对路径、dev/prod 相同**（`application-dev.yml:4`/`application-prod.yml:4`）；④ **`uploadFile` 自己 catch 了异常**（`CommonController.java:116`），未走全局处理器。
- **可迁移的思维**：**"外部输入一律不可信"**（文件名）、**"白名单优于黑名单"**、**"流式处理对抗大数据"**（EasyExcel）、**"错误收集优于立即中断"**（导入体验）。

**下一章（第 14 章 · WebSocket · 异步 · 线程池 · 定时任务）**：本章讲"文件和批量数据"，**第 14 章讲"后台跑的东西"**——**WebSocket** 怎么做到"服务端主动推消息给前端"（在线用户、消息通知）；**`@Async` + 线程池**怎么把"不该阻塞主流程的事"丢到后台（并回顾第 11 章那个"异步异常进不了全局处理器"的坑）；**定时任务**怎么在集群里"只跑一次"；以及**线程池参数怎么配才不会被撑爆**。
