# xxl-job-tenant

本仓库用于承载基于 `xuxueli/xxl-job` 最新版本的多租户改造方案。当前参考的上游版本为 **XXL-JOB v3.4.0**（2026-04 发布，JDK 17+）。由于当前执行环境无法通过 Git/HTTP 直接拉取 GitHub/Gitee/SourceForge 源码包，本次提交先落地可编译的核心领域模型、服务层与数据库迁移脚本，便于后续把同样的租户边界补丁合入完整 `xxl-job-admin` 页面与 Mapper。

## 改造目标

1. **支持增加租户、增加租户下用户**：新增 `TenantAdminService`，以 `tenantCode + username` 作为用户唯一身份。
2. **按租户登录**：新增 `TenantLoginService`，登录参数包含 `tenantCode`、`username`、`password`，登录态携带租户编码。
3. **一套注册器和执行器服务多租户**：`SharedExecutorRegistryService` 保持执行器注册全局共享；`TenantJobService` 只隔离任务数据；调度时由 `SchedulerDispatchService` 将 `tenantCode` 写入触发 payload，使共享执行器可按租户参数执行不同业务任务。

## 设计映射到 XXL-JOB

| XXL-JOB 模块 | 多租户改造点 |
| --- | --- |
| `xxl_job_user` | 增加 `tenant_id`，登录查询增加租户条件，允许不同租户使用相同用户名。 |
| `xxl_job_info` | 增加 `tenant_id`，任务列表、增删改查、启动停止、手动触发均追加租户过滤。 |
| `xxl_job_log` / `xxl_job_logglue` | 增加 `tenant_id`，日志查询和 GLUE 版本按租户隔离。 |
| `xxl_job_registry` / 执行器 | 不增加租户字段，作为全局共享资源；租户通过创建不同 Job 并选择相同 `executorAppName` 复用执行器。 |
| 调度触发参数 | 每次触发将 `tenantCode` / `tenantId` 写入 payload，执行器侧 Handler 根据参数路由到租户业务配置。 |

## 数据库脚本

执行 `doc/db/tables_xxl_job_tenant.sql` 可在上游 XXL-JOB 表结构基础上增加租户表、用户租户字段、任务租户字段与日志租户索引。

## 本地验证

```bash
javac -d target/classes $(find src/main/java -name '*.java')
java -cp target/classes com.xxl.job.tenant.TenantXxlJobApplication
javac -cp target/classes -d target/test-classes $(find src/test/java -name '*.java')
java -cp target/classes:target/test-classes com.xxl.job.tenant.TenantIsolationTest
```

## 后续合入完整 XXL-JOB 管理端时的补丁清单

- `LoginController#doLogin`：增加 `tenantCode` 参数，调用 `XxlJobUserMapper.loadByTenantAndUsername`。
- `XxlJobUser`：增加 `tenantId` 字段；用户管理页面新增租户筛选和创建入口。
- `XxlJobInfoMapper`、`XxlJobLogMapper`、`XxlJobLogGlueMapper`：所有查询/更新/删除语句追加 `tenant_id = 当前登录租户`。
- `JobTrigger`：构造触发请求时追加租户上下文参数；保持 `job_group` / executor registry 的全局共享策略。
- `JobGroupController`：执行器分组可作为共享分组展示，普通租户只可选择，不可删除全局注册节点。
