# ===== 文件上传/下载、数据权限规范 =====

> 适用于所有后端模块，与 backend-monolith.md / backend-microservice.md 配合使用

---

## 一、文件上传/下载规范

### 1. 文件上传限制

```java
// application.yml
spring:
  servlet:
    multipart:
      max-file-size: 10MB          # 单文件最大 10MB
      max-request-size: 50MB       # 单次请求最大 50MB
```

```java
/**
 * 文件类型白名单校验（不信任前端传的 Content-Type）
 */
public class FileValidator {

    private static final Set<String> ALLOWED_TYPES = Set.of(
        "image/jpeg", "image/png", "image/gif", "image/webp",
        "application/pdf",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
    );

    private static final Map<String, byte[]> FILE_SIGNATURES = Map.of(
        "image/jpeg", new byte[]{(byte) 0xFF, (byte) 0xD8, (byte) 0xFF},
        "image/png",  new byte[]{(byte) 0x89, 0x50, 0x4E, 0x47},
        "image/gif",  new byte[]{0x47, 0x49, 0x46, 0x38},
        "application/pdf", new byte[]{0x25, 0x50, 0x44, 0x46}
    );

    /**
     * 校验文件：MIME 类型 + 文件头魔数
     */
    public static void validate(MultipartFile file) {
        // 1. 检查文件是否为空
        if (file == null || file.isEmpty()) {
            throw new BizException("文件不能为空");
        }

        // 2. 检查文件扩展名
        String originalName = file.getOriginalFilename();
        String extension = FilenameUtils.getExtension(originalName).toLowerCase();
        if (extension.matches("exe|sh|bat|cmd|js|vbs|ps1")) {
            throw new BizException("不允许上传可执行文件");
        }

        // 3. 检查 MIME 类型
        String contentType = file.getContentType();
        if (!ALLOWED_TYPES.contains(contentType)) {
            throw new BizException("不支持的文件类型: " + contentType);
        }

        // 4. 检查文件头魔数（防止绕过：把 .exe 改成 .jpg）
        try {
            byte[] header = new byte[8];
            file.getInputStream().read(header);
            byte[] signature = FILE_SIGNATURES.get(contentType);
            if (signature != null) {
                for (int i = 0; i < signature.length; i++) {
                    if (header[i] != signature[i]) {
                        throw new BizException("文件内容与类型不匹配");
                    }
                }
            }
        } catch (IOException e) {
            throw new BizException("文件读取失败");
        }
    }
}
```

### 2. 文件命名规范

```java
// 文件名：UUID + 原始扩展名（防止中文文件名、特殊字符、路径穿越攻击）
String originalName = file.getOriginalFilename();
String extension = FilenameUtils.getExtension(originalName);
String newFileName = IdUtil.fastSimpleUUID() + "." + extension;

// 错误：直接使用原始文件名
String path = uploadDir + "/" + file.getOriginalFilename();  // 中文、空格、../ 都是坑
```

### 3. 存储方案选择

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **本地磁盘** | 小项目、文件量少 | 简单直接 | 不支持分布式、难扩容 |
| **MinIO（自建 OSS）** | 中小项目、私有部署 | S3 兼容、可私有化 | 需要运维 |
| **阿里 OSS / 腾讯 COS** | 生产环境 | 免运维、高可用、CDN | 有费用 |
| **FastDFS** | 大量小文件存储 | 高性能 | 运维复杂、社区不活跃 |

**推荐**：生产环境用 MinIO 或云 OSS，开发环境用本地磁盘。

### 4. 文件上传 Controller 模板

```java
@Slf4j
@RestController
@RequestMapping("/api/file")
public class FileController {

    @Value("${file.upload-dir:./uploads}")
    private String uploadDir;

    @Value("${file.base-url:http://localhost:8080/files}")
    private String baseUrl;

    @PostMapping("/upload")
    public R<FileVO> upload(@RequestParam("file") MultipartFile file) {
        // 1. 校验文件
        FileValidator.validate(file);

        // 2. 生成文件名和路径
        String extension = FilenameUtils.getExtension(file.getOriginalFilename());
        String newFileName = IdUtil.fastSimpleUUID() + "." + extension;

        // 按日期分目录：uploads/2024/06/01/xxx.jpg
        String datePath = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy/MM/dd"));
        Path filePath = Paths.get(uploadDir, datePath, newFileName);

        try {
            // 3. 保存文件
            Files.createDirectories(filePath.getParent());
            file.transferTo(filePath.toFile());
        } catch (IOException e) {
            log.error("文件保存失败", e);
            throw new BizException("文件上传失败");
        }

        // 4. 返回访问 URL
        FileVO vo = new FileVO();
        vo.setOriginalName(file.getOriginalFilename());
        vo.setFileName(newFileName);
        vo.setUrl(baseUrl + "/" + datePath + "/" + newFileName);
        vo.setSize(file.getSize());
        return R.ok(vo);
    }
}
```

### 5. 文件下载（流式，避免 OOM）

