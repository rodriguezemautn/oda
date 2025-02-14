# Módulo 4: DataGuard en ODA

## 4.1 Arquitectura DataGuard en ODA

### Componentes Principales
```plaintext
├── Broker Configuration
│   ├── DGMGRL
│   ├── Observer
│   └── Broker Files
├── Redo Transport
│   ├── SYNC
│   ├── ASYNC
│   └── FASTSYNC
├── Protection Modes
│   ├── Maximum Protection
│   ├── Maximum Availability
│   └── Maximum Performance
└── Network Configuration
    ├── TNS
    ├── Listeners
    └── SCAN
```

### Archivos Críticos
```bash
# Configuración Broker
$ORACLE_HOME/dbs/dr*.dat           # Archivos configuración broker
$ORACLE_BASE/diag/rdbms/*/*/trace  # Logs de broker

# Network
$ORACLE_HOME/network/admin/
    ./tnsnames.ora
    ./listener.ora
    ./sqlnet.ora
```

## 4.2 Preparación y Prerequisitos

### Validación Sistema
```bash
# Verificar espacio y recursos
odacli validate-system
odacli describe-system

# Requisitos de red
odacli list-networks
odacli validate-network
```

### Configuración Inicial
```sql
-- Primary Database
ALTER DATABASE FORCE LOGGING;
ALTER DATABASE FLASHBACK ON;
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(primary_db,standby_db)';

-- Parámetros críticos
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1='LOCATION=USE_DB_RECOVERY_FILE_DEST VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=primary_db';
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=standby_db ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=standby_db';
```

## 4.3 Implementación DataGuard

### Usando ODACLI
```bash
# Configuración básica
odacli configure-dataguard \
    -ps PRIMARY \
    -s sys \
    -sn ORCL_STDBY \
    -r PRIMARY \
    -p password123 \
    -st SYNC \
    -bt FULL \
    -s sys

# Configuración avanzada
odacli configure-dataguard \
    -ps PRIMARY \
    -s sys \
    -sn ORCL_STDBY \
    -r PRIMARY \
    -p password123 \
    -st SYNC \
    -bt FULL \
    -s sys \
    -cl DATA,RECO \
    -br LOCATION=/u02/backup \
    -pr BASIC
```

### Usando DGMGRL
```sql
-- Conectar al broker
DGMGRL> CONNECT sys/password
DGMGRL> CREATE CONFIGURATION 'DG_ODA' AS PRIMARY DATABASE IS 'PRIMARY_DB' CONNECT IDENTIFIER IS 'PRIMARY_DB';
DGMGRL> ADD DATABASE 'STANDBY_DB' AS CONNECT IDENTIFIER IS 'STANDBY_DB';
DGMGRL> ENABLE CONFIGURATION;
```

## 4.4 Operaciones DataGuard

### Switchover
```bash
# Usando ODACLI
odacli switchover-dataguard \
    -i database_id \
    -sn standby_name

# Usando DGMGRL
DGMGRL> SWITCHOVER TO 'STANDBY_DB';
DGMGRL> SHOW CONFIGURATION;
```

### Failover
```bash
# Usando ODACLI
odacli failover-dataguard \
    -i database_id \
    -sn standby_name

# Usando DGMGRL
DGMGRL> FAILOVER TO 'STANDBY_DB';
DGMGRL> REINSTATE DATABASE 'PRIMARY_DB';
```

### Monitorización
```sql
-- Estado de la replicación
SELECT PROCESS, STATUS, THREAD#, SEQUENCE#
FROM V$MANAGED_STANDBY;

-- Gap de archivos
SELECT * FROM V$ARCHIVE_GAP;

-- Estado del transporte
SELECT * FROM V$ARCHIVE_DEST_STATUS 
WHERE STATUS != 'INACTIVE';
```

## 4.5 Mantenimiento y Troubleshooting

### Mantenimiento Regular
```bash
# Verificación estado
odacli validate-dataguard
odacli describe-dataguardstatus

# Scripts de verificación
#!/bin/bash
# check_dg_status.sh
echo "=== DataGuard Status ==="
dgmgrl / "show configuration"
dgmgrl / "show database 'PRIMARY_DB'"
dgmgrl / "show database 'STANDBY_DB'"
```

### Resolución de Problemas
```sql
-- Problemas comunes y soluciones
-- 1. Gap detection
SELECT * FROM V$ARCHIVE_GAP;
ALTER SYSTEM SWITCH LOGFILE;

-- 2. Transport issues
SELECT ERROR, STANDBY_BECAME_PRIMARY_SCN
FROM V$DATAGUARD_STATUS
WHERE SEVERITY IN ('Error', 'Fatal');

-- 3. Performance
SELECT VALUE/1024/1024 MB 
FROM V$SYSSTAT 
WHERE NAME = 'redo transport compression ratio';
```

