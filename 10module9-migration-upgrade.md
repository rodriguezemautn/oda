# Módulo 9: Migración y Upgrade en ODA

## 9.1 Metodologías de Migración a ODA

### Evaluación y Planificación
```bash
# 1. Análisis de Origen
sqlplus / as sysdba <<EOF
SELECT SUM(bytes)/1024/1024/1024 "Size GB",
       tablespace_name
FROM dba_segments
GROUP BY tablespace_name;

SELECT COUNT(*) total_objects,
       owner, object_type
FROM dba_objects
GROUP BY owner, object_type;
EOF

# 2. Verificación de Espacio
odacli describe-storage
odacli validate-storageconfiguration

# 3. Compatibilidad
$ORACLE_HOME/rdbms/admin/preupgrd.sql
```

### Métodos de Migración Soportados

1. Data Pump
```bash
# Configuración en origen
mkdir -p /u01/datapump
CREATE OR REPLACE DIRECTORY dp_dir AS '/u01/datapump';

# Export
expdp system/password \
  directory=dp_dir \
  dumpfile=full_db_%U.dmp \
  filesize=10G \
  full=y \
  metrics=y \
  logfile=export.log \
  compression=all

# Import en ODA
impdp system/password \
  directory=dp_dir \
  dumpfile=full_db_%U.dmp \
  full=y \
  metrics=y \
  logfile=import.log \
  table_exists_action=replace
```

2. RMAN
```bash
# Backup en origen
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 4;
RMAN> BACKUP AS COMPRESSED BACKUPSET 
      DATABASE FORMAT '/backup/db_%d_%T_%U'
      PLUS ARCHIVELOG FORMAT '/backup/arch_%d_%T_%U';

# Restore en ODA
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE CONTROLFILE FROM '/backup/cf_PROD_20240205';
RMAN> ALTER DATABASE MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

3. Transportable Database
```sql
-- En origen
ALTER DATABASE OPEN READ ONLY;
BEGIN 
  DBMS_TDB.CHECK_DB(
    skip_option => DBMS_TDB.SKIP_NONE);
END;
/

