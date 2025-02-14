# Gestión de Snapshots DBSystem en ODA

## 1. Creación de Snapshots

### Snapshots ACFS
```bash
# Crear snapshot
acfsutil snap create -p vm_x1958425b1 \
  vm_x1958425b1_bkp_$(date +%Y%m%d) \
  /u05/app/sharedrepo/odadbs1

# Verificar creación
acfsutil snap info -t /u05/app/sharedrepo/odadbs1
```

### Automatización de Creación
```python
#!/usr/bin/python3
# create_dbsystem_snap.py

import subprocess
from datetime import datetime

class DBSystemSnapshot:
    def __init__(self, dbsystem_name, vm_name):
        self.dbsystem = dbsystem_name
        self.vm = vm_name
        self.date = datetime.now().strftime('%Y%m%d_%H%M')
        
    def create_snapshot(self):
        cmd = f"acfsutil snap create -p {self.vm} "\
              f"{self.vm}_bkp_{self.date} "\
              f"/u05/app/sharedrepo/{self.dbsystem}"
        return subprocess.run(cmd.split(), capture_output=True)
```

## 2. Backup Externo

### Replicación Remota
```bash
# Script de replicación
#!/bin/bash
# replicate_snapshot.sh

SOURCE_PATH="/u05/app/sharedrepo/vmstore/.ACFS/snaps"
REMOTE_HOST="backup-server"
REMOTE_PATH="/backup/dbsystems"
RETENTION_DAYS=30

# Sincronizar snapshots
rsync -avz --delete \
  $SOURCE_PATH/ \
  $REMOTE_HOST:$REMOTE_PATH/

# Limpiar snapshots antiguos
ssh $REMOTE_HOST "find $REMOTE_PATH -type f -mtime +$RETENTION_DAYS -delete"
```

### Compresión y Transferencia
```bash
# Comprimir snapshot
tar czf /tmp/snap_${VM_NAME}_${DATE}.tar.gz \
  /u05/app/sharedrepo/vmstore/.ACFS/snaps/${SNAP_NAME}

# Transferir a almacenamiento externo
scp /tmp/snap_${VM_NAME}_${DATE}.tar.gz \
  backup_user@backup-server:/backup/dbsystems/
```

## 3. Políticas de Retención

### Configuración de Retención
```python
# retention_config.py
retention_policy = {
    'daily': 7,    # Mantener 7 días
    'weekly': 4,   # Mantener 4 semanas
    'monthly': 12  # Mantener 12 meses
}

# Script de limpieza
def cleanup_snapshots():
    for policy, days in retention_policy.items():
        find_cmd = f"find {BACKUP_PATH} -name '*{policy}*' -mtime +{days} -delete"
        subprocess.run(find_cmd, shell=True)
```

### Rotación Automática
```bash
# rotate_snapshots.sh
#!/bin/bash

# Configuración
MAX_DAILY=7
MAX_WEEKLY=4
MAX_MONTHLY=12

# Eliminar snapshots antiguos
acfsutil snap delete \
  $(acfsutil snap info | \
    grep -E 'daily|weekly|monthly' | \
    sort -k2 -r | \
    tail -n +$((MAX_DAILY+MAX_WEEKLY+MAX_MONTHLY+1)))
```

## 4. Restauración

### Proceso de Restauración
```bash
# 1. Detener DBSystem
odacli stop-dbsystem -n odadbs1 --node-name oda1

# 2. Restaurar snapshot
acfsutil snap revert -f \
  vm_x1958425b1_bkp \
  /u05/app/sharedrepo/odadbs1

# 3. Restaurar OCR si es necesario
ocrconfig -restore /tmp/ocr_backup.dump

# 4. Verificar estado
crsctl stat res -t
```

### Validación Post-Restauración
```bash
#!/bin/bash
# validate_restore.sh

echo "=== Validación de Restauración ==="

# Verificar servicios
crsctl stat res -t | tee restore_validation.log

# Verificar OCR
ocrcheck >> restore_validation.log

# Verificar ASM
asmcmd lsdg >> restore_validation.log
```

## 5. Automatización y Monitoreo

### Sistema de Backup Automatizado
```python
# automated_backup.py
class DBSystemBackup:
    def __init__(self):
        self.config = self.load_config()
        
    def create_backup(self):
        # Detener servicios
        self.stop_services()
        
        # Crear snapshots
        self.create_snapshots()
        
        # Backup OCR
        self.backup_ocr()
        
        # Replicar a remoto
        self.replicate_backup()
        
        # Restaurar servicios
        self.start_services()
```

### Monitoreo de Estado
```bash
#!/bin/bash
# monitor_snapshots.sh

# Verificar espacio
df -h /u05/app/sharedrepo

# Listar snapshots
acfsutil snap info -t /u05/app/sharedrepo/odadbs1

# Verificar replicación
rsync --dry-run -avz \
  /u05/app/sharedrepo/vmstore/.ACFS/snaps/ \
  backup-server:/backup/dbsystems/
```

## 6. Recuperación de Desastres

### Plan de DR
```plaintext
1. Preparación:
   - Backup completo inicial
   - Replicación continua
   - Documentación actualizada

2. Procedimiento:
   - Evaluación de daños
   - Selección punto de recuperación
   - Restauración por fases
   - Validación completa
```

### Scripts de Recovery
```bash
# disaster_recovery.sh
#!/bin/bash

DR_SERVER="dr-site"
DR_PATH="/backup/dbsystems"

# 1. Sincronizar últimos cambios
rsync -avz \
  $DR_SERVER:$DR_PATH/ \
  /u05/app/sharedrepo/vmstore/.ACFS/snaps/

# 2. Restaurar DBSystem
odacli restore-dbsystem \
  -n odadbs1 \
  --snapshot-id ${SNAP_ID}
```

## 7. Mejores Prácticas

### Backup Strategy
```plaintext
1. Frecuencia:
   - Snapshots diarios de DBSystem
   - Backup OCR semanal
   - Replicación continua

2. Validación:
   - Test de restauración mensual
   - Verificación de integridad
   - Simulacros de DR
```

### Documentación
```plaintext
1. Procedimientos:
   - Creación de snapshots
   - Proceso de restauración
   - Recovery de desastres

2. Registros:
   - Logs de operaciones
   - Historial de snapshots
   - Incidentes y resoluciones
```