# Módulo 11: Performance Tuning Avanzado en ODA

## 11.1 Optimización Específica ODA

### Análisis de Recursos Base
```sql
-- Monitoreo de recursos del sistema
SELECT metric_name, value, metric_unit
FROM v$sysmetric
WHERE group_id = 2
AND metric_name IN (
    'CPU Usage Per Sec',
    'Database CPU Time Ratio',
    'Database Wait Time Ratio'
);

-- Análisis de espera
SELECT event, 
       total_waits,
       time_waited_micro/1000000 time_waited_seconds,
       ROUND(time_waited_micro*100/SUM(time_waited_micro) 
         OVER (), 2) pct
FROM v$system_event
WHERE wait_class != 'Idle'
ORDER BY time_waited_micro DESC;
```

### Optimización de Memoria
```sql
-- Configuración SGA
ALTER SYSTEM SET sga_target = 80G SCOPE=SPFILE;
ALTER SYSTEM SET sga_max_size = 100G SCOPE=SPFILE;

-- Configuración PGA
ALTER SYSTEM SET pga_aggregate_target = 40G SCOPE=SPFILE;
ALTER SYSTEM SET memory_target = 120G SCOPE=SPFILE;

-- Monitoreo de memoria
SELECT pool, 
       round(bytes/1024/1024/1024,2) size_gb,
       round(bytes_free/1024/1024/1024,2) free_gb,
       round((bytes_free/bytes)*100,2) pct_free
FROM v$sgastat
WHERE pool IS NOT NULL
GROUP BY pool;
```

### Optimización I/O
```sql
-- Configuración ASM
ALTER DISKGROUP DATA SET ATTRIBUTE 'au_size' = '4M';
ALTER DISKGROUP DATA SET ATTRIBUTE 'compatible.asm' = '19.0.0.0.0';

-- Monitoreo I/O
SELECT a.name disk_group,
       b.path,
       b.reads,
       b.writes,
       b.read_time,
       b.write_time
FROM v$asm_diskgroup a, v$asm_disk b
WHERE a.group_number = b.group_number;

-- Configuración Smart I/O
ALTER SYSTEM SET "_disk_sector_size_override"=TRUE SCOPE=SPFILE;
ALTER SYSTEM SET filesystemio_options='SETALL' SCOPE=SPFILE;
```

## 11.2 Afinamiento de Recursos Virtualizados

### KVM Performance
```bash
# CPU Pinning
virsh vcpupin VMNAME 0 0-1
virsh vcpupin VMNAME 1 2-3

# Memory Config
virsh setmem VMNAME 16G
virsh memtune VMNAME --hard-limit 16G

# Network Tuning
virsh domiftune VMNAME vnet0 --inbound 1000,1500,2000
virsh domiftune VMNAME vnet0 --outbound 1000,1500,2000
```

### Storage Performance
```sql
-- ASM para VMs
CREATE DISKGROUP VM_DATA NORMAL REDUNDANCY
  DISK '/dev/disk/by-path/pci-0000:00:1f.2-ata-1'
  ATTRIBUTE 'au_size'='4M',
           'compatible.asm'='19.0.0.0.0',
           'compatible.rdbms'='19.0.0.0.0';

-- Monitoreo VM I/O
SELECT * FROM v$iostat_file 
WHERE filetype_name = 'VirtualDisk';
```

### Resource Management
```sql
-- Resource Plan para VMs
BEGIN
DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA();
DBMS_RESOURCE_MANAGER.CREATE_PLAN(
  plan => 'VM_RESOURCE_PLAN',
  comment => 'Resource plan for virtualized environment');

DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
  consumer_group => 'VM_CRITICAL',
  comment => 'Critical VMs');

DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
  consumer_group => 'VM_NORMAL',
  comment => 'Normal VMs');

DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
  plan => 'VM_RESOURCE_PLAN',
  group_or_subplan => 'VM_CRITICAL',
  comment => 'Critical VMs get 70% of CPU',
  mgmt_p1 => 70);

DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
  plan => 'VM_RESOURCE_PLAN',
  group_or_subplan => 'VM_NORMAL',
  comment => 'Normal VMs get 30% of CPU',
  mgmt_p1 => 30);

DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA();
DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA();
END;
/
```

## 11.3 Best Practices para Cargas Mixtas

### Workload Management
```sql
-- Configuración de Resource Manager
CREATE_PENDING_AREA();

-- Plan para cargas mixtas
DBMS_RESOURCE_MANAGER.CREATE_PLAN(
  plan => 'MIXED_WORKLOAD_PLAN',
  comment => 'Plan for OLTP and BATCH workloads');

-- Grupos de consumidores
DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
  consumer_group => 'ONLINE_GROUP',
  comment => 'Online transactions');

DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
  consumer_group => 'BATCH_GROUP',
  comment => 'Batch operations');

-- Directivas
DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
  plan => 'MIXED_WORKLOAD_PLAN',
  group_or_subplan => 'ONLINE_GROUP',
  comment => 'Online directive',
  mgmt_p1 => 75,
  parallel_degree_limit_p1 => 8,
  switch_group => 'BATCH_GROUP',
  switch_time => 3600);

DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
  plan => 'MIXED_WORKLOAD_PLAN',
  group_or_subplan => 'BATCH_GROUP',
  comment => 'Batch directive',
  mgmt_p1 => 25,
  parallel_degree_limit_p1 => 16);
```

