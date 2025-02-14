# Backup de Configuración ASM y Metadata en ODA

## 1. Archivos Críticos para Backup

### Configuración Multipath
```bash
# 1. /etc/multipath.conf
cp /etc/multipath.conf /backup/config/multipath.conf.$(date +%Y%m%d)

# Verificar configuración
multipath -ll > /backup/config/multipath_status.$(date +%Y%m%d)
```

### ASM Configuration
```bash
# 2. /opt/oracle/extapi/asmappi.config
cp /opt/oracle/extapi/asmappi.config /backup/asm/
    
# 3. /etc/oraclefd.conf
cp /etc/oraclefd.conf /backup/asm/
```

### Device Mapping
```bash
# 4. Device Mapper Configuration
ls -lt /dev/mapper/ > /backup/asm/device_mapper.$(date +%Y%m%d)
```

## 2. Comandos de Verificación

### ASM Status
```bash
# Verificar discos ASM
odaadmcli show disk -asm > /backup/asm/disk_status.$(date +%Y%m%d)

# Verificar diskgroups
odaadmcli show diskgroup > /backup/asm/diskgroup_status.$(date +%Y%m%d)

# Diagnóstico de storage
odaadmcli stordiag <disk_resource_name> > /backup/asm/storage_diag.$(date +%Y%m%d)
```

## 3. Script de Backup Automatizado

```bash
#!/bin/bash
# backup_asm_config.sh

BACKUP_DIR="/backup/asm_config/$(date +%Y%m%d)"
LOG_FILE="/var/log/asm_backup.log"

# Crear directorio de backup
mkdir -p $BACKUP_DIR

echo "Iniciando backup ASM config $(date)" >> $LOG_FILE

# 1. Backup archivos de configuración
files_to_backup=(
    "/etc/multipath.conf"
    "/opt/oracle/extapi/asmappi.config"
    "/etc/oraclefd.conf"
)

for file in "${files_to_backup[@]}"; do
    if [ -f "$file" ]; then
        cp $file $BACKUP_DIR/
        echo "Backup de $file completado" >> $LOG_FILE
    else
        echo "WARNING: $file no encontrado" >> $LOG_FILE
    fi
done

# 2. Backup device mapper
ls -lt /dev/mapper/ > $BACKUP_DIR/device_mapper.txt

# 3. ASM Status
odaadmcli show disk -asm > $BACKUP_DIR/asm_disk_status.txt
odaadmcli show diskgroup > $BACKUP_DIR/diskgroup_status.txt

echo "Backup completado $(date)" >> $LOG_FILE
```

## 4. Momentos Críticos para Backup

### Post-Instalación
```bash
# Backup inicial
./backup_asm_config.sh

# Verificar estado inicial
odasmcli show disk -asm
odasmcli show diskgroup
```

### Pre/Post Cambios de Disco
```bash
# Pre-cambio backup
./backup_asm_config.sh pre_change

# Post-cambio backup
./backup_asm_config.sh post_change

# Comparar configuraciones
diff /backup/asm_config/pre_change/ /backup/asm_config/post_change/
```

## 5. Recovery Procedures

### Restauración de Configuración
```bash
# 1. Detener servicios
systemctl stop oracle-asm
systemctl stop oraclefd

# 2. Restaurar configuración
cp /backup/asm_config/latest/multipath.conf /etc/
cp /backup/asm_config/latest/asmappi.config /opt/oracle/extapi/
cp /backup/asm_config/latest/oraclefd.conf /etc/

# 3. Reiniciar servicios
systemctl start oraclefd
systemctl start oracle-asm
```

## 6. Verificación Post-Recovery

### Validación de Configuración
```bash
# 1. Verificar servicios
systemctl status oracle-asm
systemctl status oraclefd

# 2. Verificar ASM
su - grid
. .grid_profile
asmcmd lsdg

# 3. Verificar discos
odasmcli show disk -asm
multipath -ll
```

## 7. Mejores Prácticas

### Frecuencia de Backup
```plaintext
1. Diario: Estado de diskgroups y discos
2. Semanal: Configuración completa
3. Bajo demanda: 
   - Pre/Post cambios
   - Actualizaciones de sistema
   - Modificaciones de storage
```

### Retención
```bash
# Política de retención
find /backup/asm_config -type d -mtime +30 -exec rm -rf {} \;

# Mantener backups críticos
cp -r /backup/asm_config/important_date /backup/asm_config/permanent/
```