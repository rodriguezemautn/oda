# Gestión de Almacenamiento en ODA

## 1. ASM Diskgroups

### Estado y Configuración
```bash
# Verificar diskgroups
asmcmd lsdg
# Resultados:
# DATA: 41043M total, 13377M libre
# RECO: 41043M total, 2594M libre

# Detalles de redundancia
State    Type  Rebal  Sector  Block  AU Total_MB Free_MB
MOUNTED  FLEX  N      512     4096   4194304
```

## 2. Sistemas de Archivos

### Uso de Espacio
```bash
# Verificar espacio
df -h
# Puntos clave:
- /dev/mapper/VGExaCM-LogVolRoot: 3G (35% usado)
- /opt: 98G (26% usado)
- /u05/app/sharedrepo/repository: 206G (14% usado)
```

### Configuración fstab
```bash
# /etc/fstab
/dev/mapper/VolGroupVm-LogVolRoot /        ext4    defaults        1 1
/dev/mapper/VolGroupVm-LogVolSwap swap     swap    defaults        0 0
/dev/shm                          /dev/shm tmpfs   defaults        0 0

# NFS mounts
10.30.5.106:/asmfs /asmfs             nfs defaults 0 0
192.168.17.2:/u05/app/sharedrepo/acfs nfs defaults 0 0
```

## 3. Particionamiento

### Estructura de Discos
```bash
# Listado de discos (lsblk)
NAME          MAJ:MIN SIZE    TYPE MOUNTPOINT
sda           8:0     6.2T    disk 
├─sda1        8:1     633.4G  part 
├─sda2        8:2     633.4G  part
...
vda           11:0    37M     disk
vdb           251:0   101G    disk /boot
```

## 4. Exports y Compartición

### NFS Exports
```bash
# /etc/exports
/opt/oracle/oak/pkgrepos 192.168.17.2(rw,sync_no_subtree_check,crossmnt,no_root_squash)
/u05/app/sharedrepo/acfs 192.168.17.2(rw,sync_no_subtree_check,crossmnt,no_root_squash)
```

## 5. Scripts de Monitoreo

### Verificación de Storage
```bash
#!/bin/bash
# check_storage.sh

echo "=== ASM Diskgroups ==="
asmcmd lsdg

echo "=== Filesystem Usage ==="
df -h | grep -E "Filesystem|^/dev|^/u05"

echo "=== NFS Mounts ==="
mount | grep nfs
```

## 6. Mejores Prácticas

### ASM
```plaintext
1. Monitoreo:
   - Espacio libre en diskgroups
   - Estado de rebalanceo
   - Redundancia

2. Alertas:
   - < 20% espacio libre
   - Fallas en discos
   - Problemas de redundancia
```

### Filesystem
```plaintext
1. Particionamiento:
   - Separar OS de datos
   - Considerar snapshots
   - Planificar crecimiento

2. NFS:
   - Validar permisos
   - Monitorear performance
   - Backup de exports
```

## 7. Comandos Útiles

### Mantenimiento
```bash
# Verificar ASM
asmcmd lsdg
asmcmd lsdsk
asmcmd ls -l

# Filesystem
df -h
du -sh /*
findmnt

# Discos
lsblk
fdisk -l
```

### Troubleshooting
```bash
# Problemas NFS
showmount -e
nfsstat
mountstats

# ASM Issues
asmcmd ping
asmcmd chkdsk
kfed read /dev/sda1
```