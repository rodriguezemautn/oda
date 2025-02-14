# Módulo: Actualizaciones y Parches del Sistema Operativo ODA

## 1. Arquitectura y Componentes

### 1.1 Estructura del SO
```bash
# Componentes críticos
/etc/oracle/            # Configuración Oracle
/opt/oracle/dcs/        # Database Configuration Services
/opt/oracle/oak/        # Oracle Appliance Kit
/var/opt/oracle/        # Logs y datos variables
```

### 1.2 Repositorios y Canales
```bash
# Verificar repositorios
yum repolist
subscription-manager repos --list-enabled

# Configurar repositorios ODA
/opt/oracle/oak/bin/oakcli update -patch patch_number
```

## 2. Tipos de Actualizaciones

### 2.1 Actualizaciones Mayores
```bash
# Pre-update checks
odacli validate-system
odacli describe-system
df -h

# Backup crítico
tar -czf /backup/os_config_$(date +%Y%m%d).tar.gz \
    /etc/oracle \
    /etc/sysconfig/network-scripts \
    /etc/hosts

# Actualización
yum update oracle-database-appliance-*
yum update
```

### 2.2 Parches de Seguridad
```bash
# Identificar parches
yum updateinfo list security
yum updateinfo list cves

# Aplicar selectivamente
yum update-minimal --security
yum update-minimal --cve CVE-2024-XXXX
```

## 3. Procedimiento Rolling Upgrade

### 3.1 Preparación
```python
# check_prerequisites.py
def validate_system():
    checks = {
        'space': check_disk_space(),
        'services': check_critical_services(),
        'backups': verify_recent_backups(),
        'connectivity': check_node_connectivity()
    }
    return all(checks.values())

def prepare_upgrade():
    if validate_system():
        create_backup()
        stop_applications()
        notify_users()
```

### 3.2 Nodo 1
```bash
# 1. Migración de servicios
crsctl stop crs -f

# 2. Actualización
yum clean all
yum update oracle-database-appliance-19.12.0.0.0-1.x86_64
yum update-minimal --security

# 3. Verificación
rpm -qa | grep oracle
systemctl status oracleagi
```

### 3.3 Nodo 2
```bash
# 1. Migración
crsctl stop crs -f

# 2. Actualización
yum clean all
yum update
yum update-minimal --security

# 3. Verificación
odacli validate-system
crsctl stat res -t
```

## 4. Monitorización y Validación

### 4.1 Sistema de Monitoreo
```python
# monitor_update.py
class UpdateMonitor:
    def __init__(self, nodes):
        self.nodes = nodes
        self.metrics = {}
    
    def collect_metrics(self):
        for node in self.nodes:
            self.metrics[node] = {
                'version': get_os_version(node),
                'services': check_services(node),
                'resources': get_resource_usage(node)
            }
    
    def validate_update(self):
        for node in self.nodes:
            if not self._validate_node(node):
                return False
        return True
```

### 4.2 Verificación Post-Update
```sql
-- Registro de actualizaciones
CREATE TABLE os_update_history (
    update_date DATE,
    node_name VARCHAR2(50),
    update_type VARCHAR2(20),
    version_from VARCHAR2(50),
    version_to VARCHAR2(50),
    status VARCHAR2(10),
    details CLOB
);

-- Procedimiento de verificación
CREATE OR REPLACE PROCEDURE verify_update_status (
    p_node IN VARCHAR2
) AS
BEGIN
    -- Verificar servicios Oracle
    FOR svc IN (SELECT name FROM v$services) LOOP
        check_service_status(svc.name);
    END LOOP;
    
    -- Verificar ASM
    FOR dg IN (SELECT name FROM v$asm_diskgroup) LOOP
        check_diskgroup_status(dg.name);
    END LOOP;
END;
/
```

## 5. Automatización

### 5.1 Framework de Actualización
```python
# update_framework.py
class ODAUpdater:
    def __init__(self):
        self.nodes = ['node1', 'node2']
        self.logger = setup_logger()
    
    def update_node(self, node):
        try:
            self.logger.info(f"Starting update on {node}")
            self.pre_update_checks(node)
            self.migrate_services(node)
            self.apply_updates(node)
            self.verify_update(node)
        except Exception as e:
            self.logger.error(f"Update failed on {node}: {str(e)}")
            self.rollback(node)
    
    def rolling_update(self):
        for node in self.nodes:
            self.update_node(node)
            if not self.validate_node(node):
                raise Exception(f"Validation failed on {node}")
```

### 5.2 Scripts de Mantenimiento
```bash
#!/bin/bash
# maintenance.sh

function apply_security_fixes() {
    local node=$1
    
    echo "Applying security fixes on $node"
    ssh root@$node """
        # Security updates
        yum update-minimal --security -y
        
        # Log results
        rpm -qa --last > /var/log/security_updates_$(date +%Y%m%d).log
        
        # Validate
        odacli validate-system
    """
}

function update_os() {
    local node=$1
    
    echo "Updating OS on $node"
    ssh root@$node """
        # Full update
        yum update -y
        
        # Verify
        rpm -qa --last > /var/log/os_update_$(date +%Y%m%d).log
    """
}
```

## 6. Procedimientos de Emergencia

### 6.1 Rollback
```bash
# Identificar punto de rollback
yum history list

# Revertir cambios
yum history undo [transaction_id]

# Restaurar configuración
cd /
tar xzf /backup/os_config_*.tar.gz

# Verificar servicios
systemctl restart oracleagi
odacli validate-system
```

### 6.2 Recuperación de Emergencia
```bash
# Script de emergencia
#!/bin/bash
# emergency_recovery.sh

case "$1" in
    "rollback")
        yum history undo last
        restore_config
        restart_services
        ;;
    "verify")
        check_system_status
        verify_services
        generate_report
        ;;
    *)
        echo "Usage: $0 {rollback|verify}"
        exit 1
        ;;
esac
```

## 7. Mejores Prácticas y Recomendaciones

### 7.1 Planificación
```plaintext
1. Evaluación de Riesgos
   - Impacto en servicios
   - Ventanas de mantenimiento
   - Dependencias

2. Preparación
   - Backups completos
   - Plan de rollback
   - Pruebas previas

3. Ejecución
   - Monitoreo continuo
   - Validación por fases
   - Documentación
```

### 7.2 Documentación
```markdown
# Template de Actualización
## Información General
- Fecha:
- Tipo de actualización:
- Versiones (desde/hasta):
- Nodos afectados:

## Procedimiento
1. Pre-actualización
   - Validaciones realizadas
   - Backups tomados
   - Estado inicial

2. Actualización
   - Pasos ejecutados
   - Tiempos de ejecución
   - Incidencias

3. Post-actualización
   - Validaciones
   - Estado final
   - Pruebas realizadas
```