```java
@GetMapping("/download/{fileName}")
public void download(@PathVariable String fileName, HttpServletResponse response) {
    // 1. 安全校验：防止路径穿越
    if (fileName.contains("..") || fileName.contains("/") || fileName.contains("\\")) {
        throw new BizException("非法文件名");
    }

    // 2. 查找文件（实际项目中从 DB 查文件路径）
    Path filePath = findFilePath(fileName);
    if (!Files.exists(filePath)) {
        throw new BizException("文件不存在");
    }

    try {
        // 3. 设置响应头
        response.setContentType(MediaType.APPLICATION_OCTET_STREAM_VALUE);
        response.setHeader("Content-Disposition",
            "attachment; filename=\"" + URLEncoder.encode(fileName, "UTF-8") + "\"");
        response.setContentLengthLong(Files.size(filePath));

        // 4. 流式写入（不要一次性读入内存！）
        try (InputStream in = Files.newInputStream(filePath);
             OutputStream out = response.getOutputStream()) {
            byte[] buffer = new byte[8192];
            int len;
            while ((len = in.read(buffer)) != -1) {
                out.write(buffer, 0, len);
            }
            out.flush();
        }
    } catch (IOException e) {
        log.error("文件下载失败, fileName={}", fileName, e);
        throw new BizException("文件下载失败");
    }
}
```

### 6. 大文件上传（分片 + 断点续传）

**方案概述**：

```
前端：文件切片（每片 5MB） → 逐片上传 → 全部完成后通知合并
后端：接收分片 → 存储分片 → 合并分片 → 清理临时文件
```

```java
@PostMapping("/upload/chunk")
public R<Void> uploadChunk(
        @RequestParam("file") MultipartFile chunk,
        @RequestParam("fileId") String fileId,
        @RequestParam("chunkIndex") Integer chunkIndex) {

    String chunkDir = uploadDir + "/chunks/" + fileId;
    Path chunkPath = Paths.get(chunkDir, "chunk_" + chunkIndex);

    try {
        Files.createDirectories(chunkPath.getParent());
        chunk.transferTo(chunkPath.toFile());
    } catch (IOException e) {
        throw new BizException("分片上传失败");
    }
    return R.ok();
}

@PostMapping("/upload/merge")
public R<FileVO> mergeChunks(
        @RequestParam("fileId") String fileId,
        @RequestParam("fileName") String fileName,
        @RequestParam("totalChunks") Integer totalChunks) {

    String chunkDir = uploadDir + "/chunks/" + fileId;
    String extension = FilenameUtils.getExtension(fileName);
    String mergedFileName = fileId + "." + extension;

    try (OutputStream out = Files.newOutputStream(Paths.get(uploadDir, mergedFileName))) {
        for (int i = 0; i < totalChunks; i++) {
            Path chunkPath = Paths.get(chunkDir, "chunk_" + i);
            Files.copy(chunkPath, out);
        }
    } catch (IOException e) {
        throw new BizException("文件合并失败");
    }

    // 清理分片临时文件
    FileUtil.del(chunkDir);

    FileVO vo = new FileVO();
    vo.setFileName(mergedFileName);
    return R.ok(vo);
}
```

### 7. Excel 导入导出（EasyExcel）

```java
// 导出
@GetMapping("/export")
public void export(HttpServletResponse response) {
    response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    response.setHeader("Content-Disposition", "attachment; filename=users.xlsx");

    List<UserExcelVO> data = userService.listForExport();
    EasyExcel.write(response.getOutputStream(), UserExcelVO.class)
        .sheet("用户列表")
        .doWrite(data);
}

// 导入
@PostMapping("/import")
public R<ImportResultVO> importExcel(@RequestParam("file") MultipartFile file) {
    List<UserExcelVO> list = EasyExcel.read(file.getInputStream())
        .head(UserExcelVO.class)
        .sheet()
        .doReadSync();

    // 数据校验 + 批量写入
    return R.ok(userService.batchImport(list));
}
```

---

## 二、数据权限 / 多租户

### 1. 数据权限模型

| 权限范围 | 值 | SQL 效果 |
|---------|---|---------|
| 全部数据 | `1` | 无过滤条件 |
| 本部门数据 | `2` | `WHERE dept_id = #{currentDeptId}` |
| 本部门及下级 | `3` | `WHERE dept_id IN (#{deptIds})` |
| 仅本人数据 | `4` | `WHERE create_user = #{currentUserId}` |
| 自定义 | `5` | 按角色配置的部门列表过滤 |

