# Módulo: Administración de Sistemas ODA

## 1. Gestión del Sistema Operativo

### 1.1 Monitorización Diaria
```bash
#!/bin/bash
# daily_checks.sh

echo "=== System Status Check ==="
uptime
free -h
df -h
vmstat 1 5

echo "=== Network Status ==="
netstat -rn
ip addr show
ping -c 4 gateway_ip

echo "=== Oracle Services ==="
systemctl status oracle-database
systemctl status oracle-asm
crsctl stat res -t
```

### 1.2 Gestión de Usuarios
```bash
# Usuarios del Sistema
useradd -g oinstall -G dba,asmdba oracle
useradd -g oinstall -G asmadmin,asmdba,asmoper grid

# Permisos críticos
chmod 6751 $ORACLE_HOME/bin/oracle
chmod 6751 $GRID_HOME/bin/asmcmd
```

## 2. Storage Management

### 2.1 ASM Administration
```bash
# Monitoreo ASM
asmcmd lsdg
asmcmd iostat -G

# Rebalanceo
alter diskgroup DATA rebalance power 8;
asmcmd rebalance '+DATA' -p 8
```

### 2.2 Filesystem Management
```bash
# Monitoreo
df -h
du -sh /u01/*
findmnt -t xfs

# Expansión
lvextend -L +10G /dev/mapper/vg_oda-lv_u01
xfs_growfs /u01
```

## 3. Network Administration

### 3.1 Configuración
```bash
# Interfaces
cat > /etc/sysconfig/network-scripts/ifcfg-bond0 << EOF
DEVICE=bond0
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=none
ONBOOT=yes
IPADDR=192.168.1.10
NETMASK=255.255.255.0
BONDING_OPTS="mode=4 miimon=100"
EOF

# VLANs
vconfig add bond0 123
ip link set bond0.123 up
```

### 3.2 Monitoreo
```bash
# Performance
sar -n DEV 1 5
iperf3 -s  # servidor
iperf3 -c server_ip  # cliente

# Troubleshooting
tcpdump -i bond0 port 1521
netstat -anp | grep LISTEN
```

## 4. Backup y Recovery

### 4.1 Sistema Operativo
```bash
# Backup Configuración
tar czf /backup/os_config_$(date +%Y%m%d).tar.gz \
    /etc/oracle \
    /etc/sysconfig/network-scripts \
    /etc/hosts \
    /etc/resolv.conf

# Recovery
cd /
tar xzf /backup/os_config_*.tar.gz
```

### 4.2 Automatización
```python
# backup_manager.py
import os
import subprocess
from datetime import datetime

class BackupManager:
    def __init__(self):
        self.backup_root = "/backup"
        self.retention_days = 30
    
    def backup_config(self):
        date = datetime.now().strftime("%Y%m%d")
        backup_file = f"{self.backup_root}/config_{date}.tar.gz"
        
        cmd = f"tar czf {backup_file} /etc/oracle /etc/sysconfig"
        subprocess.run(cmd.split())
    
    def cleanup_old_backups(self):
        cmd = f"find {self.backup_root} -type f -mtime +{self.retention_days} -delete"
        subprocess.run(cmd.split())
```

## 5. Seguridad

### 5.1 Hardening
```bash
# SELinux
semanage port -a -t oracle_port_t -p tcp 1521
semanage fcontext -a -t oracle_db_log_t "/u01/app/oracle(/.*)?"

# Firewall
firewall-cmd --permanent --add-port=1521/tcp
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" port port="1521" protocol="tcp" accept'
```

### 5.2 Auditoría
```bash
# Configurar auditoría
cat >> /etc/audit/rules.d/audit.rules << EOF
-w /etc/oracle -p wa -k oracle_config
-w /u01/app/oracle -p wa -k oracle_binaries
-w /etc/passwd -p wa -k user_modification
EOF

# Revisar logs
aureport --start today
ausearch -k oracle_config
```

## 6. Performance Tuning

### 6.1 Sistema Operativo
```bash
# Kernel Parameters
cat >> /etc/sysctl.conf << EOF
kernel.shmmax = 4398046511104
kernel.shmall = 1073741824
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
fs.aio-max-nr = 1048576
fs.file-max = 6815744
EOF

# Huge Pages
echo vm.nr_hugepages = 60000 >> /etc/sysctl.conf
sysctl -p
```

### 6.2 I/O Performance
```bash
# I/O Scheduler
echo "deadline" > /sys/block/sda/queue/scheduler

# Read Ahead
blockdev --setra 16384 /dev/sda

# I/O Stats
iostat -xk 5 3
```

## 7. Automatización y Scripts

### 7.1 Monitorización Automatizada
```python
# system_monitor.py
import psutil
import time
import logging

class SystemMonitor:
    def __init__(self):
        self.logger = logging.getLogger('system_monitor')
        
    def check_resources(self):
        metrics = {
            'cpu': psutil.cpu_percent(interval=1),
            'memory': psutil.virtual_memory().percent,
            'swap': psutil.swap_memory().percent,
            'disk': {part.mountpoint: psutil.disk_usage(part.mountpoint).percent 
                    for part in psutil.disk_partitions()}
        }
        return metrics
    
    def alert_if_needed(self, metrics):
        thresholds = {
            'cpu': 80,
            'memory': 85,
            'swap': 60,
            'disk': 85
        }
        
        for metric, value in metrics.items():
            if isinstance(value, dict):
                for mount, usage in value.items():
                    if usage > thresholds['disk']:
                        self.logger.warning(f"High disk usage on {mount}: {usage}%")
            elif value > thresholds[metric]:
                self.logger.warning(f"High {metric} usage: {value}%")
```

### 7.2 Mantenimiento Automatizado
```bash
#!/bin/bash
# maintenance.sh

# Limpieza de logs
find /var/log -name "*.log" -mtime +30 -delete
find /var/log/oracle -name "*.trc" -mtime +7 -delete

# Verificación de servicios
systemctl --failed | mail -s "Failed Services Report" admin@example.com

# Backup de configuración
tar czf /backup/weekly_config_$(date +%Y%m%d).tar.gz /etc/oracle
```

## 8. Documentación

### 8.1 Inventario
```python
# inventory.py
def generate_inventory():
    inventory = {
        'hardware': get_hardware_info(),
        'software': get_installed_packages(),
        'network': get_network_config(),
        'storage': get_storage_layout()
    }
    return inventory

def get_hardware_info():
    cmd = "dmidecode"
    return subprocess.getoutput(cmd)

def get_installed_packages():
    cmd = "rpm -qa"
    return subprocess.getoutput(cmd)
```

### 8.2 Procedimientos
```markdown
# Procedimientos Operativos

## Diario
1. Verificar servicios
2. Revisar espacio
3. Monitorear rendimiento
4. Verificar backups

## Semanal
1. Análisis de logs
2. Limpieza de temporales
3. Verificación de parches
4. Backup de configuración

## Mensual
1. Análisis de capacidad
2. Revisión de seguridad
3. Optimización de recursos
4. Actualización de documentación
```