### Parallel Execution
```sql
-- Configuración parallelismo
ALTER SYSTEM SET parallel_max_servers = 64;
ALTER SYSTEM SET parallel_min_servers = 16;
ALTER SYSTEM SET parallel_force_local = TRUE;

-- Monitoreo parallelismo
SELECT username, sql_id, sql_text, 
       degree, req_degree
FROM v$sql_monitor
WHERE command_name = 'SELECT'
AND px_servers_requested IS NOT NULL;
```

### Instance Caging
```sql
-- Configuración CPU
ALTER SYSTEM SET cpu_count = 16;
ALTER SYSTEM SET resource_manager_plan = 'MIXED_WORKLOAD_PLAN';

-- Monitoreo uso CPU
SELECT consumer_group_name,
       cpu_consumed_time,
       cpu_waits,
       yields
FROM v$rsrc_consumer_group;
```

## 11.4 Monitorización y Ajuste Continuo

### Performance Metrics
```sql
-- AWR Snapshots
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
END;
/

-- ASH Analysis
SELECT session_state, 
       session_type,
       COUNT(*) samples,
       COUNT(*) * 10 seconds
FROM v$active_session_history
WHERE sample_time > SYSDATE - 1/24
GROUP BY session_state, session_type;

-- SQL Monitoring
SELECT sql_id, 
       sql_plan_hash_value,
       buffer_gets,
       disk_reads,
       elapsed_time/1000000 elapsed_seconds
FROM v$sql
WHERE executions > 0
ORDER BY elapsed_time DESC;
```

### Automatic Tuning
```sql
-- Advisor Tasks
BEGIN
  DBMS_ADVISOR.CREATE_TASK(
    advisor_name => 'SQL Tuning Advisor',
    task_name => 'SYS_TUNING_TASK',
    task_desc => 'Tuning task for system SQL');
    
  DBMS_ADVISOR.SET_TASK_PARAMETER(
    task_name => 'SYS_TUNING_TASK',
    parameter => 'TIME_LIMIT',
    value => 3600);
    
  DBMS_ADVISOR.EXECUTE_TASK(
    task_name => 'SYS_TUNING_TASK');
END;
/

-- Implementar recomendaciones
SELECT rec_id, benefit
FROM dba_advisor_recommendations
WHERE task_name = 'SYS_TUNING_TASK';
```

### Performance Dashboard
```python
# dashboard.py
import cx_Oracle
import pandas as pd
import plotly.express as px

def get_performance_metrics():
    connection = cx_Oracle.connect('/ as sysdba')
    cursor = connection.cursor()
    
    metrics = {}
    
    # CPU Usage
    cursor.execute("""
        SELECT value 
        FROM v$sysmetric 
        WHERE metric_name = 'CPU Usage Per Sec'
        AND group_id = 2
    """)
    metrics['cpu_usage'] = cursor.fetchone()[0]
    
    # Memory Usage
    cursor.execute("""
        SELECT round(used_percent,2)
        FROM v$pgastat
        WHERE name = 'total PGA allocated'
    """)
    metrics['memory_usage'] = cursor.fetchone()[0]
    
    # I/O Throughput
    cursor.execute("""
        SELECT sum(physical_reads+physical_writes) throughput
        FROM v$filestat
    """)
    metrics['io_throughput'] = cursor.fetchone()[0]
    
    return metrics

def create_dashboard():
    metrics = get_performance_metrics()
    df = pd.DataFrame(metrics.items(), columns=['Metric', 'Value'])
    
    fig = px.bar(df, x='Metric', y='Value',
                 title='ODA Performance Dashboard')
    fig.show()
```

## 11.5 Best Practices y Recomendaciones

### Sistema Operativo
```bash
# Kernel Parameters
cat >> /etc/sysctl.conf << EOF
kernel.shmmax = 4398046511104
kernel.shmall = 1073741824
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
fs.aio-max-nr = 1048576
fs.file-max = 6815744
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
EOF

# Resource Limits
cat >> /etc/security/limits.conf << EOF
oracle soft nofile 1024
oracle hard nofile 65536
oracle soft nproc 2047
oracle hard nproc 16384
oracle soft stack 10240
oracle hard stack 32768
EOF
```

### Database Parameters
```sql
-- Parametros críticos
ALTER SYSTEM SET db_cache_size = 60G;
ALTER SYSTEM SET shared_pool_size = 20G;
ALTER SYSTEM SET large_pool_size = 2G;
ALTER SYSTEM SET java_pool_size = 1G;
ALTER SYSTEM SET streams_pool_size = 2G;
ALTER SYSTEM SET processes = 3000;
ALTER SYSTEM SET sessions = 3500;
ALTER SYSTEM SET open_cursors = 2000;
```

### Referencias y Documentación

1. Oracle Documentation
   - [ODA Performance Guide](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/performance/)
   - [Database Performance Tuning Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgdba/)

2. White Papers
   - "ODA Performance Best Practices"
   - "Virtualización y Performance en ODA"
   - "Resource Management para Cargas Mixtas"

3. MOS Notes
   - Note 2144642.1: ODA Performance Tuning
   - Note 2521012.1: ODA Resource Management
   - Note 2558075.1: Performance Best Practices