-- Transferir archivos
scp /u01/app/oracle/oradata/* oda:/u01/app/oracle/oradata/

-- En ODA
CREATE PFILE='/tmp/init_tdb.ora' FROM SPFILE;
STARTUP NOMOUNT PFILE='/tmp/init_tdb.ora';
ALTER DATABASE MOUNT;
ALTER DATABASE OPEN;
```

## 9.2 Upgrade de Versiones de Base de Datos

### Pre-Upgrade Tasks
```sql
-- 1. Verificación Pre-Upgrade
@$ORACLE_HOME/rdbms/admin/preupgrd.sql

-- 2. Corrección de Objetos Inválidos
@$ORACLE_HOME/rdbms/admin/utlrp.sql

-- 3. Backup de Diccionario
BEGIN
  DBMS_STATS.GATHER_DICTIONARY_STATS;
END;
/

-- 4. Generar AWR Snapshot
BEGIN
  DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT();
END;
/
```

### Proceso de Upgrade

1. Upgrade Manual
```bash
# Preparación
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=ORCL

# Upgrade
dbupgrade -d $ORACLE_HOME \
          -c "ORCL" \
          -l /u01/app/oracle/upgrade_logs \
          -n 4

# Post-Upgrade
@$ORACLE_HOME/rdbms/admin/utlu192s.sql
@$ORACLE_HOME/rdbms/admin/utlrp.sql
@$ORACLE_HOME/rdbms/admin/catcon.pl -n 1 -l /tmp -b utlup catuppst.sql
```

2. Upgrade con AutoUpgrade
```bash
# Configuración
cat > config.cfg <<EOF
global.autoupg_log_dir=/u01/autoupgrade/log
global.script_dir=/u01/autoupgrade/script
upg1.source_home=/u01/app/oracle/product/12.2.0/dbhome_1
upg1.target_home=/u01/app/oracle/product/19.0.0/dbhome_1
upg1.sid=ORCL
upg1.start_time=NOW
upg1.upgrade_node=node1
EOF

# Ejecución
java -jar $ORACLE_HOME/rdbms/admin/autoupgrade.jar \
  -config config.cfg \
  -mode analyze

java -jar $ORACLE_HOME/rdbms/admin/autoupgrade.jar \
  -config config.cfg \
  -mode deploy
```

### Post-Upgrade Tasks
```sql
-- 1. Verificar componentes
SELECT comp_name, version, status 
FROM dba_registry;

-- 2. Recompilar objetos inválidos
@$ORACLE_HOME/rdbms/admin/utlrp.sql

-- 3. Actualizar estadísticas
EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;

-- 4. Verificar rendimiento
SELECT snap_id, instance_number, 
       startup_time, begin_interval_time
FROM dba_hist_snapshot
ORDER BY snap_id DESC;
```

## 9.3 Migración entre Sistemas ODA

### Preparación
```bash
# 1. Validación de sistemas
odacli validate-system
odacli describe-system

# 2. Verificar espacio
odacli describe-storage

# 3. Network Setup
odacli describe-network
```

### Métodos de Migración

1. Replicación con DataGuard
```bash
# Configurar DataGuard
odacli configure-dataguard \
  --primary oda1.example.com \
  --standby oda2.example.com \
  --type physical \
  --protection-mode MAX_AVAILABILITY \
  --transport-type SYNC \
  --database PRODDB

# Switchover a nuevo ODA
dgmgrl sys/password@PRODDB
SWITCHOVER TO 'STDBY_DB';
```

2. Backup y Restore
```bash
# Backup en ODA origen
odacli create-backup \
  --database PRODDB \
  --level FULL \
  --tag migration_backup

# Copiar backup
rsync -av /backup/PRODDB oda2:/backup/

# Restore en ODA destino
odacli recover-database \
  --database PRODDB \
  --from-backup \
  --backup-tag migration_backup
```

3. Zero Downtime Migration
```bash
# Configurar ZDM
zdmcli create migration \
  --platform oda \
  --source sys/pwd@PRODDB \
  --target sys/pwd@TARGETDB \
  --goldimage /images/db19c.zip \
  --method online

# Ejecutar migración
zdmcli start migration -id migration_id
```

### Validación Post-Migración
```sql
-- 1. Verificar objetos
SELECT owner, object_type, status, COUNT(*)
FROM dba_objects
GROUP BY owner, object_type, status;

-- 2. Validar datos
SELECT table_name, num_rows
FROM dba_tables
WHERE owner = 'SCHEMA_NAME'
ORDER BY num_rows DESC;

-- 3. Test de rendimiento
SELECT elapsed_time/1000000 seconds,
       executions,
       (elapsed_time/1000000)/executions avg_seconds,
       sql_text
FROM v$sql
WHERE executions > 100
ORDER BY elapsed_time DESC;
```

## 9.4 Mejores Prácticas

### Planificación
```plaintext
1. Análisis de Impacto
   - Tamaño de bases de datos
   - Ventanas de mantenimiento
   - Dependencias de aplicaciones

2. Plan de Rollback
   - Backup completo pre-migración
   - Scripts de rollback
   - Puntos de verificación

3. Validación
   - Checklist pre-migración
   - Test cases
   - Criterios de éxito
```

### Performance
```sql
-- Monitoreo durante migración
CREATE TABLE migration_metrics (
  timestamp DATE,
  metric_name VARCHAR2(50),
  metric_value NUMBER
);

-- Insertar métricas
INSERT INTO migration_metrics
SELECT SYSDATE, metric_name, value
FROM v$sysmetric
WHERE group_id = 2;

-- Análisis post-migración
SELECT metric_name, 
       MIN(metric_value), 
       MAX(metric_value),
       AVG(metric_value)
FROM migration_metrics
GROUP BY metric_name;
```

## 9.5 Troubleshooting

### Problemas Comunes

1. Espacio Insuficiente
```bash
# Verificar espacio
df -h
asmcmd lsdg

# Liberar espacio
odacli cleanup-storage
```

2. Performance
```sql
-- Identificar cuellos de botella
SELECT event, count(*)
FROM v$active_session_history
WHERE sample_time > SYSDATE - 1/24
GROUP BY event
ORDER BY count(*) DESC;
```

3. Network Issues
```bash
# Test conectividad
tnsping TARGETDB
nmap -p 1521 target_oda

# Verificar configuración
cat $ORACLE_HOME/network/admin/tnsnames.ora
cat $ORACLE_HOME/network/admin/listener.ora
```

## Referencias y Documentación

### Oracle Documentation
1. [Database Upgrade Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/upgrd/)
2. [ODA Migration Guide](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/migrate/)
3. [Zero Downtime Migration](https://docs.oracle.com/en/database/oracle/zero-downtime-migration/)

### MOS Notes
- Note 2485457.1: Database Migration Best Practices
- Note 2485978.1: ODA Migration Checklist
- Note 2485666.1: Database Upgrade on ODA