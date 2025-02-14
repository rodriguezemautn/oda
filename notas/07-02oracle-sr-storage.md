# Apertura de SR para Cambios de Storage en ODA

## 1. Preparación de Información

### Datos del Sistema
```plaintext
- CSI Number/ID
- Hostname
- OS Version
- ODA Version
- ASR Manager URL
- HTTPS Port Number
```

### Información de Storage
```bash
# Recolectar outputs
odacli describe-system > system_info.txt
odacli list-diskgroups > diskgroups.txt
odacli describe-component > components.txt

# Storage actual
asmcmd lsdg > current_storage.txt
df -h > filesystem_info.txt
```

## 2. Creación del SR

### Información Requerida
```plaintext
1. Severity Level: 
   - Severity 2: Impacto en producción
   - Severity 3: Planificado

2. Problema Type:
   - Hardware
   - Storage & ASM

3. Detalles:
   - Descripción del cambio
   - Justificación
   - Ventana de mantenimiento
```

### Templates

#### Agregar Disco
```plaintext
Subject: ODA - Request to Add Storage Disk

Description:
- ODA Model: [modelo]
- Current Storage: [capacidad actual]
- Requested Addition: [capacidad solicitada]
- Business Justification: [justificación]
- Maintenance Window: [fecha/hora]

Attachments:
- Current system configuration
- Storage layout
- Business approval if required
```

#### Reemplazo de Disco
```plaintext
Subject: ODA - Disk Replacement Request

Description:
- Faulty Disk Location: [ubicación]
- Disk Serial Number: [serial]
- Error Messages: [mensajes]
- Impact: [impacto]
- Urgency: [urgencia]

Attachments:
- Alert logs
- ASM logs
- Current disk status
```

## 3. Seguimiento

### Actualizaciones al SR
```plaintext
1. Documentación Adicional:
   - Confirmación de backup
   - Plan de rollback
   - Ventana de mantenimiento aprobada

2. Comunicación:
   - Responder dentro de 24h
   - Incluir logs solicitados
   - Actualizar progreso
```

### Post-Implementación
```bash
# Recolectar nueva información
odacli describe-system
odacli validate-storageconfiguration
asmcmd lsdg

# Actualizar SR con resultados
- Confirmación de cambios
- Nuevas capacidades
- Pruebas realizadas
```

## 4. Documentación

### Checklist Pre-SR
```plaintext
□ Recolección de información base
□ Aprobaciones necesarias
□ Plan de implementación
□ Plan de rollback
□ Ventana de mantenimiento
```

### Tracking
```sql
-- Tabla de seguimiento
CREATE TABLE storage_sr_tracking (
    sr_number VARCHAR2(20),
    creation_date DATE,
    description VARCHAR2(200),
    status VARCHAR2(20),
    closure_date DATE
);
```

## 5. Mejores Prácticas

### Preparación
```plaintext
1. Información Clara:
   - Descripción detallada
   - Justificación de negocio
   - Impacto y urgencia

2. Documentación:
   - Diagramas actuales
   - Configuración propuesta
   - Procedimientos paso a paso
```

### Seguimiento
```plaintext
1. Comunicación:
   - Actualizaciones regulares
   - Respuestas oportunas
   - Documentación de cambios

2. Cierre:
   - Verificación completa
   - Documentación actualizada
   - Lecciones aprendidas
```