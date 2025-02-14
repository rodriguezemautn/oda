# Notas de Virtualización ODA

## 1. KVM (Kernel-based Virtual Machine)

### Componentes Clave
```plaintext
- QEMU: Emulador
- libvirt: API de virtualización
- virtio: Drivers optimizados
- bridges: Networking virtual
```

### Comandos Esenciales
```bash
# Gestión de VMs
virsh list --all
virsh start|stop|destroy VM_NAME
virsh edit VM_NAME

# Storage
virsh vol-list default
virsh pool-list --details

# Networking
virsh net-list
brctl show
```

## 2. DBSystems

### Verificación y Monitoreo
```bash
# Estado del Grid
/u01/app/19.24.0.0/grid/bin/crsctl stat res -t
/u01/app/19.24.0.0/grid/bin/crsctl check cluster

# Recursos
/u01/app/19.24.0.0/grid/bin/crsctl status resource -t
/u01/app/19.24.0.0/grid/bin/srvctl status database
```

### Logs Importantes
```plaintext
/u01/app/grid/diag/crs
/u01/app/grid/log
/u01/app/oracle/diag
```

## 3. Acceso VNC

### Configuración Segura
```bash
# Setup VNC
vncserver :1 -geometry 1024x768
vncpasswd

# Túnel SSH
ssh -L 5901:localhost:5901 oda_host

# Firewall
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload
```

### Mejores Prácticas
```plaintext
1. Túnel SSH obligatorio
2. Contraseñas robustas
3. VNC sobre VLAN dedicada
4. Acceso limitado por IP
5. Monitoreo de sesiones
```

## 4. Aspectos Críticos

### Performance
```plaintext
1. CPU Pinning
2. Memory Ballooning
3. NUMA Awareness
4. I/O Throttling
5. Network QoS
```

### Alta Disponibilidad
```plaintext
1. Live Migration
2. VM Failover
3. Storage Replication
4. Network Redundancy
5. Resource Fencing
```

## 5. Scripts de Automatización

### Monitoreo VM
```python
#!/usr/bin/python3
import libvirt
import sys

def check_vm_status():
    conn = libvirt.open('qemu:///system')
    if conn == None:
        print('Failed to connect to hypervisor')
        sys.exit(1)

    domains = conn.listAllDomains(0)
    for domain in domains:
        print(f"Domain: {domain.name()}")
        print(f"State: {domain.state()}")
        print(f"Max Memory: {domain.maxMemory()}")
        print("---")

    conn.close()
```

### Backup VMs
```bash
#!/bin/bash
# backup_vms.sh

VM_LIST=$(virsh list --name)
BACKUP_DIR="/backup/vms"
DATE=$(date +%Y%m%d)

for VM in $VM_LIST; do
    echo "Backing up $VM"
    virsh snapshot-create-as $VM ${VM}_${DATE}
    virsh dumpxml $VM > $BACKUP_DIR/${VM}_${DATE}.xml
done
```

## 6. Recomendaciones de Diseño

### Arquitectura
```plaintext
1. Separación de redes
   - Management
   - Storage
   - Application
   - Backup

2. Storage Layout
   - ASM para DBs
   - File System para VMs
   - Separate IOPS pools
```

### Seguridad
```plaintext
1. Network Isolation
2. Resource Limits
3. Access Control
4. Monitoring
5. Backup Strategy
```

## 7. Troubleshooting

### Problemas Comunes
```plaintext
1. VM no inicia
   - Verificar XML
   - Logs libvirt
   - Storage access

2. Performance
   - CPU steal time
   - Memory pressure
   - I/O bottlenecks

3. Network
   - Bridge config
   - VLAN tagging
   - MTU issues
```

### Comandos Diagnóstico
```bash
# Sistema
top -p $(pgrep -d',' qemu)
iostat -xz 1
sar -n DEV 1

# VM Específica
virsh domstats VM_NAME
virsh dommemstat VM_NAME
virsh domblklist VM_NAME
```