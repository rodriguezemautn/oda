# Módulo 2: ODA Web Manager

## 2.1 Arquitectura y Acceso

### Componentes
```bash
# Servicios principales
/opt/oracle/dcs/          # DCS (Database Configuration Services)
/opt/oracle/oak/          # OAK (Oracle Appliance Kit)
/var/opt/oracle/dcs/     # Configuración DCS
/var/opt/oracle/oak/     # Configuración OAK
```

### Acceso y Autenticación
```bash
# URL de acceso
https://oda-hostname:7093/mgmt/index.html

# Usuarios por defecto
OADB_ADMIN   # Administrador principal
APPUSER      # Usuario de aplicación

# Archivo de configuración de usuarios
/opt/oracle/dcs/conf/dcs-user.conf
```

## 2.2 Interfaz y Navegación

### Dashboard Principal
1. Estado del Sistema
   - CPU y Memoria
   - Almacenamiento
   - Estado de Bases de Datos
   - Alertas Activas

2. Métricas en Tiempo Real
   - Rendimiento de BD
   - I/O por instancia
   - Espacio en ASM
   - Network throughput

### Secciones Principales
```plaintext
├── Sistema
│   ├── Estado general
│   ├── Configuración de red
│   ├── Almacenamiento
│   └── Actualizaciones
├── Bases de Datos
│   ├── Listado
│   ├── Creación
│   ├── Backup/Restore
│   └── Parches
├── Monitoreo
│   ├── Métricas
│   ├── Alertas
│   ├── Jobs
│   └── Logs
└── Configuración
    ├── Usuarios
    ├── Seguridad
    ├── Red
    └── Notificaciones
```

## 2.3 Operaciones Principales

### Gestión de Bases de Datos
```bash
# Creación de BD
1. Databases → Create
2. Parámetros principales:
   - System Identifier (SID)
   - Database Name
   - Shape (cantidad de recursos)
   - Character Set
   - Storage (ASM)

# Backup/Restore
1. Databases → Select DB → Backup
2. Opciones:
   - Full/Incremental
   - Schedule
   - Retention
```

### Gestión de Almacenamiento
```bash
# Monitoreo ASM
1. Storage → ASM
2. Métricas disponibles:
   - Free/Used Space
   - I/O Performance
   - Disk Status

# Expansión Storage
1. Storage → Expand
2. Pasos:
   - Add Physical Disks
   - Configure ASM
   - Rebalance
```

### Gestión de Red
```bash
# Configuración Interfaces
1. Network → Interfaces
2. Opciones:
   - Bond Configuration
   - VLAN Setup
   - IP Assignment

# Public/Private Networks
1. Network → Network Settings
2. Configuración:
   - Public Network
   - Private Network
   - SCAN Configuration
```

## 2.4 Monitorización y Alertas

### Sistema de Alertas
```bash
# Tipos de Alertas
- Critical: Requiere acción inmediata
- Warning: Atención requerida
- Info: Informativo

# Configuración
1. Monitoring → Alert Rules
2. Parámetros:
   - Threshold
   - Notification
   - Auto-clear
```

### Métricas Clave
```bash
# Performance Metrics
- CPU Utilization
- Memory Usage
- I/O Statistics
- Network Throughput

# Database Metrics
- Active Sessions
- SQL Response Time
- Wait Events
- Buffer Cache Hit Ratio
```

## 2.5 Mantenimiento

### Actualizaciones y Parches
```bash
# Proceso de Actualización
1. Updates → Available Updates
2. Pasos:
   - Download Patches
   - Validate Prerequisites
   - Apply Patches
   - Verify Installation

# Tipos de Patches
- Security Patches
- Bundle Patches (BP)
- Patch Set Updates (PSU)
- Release Updates (RU)
```

### Backup y Recovery
```bash
# Configuración Backup
1. Backup → Configuration
2. Parámetros:
   - Backup Destination
   - Retention Policy
   - Schedule
   - Compression

# Recovery Procedures
1. Recovery → Database
2. Opciones:
   - Point-in-Time
   - SCN-based
   - Full Recovery
```

## 2.6 Mejores Prácticas

### Seguridad
```bash
# Recomendaciones
1. Cambio regular de contraseñas
2. Implementar SSL/TLS
3. Monitoreo de accesos
4. Auditoría de operaciones

# Configuración SSL
1. Security → SSL Configuration
2. Pasos:
   - Generate CSR
   - Install Certificate
   - Configure Listeners
```

### Performance
```bash
# Optimización
1. Regular monitoring
2. Resource allocation
3. Storage balancing
4. Network optimization

# Mantenimiento
1. Regular cleanup
2. Log rotation
3. Estadísticas actualizadas
4. Backup verification
```

## 2.7 Troubleshooting

### Problemas Comunes
```bash
# Acceso Web
1. Verificar servicio DCS
   systemctl status oracle-dcs
2. Verificar puertos
   netstat -tulpn | grep 7093

# Performance
1. Check resources
   top, iostat, vmstat
2. Verify metrics
   Browser Developer Tools
```

### Logs Relevantes
```bash
# Ubicaciones
/var/opt/oracle/dcs/log/         # DCS logs
/var/opt/oracle/oak/log/         # OAK logs
/var/log/oracle/dcs/            # Web logs
/var/log/oracle/oak/            # Application logs

# Comandos útiles
tail -f /var/opt/oracle/dcs/log/dcs-agent.log
grep ERROR /var/opt/oracle/oak/log/oak-manager.log
```

## 2.8 APIs y Automatización

### REST APIs
```bash
# Endpoints principales
https://oda-host:7093/api/database
https://oda-host:7093/api/storage
https://oda-host:7093/api/network

# Ejemplo curl
curl -k -X GET \
  -H "Authorization: Bearer $TOKEN" \
  https://oda-host:7093/api/database/list
```

### Automatización
```bash
# Script ejemplo
#!/bin/bash
# get_db_status.sh
TOKEN=$(get_auth_token)
curl -k -X GET \
  -H "Authorization: Bearer $TOKEN" \
  https://oda-host:7093/api/database/status \
  | jq .
```