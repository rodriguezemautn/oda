# Módulo 8: Proyecto Final - Implementación Integral ODA

## 8.1 Descripción del Proyecto

### Escenario
Implementación de un entorno de alta disponibilidad para una empresa con:
- 2 bases de datos OLTP (Producción)
- 1 base de datos DW (Business Intelligence)
- 3 VMs para aplicaciones
- DataGuard para BD críticas
- Monitorización 24/7

### Requerimientos Técnicos
```plaintext
1. Hardware
   - ODA X8-2
   - 384GB RAM
   - 24 cores
   - 10TB Storage

2. Software
   - Oracle 19c
   - OL7/OL8
   - Enterprise Manager
   - Oracle GoldenGate
```

## 8.2 Plan de Implementación

### Fase 1: Configuración Base
```bash
# 1. Configuración de Red
odacli create-network \
    --name prod_public \
    --domain prod.company.com \
    --ip 192.168.10.10 \
    --netmask 255.255.255.0 \
    --gateway 192.168.10.1 \
    --type PUBLIC

odacli create-network \
    --name prod_private \
    --domain prod.internal \
    --ip 10.0.0.10 \
    --netmask 255.255.255.0 \
    --type PRIVATE

# 2. Storage Layout
odacli create-diskgroup \
    --name DATA \
    --redundancy HIGH \
    --disks /dev/sde,/dev/sdf

odacli create-diskgroup \
    --name RECO \
    --redundancy HIGH \
    --disks /dev/sdg,/dev/sdh
```

### Fase 2: Bases de Datos
```bash
# OLTP Production
odacli create-database \
    --name OLTP1 \
    --dbVersion 19.12.0.0 \
    --dbClass OLTP \
    --dbShape odb2 \
    --characterSet AL32UTF8 \
    --totalSize 2T

# DataWarehouse
odacli create-database \
    --name DWDB \
    --dbVersion 19.12.0.0 \
    --dbClass DSS \
    --dbShape odb4 \
    --characterSet AL32UTF8 \
    --totalSize 5T
```

### Fase 3: DataGuard
```bash
# Configuración DataGuard
odacli configure-dataguard \
    --primary primary.prod.com \
    --standby standby.prod.com \
    --type physical \
    --protection-mode MAX_AVAILABILITY \
    --transport-type SYNC \
    --database OLTP1
```

### Fase 4: Virtualización
```bash
# Creación VMs
odacli create-vm \
    --name appserver1 \
    --memory 32 \
    --vcpu 4 \
    --os_type OL8 \
    --disk_size 200G \
    --network "['prod_public','prod_private']"

odacli create-vm \
    --name appserver2 \
    --memory 32 \
    --vcpu 4 \
    --os_type OL8 \
    --disk_size 200G \
    --network "['prod_public','prod_private']"
```

## 8.3 Monitorización y Administración

### Sistema de Monitoreo
```python
# monitor_production.py
import cx_Oracle
import json
import time
from datetime import datetime

class ODAMonitor:
    def __init__(self):
        self.databases = ['OLTP1', 'DWDB']
        self.metrics = {}
    
    def collect_db_metrics(self):
        for db in self.databases:
            conn = cx_Oracle.connect(f'/ as sysdba', encoding='UTF-8')
            cursor = conn.cursor()
            
            # Performance metrics
            cursor.execute("""
                SELECT metric_name, value
                FROM v$sysmetric
                WHERE group_id = 2
                AND metric_name IN ('CPU Usage Per Sec',
                                  'Buffer Cache Hit Ratio',
                                  'User Transaction Per Sec')
            """)
            
            self.metrics[db] = cursor.fetchall()
            cursor.close()
            conn.close()
    
    def collect_system_metrics(self):
        cmd = "odacli describe-system"
        # Implementation details
        
    def generate_report(self):
        report = {
            'timestamp': datetime.now().isoformat(),
            'databases': self.metrics,
            'system': self.system_metrics
        }
        return report

# Implementación
monitor = ODAMonitor()
while True:
    monitor.collect_db_metrics()
    monitor.collect_system_metrics()
    report = monitor.generate_report()
    
    with open('/var/log/oda/production_metrics.json', 'a') as f:
        json.dump(report, f)
    
    time.sleep(300)
```

### Alertas y Notificaciones
```python
# alert_manager.py
def check_thresholds(metrics):
    alerts = []
    thresholds = {
        'CPU Usage Per Sec': 80,
        'Buffer Cache Hit Ratio': 85,
        'Disk Usage': 85
    }
    
    for metric, value in metrics.items():
        if metric in thresholds and value > thresholds[metric]:
            alerts.append({
                'metric': metric,
                'value': value,
                'threshold': thresholds[metric],
                'timestamp': datetime.now().isoformat()
            })
    
    return alerts
```

## 8.4 Backup y Recovery

### Estrategia de Backup
```bash
# Configuración general
odacli create-backupconfig \
    --name prod_backup \
    --location /backup \
    --policy LONG_TERM \
    --schedule daily \
    --level FULL,ARCHIVELOG

# Backup específico OLTP
odacli create-backup \
    --database OLTP1 \
    --level FULL \
    --tag monthly_full

# Backup específico DW
odacli create-backup \
    --database DWDB \
    --level FULL \
    --tag monthly_full
```

## 8.5 Documentación y Entregables

### Documentación Técnica
```markdown
# 1. Arquitectura
- Diagrama de infraestructura
- Network layout
- Storage configuration
- Security implementation

# 2. Procedimientos Operativos
- Daily checks
- Backup/recovery
- Patching
- Emergency procedures

# 3. Monitorización
- Métricas clave
- Thresholds
- Alerting workflow
- Escalation procedures
```

### Scripts y Herramientas
```python
# Estructura del repositorio
/scripts
    /monitoring
        monitor_production.py
        alert_manager.py
    /backup
        backup_verify.sh
        cleanup_old_backups.sh
    /maintenance
        patch_check.py
        generate_reports.py
```

## 8.6 Plan de Contingencia

### Disaster Recovery
```bash
# Procedimientos DR
1. Failover DataGuard
odacli failover-dataguard \
    --database OLTP1 \
    --standby STDBY

2. Recovery desde backup
odacli recover-database \
    --database OLTP1 \
    --from-backup \
    --latest

3. VM Recovery
odacli restore-vm \
    --name appserver1 \
    --snapshot snap_latest
```

## 8.7 Criterios de Evaluación

### Aspectos Técnicos
```plaintext
1. Alta Disponibilidad
   - RPO/RTO cumplidos
   - Failover exitoso
   - Recuperación verificada

2. Performance
   - Tiempo respuesta < 1s
   - CPU utilization < 70%
   - I/O throughput óptimo

3. Monitorización
   - Alertas funcionando
   - Reportes automáticos
   - Dashboard operativo
```

### Documentación
```plaintext
1. Technical Design Document
2. Operational Procedures
3. Disaster Recovery Plan
4. Monitoring Setup
5. Security Implementation
```

## 8.8 Referencias y Recursos

### Documentación Oracle
1. [ODA Documentation](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/)
2. [Database 19c](https://docs.oracle.com/en/database/oracle/oracle-database/19/)
3. [DataGuard](https://docs.oracle.com/en/database/oracle/oracle-database/19/sbydb/)

### MOS Notes
- Note 2144642.1: ODA Best Practices
- Note 2521012.1: ODA Troubleshooting
- Note 2558075.1: ODA Performance Guide