## 4.6 Alta Disponibilidad Avanzada

### Fast-Start Failover
```sql
-- Configuración FSFO
DGMGRL> EDIT CONFIGURATION SET PROPERTY FastStartFailoverThreshold = 30;
DGMGRL> EDIT CONFIGURATION SET PROPERTY FastStartFailoverPmyShutdown = TRUE;
DGMGRL> ENABLE FAST_START FAILOVER;

-- Observer
DGMGRL> START OBSERVER;
DGMGRL> SHOW FAST_START FAILOVER;
```

### Multiple Standby
```bash
# Configuración con múltiples standby
odacli configure-dataguard \
    -ps PRIMARY \
    -s sys \
    -sn STDBY1,STDBY2 \
    -r PRIMARY \
    -p password123 \
    -st SYNC,ASYNC
```

## 4.7 Mejores Prácticas

### Performance
```sql
-- Optimización del transporte
ALTER SYSTEM SET LOG_ARCHIVE_MAX_PROCESSES = 8;
ALTER SYSTEM SET DB_FILE_ASYNC_IO_SUBMIT = TRUE;
ALTER SYSTEM SET LOG_BUFFER = 524288000;

-- Monitorización de performance
SELECT NAME, VALUE, UNIT FROM V$DATAGUARD_STATS;
```

### Seguridad
```bash
# Wallet configuration
orapki wallet create -wallet /oracle/dg/wallet -pwd WalletPasswd123
mkstore -wrl /oracle/dg/wallet -createCredential PRIMARY_DB sys password
mkstore -wrl /oracle/dg/wallet -createCredential STANDBY_DB sys password

# Network encryption
# sqlnet.ora
SQLNET.ENCRYPTION_SERVER=REQUIRED
SQLNET.CRYPTO_CHECKSUM_SERVER=REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER=(AES256,AES192,AES128)
```

## 4.8 Scripts y Automatización

### Scripts de Monitoreo
```python
# dg_monitor.py
import cx_Oracle
import smtplib
from email.message import EmailMessage

def check_dg_status():
    connection = cx_Oracle.connect("sys/password@primary as sysdba")
    cursor = connection.cursor()
    
    # Check transport lag
    cursor.execute("""
        SELECT VALUE 
        FROM V$DATAGUARD_STATS 
        WHERE NAME = 'transport lag'
    """)
    transport_lag = cursor.fetchone()[0]
    
    # Check apply lag
    cursor.execute("""
        SELECT VALUE 
        FROM V$DATAGUARD_STATS 
        WHERE NAME = 'apply lag'
    """)
    apply_lag = cursor.fetchone()[0]
    
    if int(transport_lag) > 300 or int(apply_lag) > 300:
        send_alert(f"DG Lag Alert: Transport={transport_lag}, Apply={apply_lag}")
```

### Automation Framework
```python
# dg_automation.py
class DataGuardManager:
    def __init__(self, primary_conn, standby_conn):
        self.primary = primary_conn
        self.standby = standby_conn
    
    def validate_configuration(self):
        # Check DG configuration
        pass
    
    def perform_switchover(self):
        # Execute switchover
        pass
    
    def check_health(self):
        # Monitor health metrics
        pass
    
    def generate_report(self):
        # Create status report
        pass
```

## 4.9 Referencias y Documentación

### Oracle Documentation
1. [DataGuard Concepts and Administration](https://docs.oracle.com/en/database/oracle/oracle-database/19/sbydb/)
2. [DataGuard Broker](https://docs.oracle.com/en/database/oracle/oracle-database/19/dgbkr/)
3. [ODA DataGuard Implementation](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/)

### MOS Notes Relevantes
- Note 2246444.1: DataGuard Configuration on ODA
- Note 1265700.1: DataGuard Broker Troubleshooting
- Note 1905213.1: Fast-Start Failover Configuration

### White Papers
1. "Maximum Availability Architecture: Oracle Data Guard on ODA"
2. "Best Practices for DataGuard Implementation on ODA"
3. "DataGuard Performance Tuning Guide"

### Recursos Adicionales
1. Oracle Learning Library
   - "DataGuard Administration on ODA"
   - "Advanced DataGuard Features"

2. Oracle By Example Series
   - "Setting up DataGuard on ODA"
   - "DataGuard Broker Operations"