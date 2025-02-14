# Gestión de DBSystems en ODA

## Verificación de Estado
```bash
# Estado general
/u01/app/19.24.0.0/grid/bin/crsctl stat res -t
/u01/app/19.24.0.0/grid/bin/crsctl check cluster

# Estado de bases de datos
srvctl status database -d <db_name>
srvctl config database -d <db_name>
```

## Operaciones Comunes
```bash
# Start/Stop Database
srvctl start database -d <db_name>
srvctl stop database -d <db_name>

# Gestión de servicios
srvctl add service -d <db_name> -s <svc_name>
srvctl modify service -d <db_name> -s <svc_name>
srvctl remove service -d <db_name> -s <svc_name>
```

## Monitoreo
```sql
-- ASM Space
SELECT name, total_mb, free_mb, 
       ROUND((free_mb/total_mb)*100,2) pct_free
FROM v$asm_diskgroup;

-- Servicios activos
SELECT name, network_name, creation_date
FROM dba_services;
```

## Logs Importantes
```plaintext
/u01/app/grid/diag/crs/*/crs/trace
/u01/app/grid/log/*/
/u01/app/oracle/diag/rdbms/*/*/trace
```

## Scripts de Mantenimiento
```bash
#!/bin/bash
# check_dbsystems.sh

echo "=== Cluster Status ==="
$GRID_HOME/bin/crsctl stat res -t

echo "=== Database Status ==="
for db in $(srvctl config database | cut -d' ' -f1)
do
    echo "Database: $db"
    srvctl status database -d $db
done
```

## Alta Disponibilidad
```bash
# Failover manual
srvctl relocate database -d <db_name> -t <target_node>

# Política de failover
srvctl modify database -d <db_name> -policy AUTOMATIC
```

## Troubleshooting
```bash
# Logs de diagnóstico
adrci> show problem
adrci> show incident

# Validación cluster
cluvfy comp healthcheck -n all
```