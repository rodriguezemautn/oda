# Módulo 6: Monitorización y Soporte en ODA

## 6.1 Herramientas de Monitorización

### Enterprise Manager Integration
```bash
# Configuración agente EM
/u01/app/oracle/product/agent13c/agent_13.5/bin/emctl status agent
/u01/app/oracle/product/agent13c/agent_13.5/bin/emctl upload agent

# Métricas principales
- DB Performance
- Storage Utilization
- Network Statistics
- Hardware Health
```

### Automatic Diagnostic Repository (ADR)
```bash
# Estructura ADR
$ORACLE_BASE/diag/
    rdbms/          # Database
    tnslsnr/        # Listener
    asmtool/        # ASM
    oda/            # Appliance

# Comandos ADRCI
adrci> show homes
adrci> set home diag/rdbms/orcl
adrci> show incident
adrci> show problem
```

## 6.2 Monitorización Avanzada

### Scripts de Monitoreo
```python
# oda_monitor.py
import subprocess
import json
import time

class ODAMonitor:
    def get_system_health(self):
        cmd = "odacli describe-system"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        return json.loads(result.stdout)
    
    def get_storage_stats(self):
        cmd = "odacli describe-storage"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        return json.loads(result.stdout)
    
    def monitor_databases(self):
        cmd = "odacli list-databases"
        result = subprocess.run(cmd.split(), capture_output=True, text=True)
        return json.loads(result.stdout)

# Implementación
monitor = ODAMonitor()
while True:
    health = monitor.get_system_health()
    storage = monitor.get_storage_stats()
    databases = monitor.monitor_databases()
    
    # Log results
    with open('/var/log/oda/monitoring.log', 'a') as f:
        json.dump({
            'timestamp': time.time(),
            'health': health,
            'storage': storage,
            'databases': databases
        }, f)
    
    time.sleep(300)  # 5 minutes interval
```

### Alertas y Notificaciones
```bash
# Configuración de alertas
odacli create-alert-rule \
    --name HighCPU \
    --description "CPU Usage > 80%" \
    --metric CPU_UTILIZATION \
    --threshold 80 \
    --comparison-operator GREATER_THAN \
    --notification-type EMAIL \
    --notification-target dba@company.com

# Alert History
odacli list-alerts
odacli describe-alert -i alert_id
```

## 6.3 Performance Monitoring

### AWR y Statspack
```sql
-- Generar AWR
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
END;
/

-- AWR Report
@?/rdbms/admin/awrrpt.sql

-- Statspack
@?/rdbms/admin/spcreate.sql
@?/rdbms/admin/spreport.sql
```

### ASH Analytics
```sql
-- Active Session History
SELECT session_id, session_serial#, 
       sql_id, event, wait_class
FROM V$ACTIVE_SESSION_HISTORY
WHERE sample_time > SYSDATE - 1/24
ORDER BY sample_time;

-- Top SQL por CPU
SELECT sql_id, 
       COUNT(*) samples,
       COUNT(*) * 10 seconds
FROM V$ACTIVE_SESSION_HISTORY
WHERE session_state = 'ON CPU'
GROUP BY sql_id
ORDER BY samples DESC;
```

## 6.4 Soporte y Mantenimiento

### Recolección de Diagnósticos
```bash
# Recopilar logs
odacli collect-diagnostics

# Trace específico
oradiag collect \
    --from "2024-02-01" \
    --to "2024-02-05" \
    --component database,asm,listener

# Analizar AWR
oradiag analyze-awr \
    --dbname ORCL \
    --from "2024-02-01" \
    --to "2024-02-05"
```

### Gestión de Parches
```bash
# Verificar parches disponibles
odacli list-availablepatches

# Análisis pre-patch
odacli create-prepatchreport \
    --dbhome-version 19.12.0.0.0

# Aplicar parches
odacli update-server \
    --version 19.12.0.0.0
```

## 6.5 Troubleshooting

### Problemas Comunes
```bash
# 1. Problemas de Storage
odacli validate-storageconfiguration
odacli storage analyze

# 2. Problemas de Red
odacli validate-network
odacli show-network-failures

# 3. Problemas de Base de Datos
odacli validate-database
sqlplus / as sysdba <<EOF
@?/rdbms/admin/health_check.sql
EOF
```

### Log Analysis
```bash
# Herramienta de análisis de logs
#!/bin/bash
# analyze_logs.sh

LOG_DIR="/var/log/oracle"
DAYS_TO_ANALYZE=7

find $LOG_DIR -type f -mtime -$DAYS_TO_ANALYZE -exec grep -H "ORA-" {} \; > errors.log
find $LOG_DIR -type f -mtime -$DAYS_TO_ANALYZE -exec grep -H "WARNING" {} \; > warnings.log

# Análisis de errores comunes
sort errors.log | uniq -c | sort -nr > error_summary.txt
```

## 6.6 Backup y Recovery

### Estrategia de Backup
```bash
# Configuración de backup
odacli create-backupconfig \
    --name daily_backup \
    --location /backup \
    --policy BASIC \
    --schedule daily \
    --level FULL,ARCHIVELOG

# Verificación de backups
odacli list-backups
odacli validate-backup
```

### Recovery Procedures
```bash
# Recovery completo
odacli recover-database \
    --database-name ORCL \
    --backup-id backup_20240205 \
    --latest

# Recovery point-in-time
odacli recover-database \
    --database-name ORCL \
    --backup-id backup_20240205 \
    --timestamp "2024-02-05 14:30:00"
```

## 6.7 Documentación y Procedimientos

### Templates
```markdown
# Incident Report Template
## Incident Details
- Date/Time:
- Severity:
- Components Affected:
- Impact:

## Root Cause Analysis
- Initial Symptoms:
- Investigation Steps:
- Root Cause:

## Resolution
- Actions Taken:
- Verification Steps:
- Prevention Measures:
```

### Runbooks
```bash
# Daily Checks
#!/bin/bash
# daily_checks.sh

echo "=== System Health ==="
odacli describe-system

echo "=== Storage Status ==="
odacli describe-storage

echo "=== Database Status ==="
odacli list-databases

echo "=== Recent Alerts ==="
odacli list-alerts --hours 24
```

## 6.8 Mejores Prácticas

### Monitoring
```plaintext
1. Establecer baseline de performance
2. Implementar monitoreo proactivo
3. Configurar alertas críticas
4. Mantener histórico de métricas
```

### Maintenance
```plaintext
1. Planificar ventanas de mantenimiento
2. Documentar cambios y procedimientos
3. Validar backups regularmente
4. Mantener parches al día
```

## 6.9 Referencias

### Oracle Documentation
1. [ODA Monitoring Guide](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/monitor/)
2. [ODA Administration Guide](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/admin/)

### MOS Notes
- Note 2144642.1: ODA Monitoring Best Practices
- Note 2521012.1: ODA Troubleshooting Guide
- Note 2558075.1: ODA Performance Tuning

### Scripts y Herramientas
```bash
# GitHub Repositories
https://github.com/oracle/oda-monitoring
https://github.com/oracle/oda-tools
```

### Comunidad y Recursos
1. Oracle Database Insider Blog
   - "ODA Performance Tuning"
   - "Monitoring Best Practices"

2. Oracle Support Community
   - ODA Technical Forum
   - Monitoring & Tuning Discussion