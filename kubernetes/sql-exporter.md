# 安装

## 1. 准备配置文件

```yaml
databases:
  oracle:
    dsn: oracle+cx_oracle://prometheus:prom123@3.1.20.25:1521/orcl

metrics:
  oracle_sessions:
    type: gauge
    description: Number of sessions
    labels: [status, type]
  oracle_resouce_current_utilization:
    type: counter
    description: Current value from v$resource_limit view
    labels: [name]
  oracle_resource_limit:
    type: counter
    description: Limit value from v$resource_limit view
    labels: [name]
  oracle_asm_diskgroup_total:
    type: counter
    description: Total size of ASM disk group
    labels: [name]
  oracle_asm_diskgroup_free:
    type: counter
    description: Free space on ASM disk group
    labels: [name]
  oracle_activity:
    type: counter
    labels: [name]
    description: Number of v$sysstat view

queries:
  sessions_stats:
    databases: [oracle]
    metrics:
      - oracle_sessions
    sql: >
      SELECT
        status,
        type,
        COUNT(*) AS oracle_sessions
      FROM v$session
      GROUP BY status, type
  resources_stats:
    databases: [oracle]
    metrics:
      - oracle_resouce_current_utilization
      - oracle_resource_limit
    sql: >
      SELECT
        resource_name AS name
        current_utilization AS oracle_resouce_current_utilization
        limit_value AS oracle_resource_limit
      FROM v$resource_limit
  asm_diskgroup_stats:
    databases: [oracle]
    metrics:
      - oracle_asm_diskgroup_total
      - oracle_asm_diskgroup_free
    sql: >
      SELECT
        name,
        total_mb * 1024 * 1024 AS oracle_asm_diskgroup_total,
        free_mb * 1024 * 1024 AS oracle_asm_diskgroup_free
      FROM v$asm_diskgroup
  activity_stats:
    databases: [oracle]
    metrics:
      - oracle_activity
    sql: >
      SELECT
        name,
        value AS oracle_activity
      FROM v$sysstat
      WHERE name IN ('parse count (total)', 'execute count', 'user commits', 'user rollbacks')

```

## 2. 启动exporter

```bash
 docker run -d -p 9560:9560/tcp -v "/root/oracle/config.yaml:/config.yaml"  --name oracle_exporter adonato/query-exporter:latest
 ```

 ## 3. 访问

 ```bash
 curl http://3.1.20.108:9560/metrics
 ```

 > 其他数据源参考`https://github.com/albertodonato/query-exporter`