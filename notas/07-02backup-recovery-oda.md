# Backup y Recovery en ODA

## 1. Destinos de Backup Soportados

### Tipos de Destino
```plaintext
1. FRA (Fast Recovery Area)
   - Almacenamiento en disco local
   - Gestión automatizada de espacio
   - Retención configurable

2. Object Store
   - Almacenamiento en la nube
   - Escalabilidad flexible
   - Retención a largo plazo

3. NFS Mount
   - Almacenamiento en red
   - Soporte para alta disponibilidad
   - Configuración flexible
```

## 2. Configuración NFS

### Requisitos
```bash
# Mount points en HA
mount -t nfs nfs_server:/backup /backup

# Permisos
chown oracle:oinstall /backup
chmod 755 /backup

# Verificar acceso
su - oracle -c "touch /backup/test"
```

### Solución de Problemas NFS
```bash
# Error DCS-10074
# Verificar permisos oracle
id oracle
ls -l /nfs_backup_client

# Error DCS-10013
# Verificar directorio
mkdir -p /nfs_backup_client
chown oracle:oinstall /nfs_backup_client
```

## 3. Backup de Componentes Críticos

### ASM/ACFS Metadata
```bash
# Archivos críticos
/etc/multipath.conf
/opt/oracle/expapi/asmappi.config
/etc/oraclefd.conf
ls -lt /dev/mapper/*

# Comandos de verificación
odaadmcli show disk -asm
odaadmcli show diskgroup
odaadmcli stordiag <disk_resource_name>
```

### TDE Wallet
```bash
# Separación de backups
# Database backup: /backup/database
# TDE wallet: /backup/tde_wallet

# Verificar configuración
orapki wallet display -wallet /backup/tde_wallet
```

## 4. Procedimientos de Backup

### Backup Completo
```bash
# Configurar backup
odacli create-backupconfig \
  --database PRODDB \
  --recovery-window 14 \
  --backup-destination /backup \
  --enable-tde

# Ejecutar backup
odacli create-backup \
  --database PRODDB \
  --level FULL
```

### Backup Incremental
```bash
# Backup incremental
odacli create-backup \
  --database PRODDB \
  --level INCREMENTAL

# Verificar estado
odacli list-backups \
  --database PRODDB
```

## 5. Recovery Procedures

### Recovery Completo
```bash
# Recovery desde último backup
odacli recover-database \
  --database PRODDB \
  --latest \
  --from-backup

# Recovery point-in-time
odacli recover-database \
  --database PRODDB \
  --timestamp "2024-02-07 14:30:00"
```

### Recovery de Archivos Específicos
```bash
# Recuperar wallet
odacli irecover-database \
  --database PRODDB \
  --json '{"files":["/opt/oracle/dcs/commonstore/wallets/tde/$ORACLE_SID"]}' \
  --from-backup
```

## 6. Mejores Prácticas

### Planificación
```plaintext
1. Backup regular de:
   - Configuración ASM
   - TDE wallets
   - Metadata crítica
   - Diskgroups

2. Momentos críticos:
   - Post-creación de ambiente
   - Pre/Post cambios de disco
   - Antes de actualizaciones
```

### Monitoreo
```bash
# Verificar estado backups
odacli list-backups

# Espacio utilizado
odacli describe-backupreport

# Logs
tail -f /var/log/oracle/backup/*.log
```

## 7. Scripts de Automatización

### Backup Automation
```python
#!/usr/bin/python3
# backup_manager.py

def verify_backup_config():
    """Verifica configuración de backup"""
    cmd = "odacli describe-backupconfig"
    return subprocess.run(cmd.split(), 
                        capture_output=True).stdout

def execute_backup(db_name, level="FULL"):
    """Ejecuta backup"""
    cmd = f"odacli create-backup --database {db_name} --level {level}"
    return subprocess.run(cmd.split(), 
                        capture_output=True).stdout

def monitor_backup_status(job_id):
    """Monitorea estado del backup"""
    cmd = f"odacli describe-job -i {job_id}"
    return subprocess.run(cmd.split(), 
                        capture_output=True).stdout
```

### Verificación Diaria
```bash
#!/bin/bash
# daily_backup_check.sh

echo "=== Backup Status Check ==="
date

# Listar backups últimas 24h
odacli list-backups --hours 24

# Verificar espacio
df -h /backup

# Verificar logs
find /var/log/oracle/backup -mtime -1 -type f -exec tail -n 50 {} \;
```