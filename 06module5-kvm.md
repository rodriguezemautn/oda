# Módulo 5: Virtualización con KVM en ODA

## 5.1 Arquitectura KVM en ODA

### Componentes
```plaintext
├── Hypervisor (KVM)
│   ├── QEMU
│   ├── libvirt
│   └── Kernel Modules
├── Storage
│   ├── ASM
│   ├── VDisks
│   └── File Systems
├── Networking
│   ├── Bridges
│   ├── VLANs
│   └── Virtual Switches
└── Management
    ├── oakcli
    ├── odacli
    └── virsh
```

### Directorios Clave
```bash
/opt/oracle/oak/              # OAK binaries
/opt/oracle/oak/vm/          # VM configuration
/var/lib/libvirt/           # libvirt data
/etc/libvirt/              # libvirt config
/var/log/libvirt/         # KVM logs
```

## 5.2 Gestión de VMs

### Creación de VMs
```bash
# Usando ODACLI
odacli create-vm \
    --name testvm1 \
    --memory 8 \
    --vcpu 4 \
    --os_type OL7U9 \
    --disk_size 100G \
    --network "['priv1','pub1']" \
    --repository repo1

# Usando virsh
virt-install \
    --name testvm1 \
    --memory 8192 \
    --vcpus 4 \
    --disk path=/VMS/testvm1.qcow2,size=100 \
    --os-variant ol7.9 \
    --network bridge=priv1
```

### Operaciones Básicas
```bash
# Estado de VMs
odacli list-vms
odacli describe-vm --name testvm1

# Control de VMs
odacli start-vm --name testvm1
odacli stop-vm --name testvm1
odacli modify-vm --name testvm1 --memory 16

# Snapshot
odacli create-vmsnapshot --name testvm1 --snap_name snap01
odacli restore-vmsnapshot --name testvm1 --snap_name snap01
```

## 5.3 Storage Management

### ASM para VMs
```sql
-- Crear Diskgroup para VMs
CREATE DISKGROUP VM_DATA NORMAL REDUNDANCY
  DISK '/dev/disk/by-path/pci-0000:00:1f.2-ata-1'
  ATTRIBUTE 'compatible.asm' = '19.0';

-- Monitorear espacio
SELECT name, total_mb, free_mb, 
       ROUND((free_mb/total_mb)*100,2) as free_pct
FROM v$asm_diskgroup;
```

### Virtual Disks
```bash
# Crear vDisk
odacli create-vdisk \
    --vdisk_name data_disk1 \
    --vdisk_size 500G \
    --diskgroup DATA

# Asociar vDisk
odacli modify-vm \
    --name testvm1 \
    --attach_vdisk data_disk1

# Listar vDisks
odacli list-vdisks
odacli describe-vdisk --name data_disk1
```

## 5.4 Networking

### Virtual Networks
```bash
# Crear red virtual
odacli create-vnetwork \
    --name internal_net1 \
    --bridge bridge1 \
    --vlan-id 100 \
    --subnet 192.168.10.0 \
    --netmask 255.255.255.0

# Configurar interfaces
odacli modify-vm \
    --name testvm1 \
    --network "['internal_net1']"
```

### Network Script Examples
```bash
# /etc/sysconfig/network-scripts/ifcfg-bridge1
DEVICE=bridge1
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
DELAY=0
NM_CONTROLLED=no
IPADDR=192.168.10.1
NETMASK=255.255.255.0
```

## 5.5 Performance y Monitorización

### Monitoreo de Recursos
```bash
# CPU y Memoria
virt-top
virsh domstats testvm1 --stats

# I/O Stats
virsh domblkstat testvm1
virsh domifstat testvm1

# Scripts de monitoreo
#!/bin/bash
# monitor_vms.sh
for vm in $(virsh list --name); do
    echo "=== $vm ==="
    virsh dominfo $vm
    virsh cpu-stats $vm
    virsh domstats $vm --stats
done
```

### Resource Management
```bash
# CPU Pinning
virsh vcpupin testvm1 0 0-1
virsh vcpupin testvm1 1 2-3

# Memory Control
virsh setmem testvm1 8G
virsh memtune testvm1 --hard-limit 8G
```

## 5.6 Alta Disponibilidad

### VM HA Configuration
```bash
# Configurar HA
odacli configure-vmha \
    --name testvm1 \
    --priority HIGH \
    --policy RESTART

# Verificar estado
odacli describe-vmha --name testvm1
```

### Migration
```bash
# Live Migration
virsh migrate --live testvm1 qemu+ssh://desthost/system

# Storage Migration
virsh migrate --live --persistent --copy-storage-all \
    testvm1 qemu+ssh://desthost/system
```

## 5.7 Backup y Recovery

### VM Backup
```bash
# Backup completo
odacli create-vmbackup \
    --name testvm1 \
    --type FULL \
    --location /backup/vms

# Backup incremental
odacli create-vmbackup \
    --name testvm1 \
    --type INCREMENTAL \
    --location /backup/vms
```

### Restore
```bash
# Restore completo
odacli restore-vmbackup \
    --name testvm1 \
    --backup_id backup_20240205

# Restore a punto específico
odacli restore-vmbackup \
    --name testvm1 \
    --backup_id backup_20240205 \
    --time "2024-02-05 14:30:00"
```

## 5.8 Automatización y Scripting

### Deployment Automation
```python
# vm_deploy.py
import subprocess
import json

def deploy_vm(config):
    cmd = [
        'odacli', 'create-vm',
        '--name', config['name'],
        '--memory', str(config['memory']),
        '--vcpu', str(config['vcpu']),
        '--os_type', config['os_type']
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)

def setup_network(vm_name, networks):
    cmd = [
        'odacli', 'modify-vm',
        '--name', vm_name,
        '--network', json.dumps(networks)
    ]
    subprocess.run(cmd)
```

### Monitoring Framework
```python
# vm_monitor.py
import libvirt
import time

class VMMonitor:
    def __init__(self):
        self.conn = libvirt.open('qemu:///system')
    
    def get_vm_stats(self, vm_name):
        domain = self.conn.lookupByName(vm_name)
        stats = domain.getCPUStats(True)
        mem_stats = domain.memoryStats()
        return {
            'cpu_time': stats[0]['cpu_time'],
            'memory_used': mem_stats['actual']/1024
        }
    
    def monitor_all_vms(self):
        for domain in self.conn.listAllDomains():
            stats = self.get_vm_stats(domain.name())
            print(f"{domain.name()}: {stats}")
```

## 5.9 Mejores Prácticas

### Performance
```bash
# CPU
- Use CPU pinning for critical VMs
- Avoid overcommitment
- Monitor CPU steal time

# Memory
- Configure huge pages
- Use memory ballooning
- Monitor swap usage

# Storage
- Use virtio drivers
- Implement storage multipathing
- Monitor I/O statistics
```

### Security
```bash
# SELinux Configuration
setsebool -P virt_use_fusefs 1
setsebool -P virt_use_nfs 1

# Network Security
- Implement network isolation
- Use VLANs for separation
- Configure firewall rules
```

## 5.10 Referencias

### Documentación Oracle
1. [ODA Virtualization](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/virtual/)
2. [KVM on ODA](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/kvm/)

### MOS Notes
- Note 2244356.1: KVM Best Practices on ODA
- Note 2287819.1: VM Performance Tuning
- Note 2365765.1: VM Troubleshooting Guide

### Recursos Adicionales
1. [KVM Documentation](https://www.linux-kvm.org/page/Documents)
2. [libvirt Documentation](https://libvirt.org/docs.html)
3. [QEMU Documentation](https://www.qemu.org/docs/master/)