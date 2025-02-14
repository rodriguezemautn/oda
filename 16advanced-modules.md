# Módulos Avanzados ODA

## Módulo: CPU Pool Management

### Configuración de Pools
```bash
# Crear CPU pool
odacli create-cpupool \
  --name Production \
  --number-of-cores 8

# Modificar asignación
odacli modify-cpupool \
  --name Production \
  --number-of-cores 12
```

### Monitoreo
```sql
-- Uso de CPU por pool
SELECT pool_name, used_cores, total_cores
FROM v$cpupool_config;

-- Performance metrics
SELECT metric_name, value 
FROM v$sysmetric 
WHERE metric_name LIKE '%CPU%';
```

## Módulo: Gestión VMs CLI/BUI

### CLI Operations
```bash
# Crear VM
odacli create-vm \
  --name prodvm1 \
  --memory 16384 \
  --vcpu 4 \
  --networks "[{'name': 'public'}]"

# Snapshot
odacli create-vmsnapshot \
  --name prodvm1 \
  --snapshot_name snap01
```

### BUI Management
```yaml
# Template configuración
vm_config:
  name: prodvm1
  shape: 
    cpu: 4
    memory: 16384
  storage:
    system: 50G
    data: 100G
  network:
    - name: public
      vlan: 123
    - name: private
      vlan: 456
```

## Módulo: Backup y Recovery

### Configuración Backup
```bash
# Policy
odacli create-backupconfig \
  --name daily_backup \
  --recovery-window 14 \
  --protection-mode MAX_AVAILABILITY

# Ejecutar backup
odacli create-backup \
  --database PROD \
  --level FULL
```

### Recovery
```bash
# Recovery completo
odacli recover-database \
  --database PROD \
  --latest

# Recovery point-in-time 
odacli recover-database \
  --database PROD \
  --timestamp "2024-02-06 14:30:00"
```

## Módulo: Parcheo ODA y VMs

### Análisis Pre-Patch
```bash
# Verificar parches
odacli list-availablepatches

# Generar reporte
odacli create-prepatchreport \
  --version 19.12.0.0.0
```

### Aplicación de Parches
```bash
# Update repositorio
odacli update-repository \
  --filename p12345_19.12.0.0.zip

# Aplicar parche
odacli update-server \
  --version 19.12.0.0.0
```

### Parcheo VMs
```bash
# Actualizar VM OS
odacli update-vmtemplate \
  --name OL7 \
  --version 7.9

# Parches de seguridad
yum update-minimal --security
```

## Módulo: Patching Hardware

### Pre-Validación
```bash
# Hardware check
odacli validate-storageconfiguration
odacli validate-networkconfig

# Componentes
odacli describe-component
```

### Procedimiento
```bash
# 1. Backup
odacli create-backup --level FULL

# 2. Migrar servicios
crsctl stop cluster -all

# 3. Aplicar firmware
odaadmcli update -f

# 4. Verificar
odacli validate-system
```

### Monitoreo Post-Patch
```python
# monitor_patch.py
def verify_patch_status():
    checks = {
        'firmware': check_firmware_version(),
        'hardware': validate_hardware(),
        'services': check_critical_services()
    }
    return all(checks.values())

def generate_report():
    return {
        'patch_level': get_patch_version(),
        'components': get_component_status(),
        'validation': verify_patch_status()
    }
```

## Mejores Prácticas

### CPU Pool
```plaintext
1. Reservar cores para críticos
2. Monitorear uso continuo
3. Ajustar según workload
4. Documentar asignaciones
```

### VMs
```plaintext
1. Estandarizar nombres
2. Backup regular
3. Monitoreo recursos
4. Planificar capacity
```

### Patching
```plaintext
1. Validar prereqs
2. Backup completo
3. Ventana mantenimiento
4. Plan rollback
5. Verificación post-patch
```