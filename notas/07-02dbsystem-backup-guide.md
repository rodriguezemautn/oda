# Backup de DBSystem en ODA

## 1. Verificación del Entorno

### Identificar Nodos y DBSystems
```bash
# Listar DBSystems existentes
odacli list-dbsystems

# Verificar detalles del DBSystem
odacli describe-dbsystem -n odadbs1

Información importante:
- Shape: odb6
- Memory: 16.00 GB
- Status: CONFIGURED
- VM names y hosts
```

## 2. Backup de OCR (Oracle Cluster Registry)

### Preparación
```bash
# Detener servicios en nodos
/opt/oracle/dcs/hasmctl.sh stop --ensemble GDA_DCS --member GDA_DCS0
/opt/oracle/dcs/hasmctl.sh stop --ensemble GDA_DCS --member GDA_DCS1

# Verificar estado
/opt/oracle/dcs/hasmctl.sh status --ensemble GDA_DCS
```

### Backup OCR
```bash
# Crear backup manual OCR
/u01/app/19.20.0.0/grid/bin/ocrconfig -manualbackup

# Verificar backup
ls -l /DATA/db385531df36/OCRBACKUP/backup_*
```

### Copiar Backup
```bash
# Copiar a filesystem local
BKPID=`ls -tr /DATA/db385531df36/OCRBACKUP/backup_* | tail -1`
cp $BKPID /tmp/ocr_odadbs1n1.dump

# Copiar al DBSystem
cp /tmp/ocr_odadbs1n1.dump root@odadbs1n1:/tmp
```

## 3. Snapshots ACFS

### Identificar VMs
```bash
# Listar VMs en ambos nodos
virsh domblklist x1958425b1 | grep vda

# Verificar ubicación
/u05/app/sharedrepo/odadbs1/.ACFS/snaps/vm_x1958425b1/x1958425b1
```

### Crear Snapshots
```bash
# Crear snapshot ACFS
acfsutil snap create -p vm_x1958425b1 vm_x1958425b1_bkp \
  /u05/app/sharedrepo/odadbs1

# Verificar snapshots
acfsutil snap info -t /u05/app/sharedrepo/odadbs1
```

## 4. Restauración de Servicios

### Iniciar Servicios
```bash
# Para versiones 19.20 y posteriores
/opt/oracle/dcs/hasmctl.sh start

# Verificar estado
/opt/oracle/dcs/hasmctl.sh status --ensemble GDA_DCS
```

## 5. Validación

### Verificar Backup
```bash
# Verificar OCR backup
ocrcheck

# Verificar snapshots
acfsutil snap info -t /u05/app/sharedrepo/odadbs1

# Verificar estado DBSystem
odacli describe-dbsystem -n odadbs1
```

## 6. Scripts de Automatización

### Backup Completo
```bash
#!/bin/bash
# dbsystem_backup.sh

# Variables
DBSYS_NAME="odadbs1"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/dbsystem/${DBSYS_NAME}/${BACKUP_DATE}"

# 1. Preparar entorno
mkdir -p $BACKUP_DIR
echo "Starting backup at $(date)" > $BACKUP_DIR/backup.log

# 2. Detener servicios
/opt/oracle/dcs/hasmctl.sh stop --ensemble GDA_DCS --member GDA_DCS0
/opt/oracle/dcs/hasmctl.sh stop --ensemble GDA_DCS --member GDA_DCS1

# 3. Backup OCR
/u01/app/19.20.0.0/grid/bin/ocrconfig -manualbackup

# 4. Crear snapshots
acfsutil snap create -p vm_x1958425b1 \
  vm_x1958425b1_bkp_${BACKUP_DATE} \
  /u05/app/sharedrepo/odadbs1

# 5. Restaurar servicios
/opt/oracle/dcs/hasmctl.sh start

# 6. Verificación
echo "Backup completed at $(date)" >> $BACKUP_DIR/backup.log
```

## 7. Consideraciones Importantes

### Precauciones
```plaintext
1. NO congelar I/O de VMs guest
   - Puede causar evicción por clusterware

2. Espacio requerido
   - OCR backup
   - Snapshots ACFS
   - Filesystem local

3. Timing
   - Ventana de mantenimiento
   - Impacto en servicios
   - Tiempo de recuperación
```

### Mejores Prácticas
```plaintext
1. Backup regular
   - OCR: backup semanal
   - Snapshots: según cambios
   - Retención definida

2. Documentación
   - Procedimientos
   - Ubicaciones
   - Contactos

3. Testing
   - Restore periódico
   - Validación de integridad
   - Actualización de procedimientos
```