### 2. @DataScope 自定义注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DataScope {
    /** 部门表的别名 */
    String deptAlias() default "d";
    /** 用户表的别名 */
    String userAlias() default "u";
}
```

### 3. MyBatis 拦截器实现方案

```java
@Intercepts({
    @Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})
})
@Component
public class DataScopeInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler handler = (StatementHandler) invocation.getTarget();
        MetaObject metaObject = SystemMetaObject.forObject(handler);

        // 获取 Mapper 方法上的 @DataScope 注解
        MappedStatement ms = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        DataScope dataScope = getDataScopeAnnotation(ms);
        if (dataScope == null) {
            return invocation.proceed();  // 没有注解，直接放行
        }

        // 获取当前用户的数据权限配置
        LoginUser loginUser = SecurityUtil.getLoginUser();
        if (loginUser == null || loginUser.isAdmin()) {
            return invocation.proceed();  // 管理员不过滤
        }

        // 拼接数据权限 WHERE 条件
        String originalSql = (String) metaObject.getValue("delegate.boundSql.sql");
        String dataScopeSql = buildDataScopeSql(originalSql, dataScope, loginUser);
        metaObject.setValue("delegate.boundSql.sql", dataScopeSql);

        return invocation.proceed();
    }

    private String buildDataScopeSql(String sql, DataScope scope, LoginUser user) {
        StringBuilder condition = new StringBuilder();

        switch (user.getDataScopeType()) {
            case 1: // 全部
                return sql;  // 不追加条件
            case 2: // 本部门
                condition.append(String.format(" AND %s.dept_id = %d",
                    scope.deptAlias(), user.getDeptId()));
                break;
            case 3: // 本部门及下级
                List<Long> deptIds = deptService.getDeptAndChildrenIds(user.getDeptId());
                condition.append(String.format(" AND %s.dept_id IN (%s)",
                    scope.deptAlias(), CollUtil.join(deptIds, ",")));
                break;
            case 4: // 仅本人
                condition.append(String.format(" AND %s.create_user = %d",
                    scope.userAlias(), user.getUserId()));
                break;
            case 5: // 自定义
                List<Long> customDeptIds = roleService.getDataScopeDeptIds(user.getRoleId());
                condition.append(String.format(" AND %s.dept_id IN (%s)",
                    scope.deptAlias(), CollUtil.join(customDeptIds, ",")));
                break;
        }

        // 在 SQL 的 WHERE 子句后面追加条件
        if (sql.contains("WHERE")) {
            return sql.replace("WHERE", "WHERE 1=1 " + condition + " AND");
        } else {
            // 没有 WHERE 子句的情况
            return sql + " WHERE 1=1 " + condition;
        }
    }
}
```

### 4. 使用方式

```java
// Mapper 方法上加注解
@DataScope(deptAlias = "d", userAlias = "u")
List<OrderVO> selectOrderList(OrderQueryDTO query);

// 对应 SQL
// SELECT o.* FROM orders o LEFT JOIN department d ON o.dept_id = d.id
// WHERE o.status = #{query.status}
// [DataScopeInterceptor 自动追加] AND d.dept_id = #{currentDeptId}
```

### 5. 多租户方案选择

| 方案 | 隔离级别 | 复杂度 | 成本 | 适用场景 |
|------|---------|--------|------|---------|
| **字段隔离**（tenant_id） | 行级 | 低 | 低 | SaaS 中小项目（**推荐起步方案**） |
| 独立 Schema | 库级 | 中 | 中 | 租户间数据需强隔离 |
| 独立数据库 | 实例级 | 高 | 高 | 大客户/合规要求 |

**字段隔离实现**（MyBatis 拦截器自动追加 tenant_id）：

```java
@Intercepts({
    @Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})
})
@Component
public class TenantInterceptor implements Interceptor {

    // 需要排除的表（全局共享数据）
    private static final Set<String> EXCLUDE_TABLES = Set.of(
        "sys_tenant", "sys_config", "sys_dict"
    );

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Long tenantId = TenantContext.getTenantId();  // ThreadLocal 存储当前租户
        if (tenantId == null) {
            return invocation.proceed();
        }

        StatementHandler handler = (StatementHandler) invocation.getTarget();
        String sql = handler.getBoundSql().getSql();

        // 检查是否涉及排除的表
        if (EXCLUDE_TABLES.stream().anyMatch(sql::contains)) {
            return invocation.proceed();
        }

        // 自动追加 tenant_id 条件
        String newSql = addTenantCondition(sql, tenantId);
        MetaObject metaObject = SystemMetaObject.forObject(handler);
        metaObject.setValue("delegate.boundSql.sql", newSql);

        return invocation.proceed();
    }
}
```

---

## 三、禁止事项

- **禁止直接使用用户上传的文件名**（路径穿越攻击：`../../etc/passwd`）
- **禁止信任前端传的 Content-Type**（必须校验文件头魔数）
- **禁止一次性读取大文件到内存**（用流式读取，buffer 8KB）
- **禁止文件保存到项目 jar 包目录**（会被覆盖/删除，用外部目录或 OSS）
- **禁止 SQL 中硬编码租户/用户 ID**（用拦截器自动追加）
- **禁止遗漏数据权限过滤**（Mapper 查询方法必须检查是否需要 @DataScope）
- **禁止在应用服务器本地存大量文件**（超过 1000 个文件就该上 OSS）
- **禁止关闭文件类型的白名单校验**（即使"只有内部使用"也要校验）
