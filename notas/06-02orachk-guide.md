# ORAchk en ODA

## Ejecución Básica
```bash
$ORACLE_HOME/suptools/orachk/orachk

# Opciones comunes
orachk -verbose          # Salida detallada
orachk -profile oda      # Perfil específico ODA
orachk -localonly       # Solo nodo local
```

## Análisis Específicos
```bash
# Pre-upgrade
orachk -upgrade -src_dbver 12.2.0.1 -dst_dbver 19.12.0.0

# Dataguard
orachk -profile dataguard

# RAC
orachk -profile rac
```

## Recolección de Datos
```bash
# Recolectar diagnósticos
orachk -collect all

# Análisis específico
orachk -profile sysadmin
orachk -profile dba
```

## Reportes
```bash
# Generar HTML
orachk -output html -outfile /tmp/report.html

# JSON output
orachk -output json -outfile /tmp/report.json
```

## Validaciones Críticas
```bash
# Security checks
orachk -profile security

# Performance
orachk -profile performance

# Best practices
orachk -profile bestpractices
```

## Automatización
```bash
# Programar chequeos
orachk -schedule daily

# Retención
orachk -retention 30

# Email
orachk -notification_email dba@example.com
```

## Troubleshooting
```bash
# Debug mode
orachk -debug

# Clean history
orachk -clean all

# Version
orachk -version
```

## Checklist Recomendado
```plaintext
Diario:
- Security checks
- Performance baseline
- Critical patches

Semanal:
- Best practices
- Storage analysis
- Network validation

Mensual:
- Full system check
- Compliance audit
- Trend analysis
```