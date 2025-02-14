# Módulo 3: ODACLI (Oracle Database Appliance Command Line Interface)

## 3.1 Fundamentos ODACLI

### Arquitectura
```bash
# Ubicación de binarios
/opt/oracle/dcs/bin/odacli           # Ejecutable principal
/opt/oracle/dcs/conf/odacli.conf     # Configuración
/var/opt/oracle/log/odacli/          # Logs

# Estructura de comandos
odacli [comando] [objeto] [opciones]
```

### Autenticación y Permisos
```bash
# Usuarios con acceso
root           # Acceso total
oracle         # Acceso limitado a DB
grid           # Acceso a Grid/ASM

# Configuración sudo
# /etc/sudoers.d/odacli
oracle ALL=(root) NOPASSWD: /opt/oracle/dcs/bin/odacli
```

## 3.2 Comandos Fundamentales

### Sistema y Hardware
```bash
# Información del sistema
odacli describe-system
odacli list-cpucores
odacli describe-component

# Estado y diagnóstico
odacli validate-system-patch
odacli show-server-patch
odacli inspect-system

# Ejemplos prácticos
odacli describe-system -j # Salida en JSON
odacli describe-component -n NetworkInterface # Solo interfaces de red
```

### Gestión de Bases de Datos
```bash
# Listado y descripción
odacli list-databases
odacli describe-database -i {db_id}
odacli describe-dbhome -i {dbhome_id}

# Creación de BD
odacli create-database \
    -n ORCL \
    -u TESTDB \
    -cp 12.2.0.1 \
    -c OLTP \
    -cs AL32UTF8 \
    -sl 2 \
    -nn RAC \
    -p "MyDBPassw0rd"

# Modificación
odacli modify-database \
    -i {database_id} \
    -m 8G \
    -r 2

# Eliminación
odacli delete-database -i {database_id}
```

### Gestión de Storage
```bash
# ASM y Diskgroups
odacli list-diskgroups
odacli describe-diskgroup -g DATA
odacli create-diskgroup \
    -n FLASHDG \
    -d /dev/sdg,/dev/sdh \
    -r HIGH

# Espacio y performance
odacli describe-storage
odacli storage analyze
odacli validate-storageconfiguration
```

### Networking
```bash
# Configuración de red
odacli list-networks
odacli describe-network
odacli create-network \
    -n publicnet \
    -i eth0 \
    -t PUBLIC \
    -ip 192.168.1.10 \
    -nm 255.255.255.0 \
    -gw 192.168.1.1

# SCAN y VIP
odacli configure-scan \
    -sn oda-scan \
    -ip 192.168.1.20,192.168.1.21
```

## 3.3 Operaciones Avanzadas

### Backup y Recovery
```bash
# Configuración de backup
odacli create-backupconfig \
    -d NFS \
    -n backup_config1 \
    -w 7 \
    -cl ARCHIVE \
    -ln /backup/nfs \
    -pr BASIC

# Ejecución de backups
odacli create-backup \
    -i {database_id} \
    -bn backup_full_$(date +%Y%m%d) \
    -bt REGULAR-L0

# Recovery
odacli recover-database \
    -i {database_id} \
    -br {backup_report_id} \
    -tp LATEST \
    -sl FULL
```

### Patching y Actualizaciones
```bash
# Análisis de parches
odacli list-availablepatches
odacli describe-patch \
    -i {patch_id}

# Aplicación de parches
odacli create-prepatchreport \
    -s {patch_version}
odacli update-repository \
    -f /path/to/patch.zip
odacli update-server \
    -v {patch_version}

# Verificación
odacli describe-job \
    -i {job_id}
odacli validate-storageconfiguration
```

### Alta Disponibilidad
```bash
# Configuración DataGuard
odacli configure-dataguard \
    -ps PRIMARY \
    -primary hostname \
    -standby hostname2 \
    -sl SYNC \
    -bt FULL \
    -u sys \
    -p password123

# Switchover/Failover
odacli switchover-dataguard \
    -i {database_id} \
    -sn {standby_name}

odacli failover-dataguard \
    -i {database_id} \
    -sn {standby_name}
```

## 3.4 Monitorización y Troubleshooting

### Monitoreo de Recursos
```bash
# CPU y Memoria
odacli show-server-memory
odacli show-server-cpu

# Storage
odacli storage analyze
odacli show-storage-history

# Ejemplos de scripts de monitoreo
#!/bin/bash
# monitor_resources.sh

echo "=== CPU Usage ==="
odacli show-server-cpu | grep "CPU_USAGE"

echo "=== Memory Stats ==="
odacli show-server-memory | grep "MEMORY"

echo "=== Storage Usage ==="
odacli storage analyze | grep "USED"
```

