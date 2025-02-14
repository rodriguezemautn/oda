# Módulo 1: Fundamentos Oracle Linux para ODA

## 1.1 Estructura del Sistema

### Directorios Críticos
```bash
/opt/oracle/                    # Binarios Oracle
    ./dcs/                     # Database Configuration Services
    ./oak/                     # Oracle Appliance Kit
    ./ovm/                     # Oracle Virtual Machine
    ./scripts/                 # Scripts de administración
/u01/                          # Datos
    ./app/                     # Aplicaciones Oracle
        ./oracle/              # Bases de datos
        ./oraInventory/        # Inventario Oracle
    ./grid/                    # Grid Infrastructure
/var/log/oracle/               # Logs
    ./diag/                    # Diagnósticos
    ./cfgtoollogs/            # Logs de configuración
/etc/oracle/                   # Configuración
    ./olr.loc                  # Oracle Local Registry
    ./registry.xml            # Registro de componentes
/opt/oracle/oak/conf/          # Configuración OAK
/opt/oracle/dcs/conf/          # Configuración DCS
```

### Ficheros de Configuración Clave
```bash
# Configuración del Sistema
/etc/sysctl.conf              # Parámetros kernel
/etc/security/limits.conf     # Límites de recursos
/etc/hosts                    # Resolución de nombres
/etc/networks                 # Configuración de red
/etc/resolv.conf             # Configuración DNS

# Configuración Oracle
$ORACLE_HOME/network/admin/listener.ora    # Configuración listener
$ORACLE_HOME/network/admin/tnsnames.ora    # Configuración TNS
$ORACLE_HOME/dbs/init$ORACLE_SID.ora      # Parámetros instancia
$GRID_HOME/crs/config/crsconfig_params    # Parámetros Grid

# Logs Importantes
/var/log/messages            # Logs del sistema
/var/log/dmesg              # Mensajes del kernel
/var/log/oracle/diag/rdbms  # Logs de base de datos
/var/log/oracle/diag/tnslsnr # Logs del listener
```

## 1.2 Comandos Esenciales

### Administración del Sistema
```bash
# Gestión de Servicios
systemctl status oracle-database    # Estado servicio DB
systemctl status oracle-asm         # Estado servicio ASM
systemctl status oracle-listener    # Estado listener
systemctl status ntpd              # Estado servicio NTP

# Monitorización
top -u oracle                      # Procesos Oracle
iostat -xk 5                       # Estadísticas I/O
vmstat 5                           # Estadísticas memoria
netstat -rn                        # Tabla de ruteo
ps -ef | grep pmon                 # Procesos Oracle

# Gestión de Red
ip addr show                       # Interfaces de red
ethtool bond0                      # Estado bonding
nmcli connection show              # Conexiones NetworkManager
tcpdump -i bond0                   # Captura de tráfico
```

### Comandos Oracle y ODA
```bash
# ODACLI
odacli list-databases             # Listar bases de datos
odacli describe-database          # Detalles de BD
odacli list-nodes                 # Listar nodos
odacli describe-system            # Estado del sistema

# ASM
asmcmd lsdg                       # Listar diskgroups
asmcmd ls                         # Listar archivos ASM
asmcmd du                         # Uso de espacio

# Database
sqlplus / as sysdba              # Conexión como SYSDBA
rman target /                    # Conexión RMAN
lsnrctl status                   # Estado listener
adrci                           # Automatic Diagnostic Repository
```

## 1.3 Procedimientos Operativos

### Arranque y Parada
```bash
# Secuencia de Arranque
1. systemctl start ntpd
2. systemctl start oracle-asm
3. systemctl start oracle-database
4. systemctl start oracle-listener

# Secuencia de Parada
1. systemctl stop oracle-listener
2. systemctl stop oracle-database
3. systemctl stop oracle-asm
4. systemctl stop ntpd
```

### Verificación Post-Arranque
```bash
# Checklist
1. ps -ef | grep pmon            # Verificar procesos
2. df -h                         # Espacio en filesystem
3. ipcs -a                       # Recursos IPC
4. lsnrctl status               # Estado listener
5. asmcmd lsdg                  # Estado diskgroups
```

### Backup del Sistema
```bash
# Directorios Críticos
/etc/                           # Configuración sistema
/opt/oracle/oak/conf/           # Configuración OAK
/opt/oracle/dcs/conf/           # Configuración DCS
/var/log/oracle/                # Logs Oracle

# Comando de Backup
tar czf /backup/system_$(date +%Y%m%d).tar.gz \
    /etc/oracle \
    /opt/oracle/oak/conf \
    /opt/oracle/dcs/conf \
    /var/log/oracle
```

### Monitorización Diaria
```bash
# Scripts de Verificación
#!/bin/bash
# check_oda.sh
echo "=== Sistema ==="
uptime
free -m
df -h

echo "=== Oracle ==="
ps -ef | grep pmon
lsnrctl status

echo "=== ASM ==="
asmcmd lsdg

echo "=== Alertas ==="
tail -n 50 $ORACLE_BASE/diag/rdbms/*/*/trace/alert_*.log
```

## 1.4 Resolución de Problemas

### Problemas Comunes y Soluciones
1. Servicios No Inician
```bash
# Verificar
systemctl status oracle-database
journalctl -u oracle-database

# Solución
systemctl reset-failed oracle-database
systemctl start oracle-database
```

2. Problemas de Red
```bash
# Diagnóstico
ip addr show
ethtool bond0
ping -c 4 gateway
traceroute host

# Solución
nmcli connection reload
systemctl restart network
```

3. Errores de Memoria
```bash
# Verificar
cat /proc/meminfo
grep HugePages_ /proc/meminfo

# Ajustar
echo 60000 > /proc/sys/vm/nr_hugepages
sysctl -w vm.nr_hugepages=60000
```

### Logs Críticos
```bash
# Sistema
/var/log/messages
/var/log/dmesg

# Oracle
$ORACLE_BASE/diag/rdbms/*/*/trace/alert_*.log
$ORACLE_BASE/diag/tnslsnr/*/*/trace/listener.log

# ODA
/opt/oracle/dcs/log/dcs-agent.log
/opt/oracle/oak/log/oak-manager.log
```

## 1.5 Seguridad

### Configuración Firewall
```bash
# Puertos Requeridos
1521/TCP    # Listener
1158/TCP    # EM Express
3872/TCP    # Oracle ASM
5500/TCP    # Grid

# Comandos
firewall-cmd --permanent --add-port=1521/tcp
firewall-cmd --reload
```

### SELinux
```bash
# Verificar Estado
getenforce
sestatus

# Configuración
semanage port -a -t oracle_port_t -p tcp 1521
semanage fcontext -a -t oracle_db_log_t "/u01/app/oracle(/.*)?"
restorecon -R /u01/app/oracle
```

## 1.6 Optimización

### Parámetros Kernel Recomendados
```bash
# /etc/sysctl.conf
kernel.shmmax = 4398046511104
kernel.shmall = 1073741824
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
fs.aio-max-nr = 1048576
fs.file-max = 6815744
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
```

### Límites de Recursos
```bash
# /etc/security/limits.conf
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc     2047
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   soft   memlock   unlimited
oracle   hard   memlock   unlimited
```