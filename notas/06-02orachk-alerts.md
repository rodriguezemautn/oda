# Monitoreo de Alertas ORAchk

## 1. Niveles de Alertas
```bash
# Clasificación
CRITICAL   # Requiere acción inmediata
WARNING    # Atención requerida
INFO      # Informativo

# Verificar alertas activas
orachk -findfile LATEST
orachk -findings LATEST
```

## 2. Configuración de Notificaciones
```bash
# Email
orachk -notification_email "dba@company.com,sysadmin@company.com"
orachk -notification_threshold WARNING

# Integración con sistemas externos
orachk -output json -outfile alerts.json
orachk -teefile alerts.log
```

## 3. Scripts de Monitoreo
```python
#!/usr/bin/python3
# monitor_alerts.py

def parse_orachk_alerts():
    cmd = "orachk -findings LATEST"
    alerts = {
        'CRITICAL': [],
        'WARNING': [],
        'INFO': []
    }
    # Parse output and categorize
    return alerts

def notify_team(alerts):
    if alerts['CRITICAL']:
        send_urgent_notification()
    elif alerts['WARNING']:
        send_warning_notification()
```

## 4. Acciones por Tipo

### Critical Alerts
```plaintext
1. Storage Issues
   - ASM disk space
   - Filesystem usage
   - Archive log destinations

2. Database Issues
   - Corrupt blocks
   - Invalid objects
   - Failed services

3. System Issues
   - Memory pressure
   - CPU overload
   - Network failures
```

### Warning Alerts
```plaintext
1. Performance
   - High load average
   - Slow queries
   - I/O bottlenecks

2. Space
   - Growing tablespaces
   - Backup retention
   - Trace files accumulation

3. Configuration
   - Parameter mismatches
   - Missing patches
   - Service misconfigurations
```

## 5. Automatización
```bash
# Chequeos programados
orachk -schedule "0 */4 * * *"    # Cada 4 horas
orachk -schedule "0 0 * * *"      # Diario

# Retención
orachk -purge_after 30            # Mantener 30 días
orachk -clean_findings            # Limpiar obsoletos
```

## 6. Dashboard y Reportes
```sql
-- Crear tabla de seguimiento
CREATE TABLE orachk_history (
    check_date DATE,
    alert_type VARCHAR2(10),
    description CLOB,
    status VARCHAR2(20),
    resolution_date DATE
);

-- Insertar alertas
INSERT INTO orachk_history
SELECT SYSDATE, level, message, 'OPEN', NULL
FROM orachk_findings;
```

## 7. Mejores Prácticas

### Seguimiento
```plaintext
1. Revisión diaria de CRITICAL
2. Revisión semanal de WARNING
3. Documentación de resoluciones
4. Análisis de tendencias
```

### Mantenimiento
```plaintext
1. Actualizar reglas
2. Validar falsos positivos
3. Ajustar umbrales
4. Revisar exclusiones
```