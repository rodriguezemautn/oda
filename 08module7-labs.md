# Módulo 7: Prácticas y Laboratorios

## 7.1 Laboratorio: Configuración Inicial ODA

### Lab 1: Verificación Post-Instalación
```bash
# 1. Validación del sistema
odacli validate-system
odacli describe-system

# 2. Verificación de componentes
odacli describe-component
odacli list-cpucores

# 3. Revisión de logs
tail -f /var/log/oracle/oak/dcs-agent.log
```

### Lab 2: Configuración de Red
```bash
# 1. Configurar red pública
odacli create-network \
    --name public_network \
    --domain example.com \
    --ip 192.168.1.10 \
    --netmask 255.255.255.0 \
    --gateway 192.168.1.1 \
    --type PUBLIC

# 2. Configurar red privada
odacli create-network \
    --name private_network \
    --domain internal.local \
    --ip 10.0.0.10 \
    --netmask 255.255.255.0 \
    --type PRIVATE
```

## 7.2 Laboratorio: Gestión de Bases de Datos

### Lab 3: Creación y Configuración de BD
```bash
# 1. Crear base de datos
odacli create-database \
    --name PRODDB \
    --dbVersion 19.12.0.0 \
    --dbClass OLTP \
    --dbShape odb1 \
    --sysPassword Welcome1 \
    --characterSet AL32UTF8

# 2. Configurar parámetros
sqlplus / as sysdba <<EOF
ALTER SYSTEM SET memory_target = 8G SCOPE=SPFILE;
ALTER SYSTEM SET sga_target = 6G SCOPE=SPFILE;
ALTER SYSTEM SET pga_aggregate_target = 2G SCOPE=SPFILE;
EOF
```

### Lab 4: Backup y Recovery
```bash
# 1. Configurar backup
odacli create-backupconfig \
    --name daily_backup \
    --location /u01/backup \
    --policy BASIC \
    --schedule daily

# 2. Realizar backup manual
odacli create-backup \
    --database PRODDB \
    --level FULL

# 3. Simular recovery
odacli recover-database \
    --database PRODDB \
    --latest
```

## 7.3 Laboratorio: DataGuard

### Lab 5: Configuración DataGuard
```bash
# 1. Preparar base de datos primaria
sqlplus / as sysdba <<EOF
ALTER DATABASE FORCE LOGGING;
ALTER DATABASE FLASHBACK ON;
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(proddb,stdbydb)';
EOF

# 2. Configurar DataGuard
odacli configure-dataguard \
    --primary primary.example.com \
    --standby standby.example.com \
    --type physical \
    --protection-mode MAX_AVAILABILITY \
    --transport-type SYNC \
    --database PRODDB \
    --password Welcome1
```

### Lab 6: Operaciones DataGuard
```bash
# 1. Verificar configuración
dgmgrl sys/Welcome1@PRODDB <<EOF
SHOW CONFIGURATION;
SHOW DATABASE VERBOSE 'PRODDB';
EOF

# 2. Realizar switchover
odacli switchover-dataguard \
    --database PRODDB \
    --standby STDBYDB
```

## 7.4 Laboratorio: Virtualización

### Lab 7: Gestión de VMs
```bash
# 1. Crear VM
odacli create-vm \
    --name webserver1 \
    --memory 4 \
    --vcpu 2 \
    --os_type OL7U9 \
    --disk_size 50G \
    --network "['pub1']"

# 2. Gestionar storage
odacli create-vdisk \
    --vdisk_name data_disk1 \
    --vdisk_size 100G
odacli modify-vm \
    --name webserver1 \
    --attach_vdisk data_disk1
```

### Lab 8: Network y HA
```bash
# 1. Configurar red virtual
odacli create-vnetwork \
    --name app_network \
    --bridge bridge1 \
    --vlan-id 100

# 2. Configurar HA
odacli configure-vmha \
    --name webserver1 \
    --priority HIGH
```

## 7.5 Laboratorio: Monitorización

### Lab 9: Configuración de Monitoreo
```python
# monitor_oda.py
import subprocess
import json
import time

def get_system_metrics():
    metrics = {
        'system': subprocess.getoutput('odacli describe-system'),
        'storage': subprocess.getoutput('odacli describe-storage'),
        'databases': subprocess.getoutput('odacli list-databases')
    }
    return metrics

def log_metrics(metrics):
    with open('/var/log/oda/metrics.log', 'a') as f:
        json.dump({
            'timestamp': time.time(),
            'metrics': metrics
        }, f)

while True:
    metrics = get_system_metrics()
    log_metrics(metrics)
    time.sleep(300)
```

### Lab 10: Alertas y Reportes
```bash
# 1. Configurar alertas
odacli create-alert-rule \
    --name DiskSpace \
    --description "Disk Usage > 85%" \
    --metric DISK_USAGE \
    --threshold 85 \
    --comparison-operator GREATER_THAN \
    --notification-type EMAIL

# 2. Generar reportes
#!/bin/bash
# generate_report.sh
echo "=== ODA Health Report ===" > report.txt
odacli describe-system >> report.txt
echo "\n=== Storage Status ===" >> report.txt
odacli describe-storage >> report.txt
echo "\n=== Database Status ===" >> report.txt
odacli list-databases >> report.txt
```

## 7.6 Evaluación y Entregables

### Criterios de Evaluación
```plaintext
1. Configuración correcta de componentes
2. Implementación de seguridad
3. Eficiencia en operaciones
4. Documentación de procedimientos
5. Resolución de problemas
```

### Entregables
```plaintext
1. Documentación de configuración
2. Scripts de automatización
3. Reportes de monitoreo
4. Plan de backup/recovery
5. Procedimientos de mantenimiento
```

## 7.7 Escenarios de Troubleshooting

### Escenario 1: Problemas de Performance
```sql
-- 1. Analizar waitevents
SELECT event, count(*) 
FROM v$active_session_history 
WHERE sample_time > SYSDATE - 1/24
GROUP BY event
ORDER BY count(*) DESC;

-- 2. Identificar SQL problemático
SELECT sql_id, elapsed_time/1000000 seconds,
       executions, elapsed_time/1000000/executions avg_seconds
FROM v$sql 
WHERE elapsed_time > 1000000
ORDER BY elapsed_time DESC;
```

### Escenario 2: Fallo de Storage
```bash
# 1. Verificar estado
odacli validate-storageconfiguration

# 2. Analizar ASM
asmcmd lsdg
asmcmd iostat -G

# 3. Revisar alertas
adrci exec="show alert -p 'message_text like '%ORA-15%'"
```

## 7.8 Tips y Mejores Prácticas

### Optimización
```bash
# 1. Performance
- Monitoreo regular de métricas
- Ajuste de parámetros basado en workload
- Mantenimiento proactivo

# 2. Seguridad
- Implementar control de acceso
- Auditoría de operaciones críticas
- Actualizaciones regulares
```

### Documentación
```markdown
# Template de Documentación
## Configuración
- Parámetros del sistema
- Network setup
- Storage layout

## Procedimientos
- Backup/Recovery
- Patching
- Troubleshooting

## Monitoreo
- Métricas clave
- Umbrales de alertas
- Procedimientos de escalación
```