### Gestión de Logs
```bash
# Ubicaciones principales
/var/opt/oracle/log/odacli/     # Logs ODACLI
/var/opt/oracle/log/dcs/        # Logs DCS
/opt/oracle/dcs/log/            # Logs adicionales

# Comandos de logs
odacli list-jobs               # Historial de trabajos
odacli describe-job -i {id}    # Detalle de trabajo
odacli get-job-status -i {id}  # Estado de trabajo

# Rotación de logs
# /etc/logrotate.d/odacli
/var/opt/oracle/log/odacli/*.log {
    weekly
    rotate 12
    compress
    missingok
    notifempty
}
```

### Diagnóstico y Resolución
```bash
# Herramientas de diagnóstico
odacli validate-system
odacli create-diagnostics
odacli collect-diagnostics

# Troubleshooting común
## Problema: Fallo en creación de BD
odacli describe-job -i {failed_job_id}
cat /var/opt/oracle/log/odacli/dcs-agent.log

## Problema: Problemas de red
odacli validate-network
odacli show-network-failures

## Problema: Storage
odacli validate-storageconfiguration
odacli storage analyze
```

## 3.5 Automatización y Scripting

### Scripts de Automatización
```bash
#!/bin/bash
# create_test_db.sh

# Variables
DB_NAME="TESTDB_$(date +%Y%m%d)"
SYS_PWD="Welcome1"
CHARSET="AL32UTF8"
DB_CLASS="OLTP"
SHAPE="odb1"

# Crear base de datos
create_database() {
    odacli create-database \
        -n "$DB_NAME" \
        -u "$DB_NAME" \
        -cp 19.12.0.0.0 \
        -c "$DB_CLASS" \
        -cs "$CHARSET" \
        -sl "$SHAPE" \
        -p "$SYS_PWD"
}

# Verificar creación
verify_database() {
    odacli list-databases | grep "$DB_NAME"
    if [ $? -eq 0 ]; then
        echo "Database $DB_NAME created successfully"
    else
        echo "Database creation failed"
        exit 1
    fi
}

# Ejecutar
create_database
verify_database
```

### API REST Integration
```python
# odacli_rest.py
import requests
import json

class ODACLI_API:
    def __init__(self, host, port=7093):
        self.base_url = f"https://{host}:{port}/api"
        self.token = None
    
    def login(self, username, password):
        url = f"{self.base_url}/auth"
        data = {
            "username": username,
            "password": password
        }
        response = requests.post(url, json=data, verify=False)
        self.token = response.json()["token"]
    
    def list_databases(self):
        url = f"{self.base_url}/database/list"
        headers = {"Authorization": f"Bearer {self.token}"}
        response = requests.get(url, headers=headers, verify=False)
        return response.json()

    def create_database(self, db_config):
        url = f"{self.base_url}/database/create"
        headers = {"Authorization": f"Bearer {self.token}"}
        response = requests.post(url, 
                               headers=headers, 
                               json=db_config, 
                               verify=False)
        return response.json()
```

## 3.6 Mejores Prácticas

### Seguridad
```bash
# 1. Gestión de credenciales
## Usar wallet para almacenar credenciales
orapki wallet create -wallet /opt/oracle/dcs/conf/.wallet -pwd wallet_password
mkstore -wrl /opt/oracle/dcs/conf/.wallet -createCredential ORCL sys password

# 2. Hardening de permisos
chmod 600 /opt/oracle/dcs/conf/odacli.conf
chown oracle:oinstall /opt/oracle/dcs/conf/odacli.conf

# 3. Auditoría
## Habilitar auditoría extendida
odacli update-configuration --name AuditLevel --value ALL
```

### Performance
```bash
# 1. Optimización de jobs
## Limitar jobs concurrentes
odacli update-configuration --name MaxConcurrentJobs --value 5

# 2. Gestión de recursos
## Monitoreo regular
odacli show-server-memory -j > /var/log/memory_$(date +%Y%m%d).log
odacli show-server-cpu -j > /var/log/cpu_$(date +%Y%m%d).log

# 3. Mantenimiento
## Limpieza regular de logs
find /var/opt/oracle/log/odacli -name "*.log" -mtime +30 -delete
```

## 3.7 Referencias y Documentación

### Oracle Documentation
1. [ODACLI Command Reference](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/odacl/)
2. [ODACLI API Guide](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/rest-api/)

### MOS Notes
- Note 2144642.1: ODACLI Command Reference Guide
- Note 2521012.1: ODACLI Troubleshooting Guide
- Note 2558075.1: Common ODACLI Issues and Solutions

### Blogs y Recursos
1. Oracle Database Insider
   - "ODACLI Best Practices"
   - "Automation with ODACLI"

2. Oracle DBA Blog
   - "ODACLI Performance Tuning"
   - "Advanced ODACLI Scripts"

### Scripts de Ejemplo
```bash
# GitHub Repositories
https://github.com/oracle/oda-scripts
https://github.com/oracle/db-scripts
```