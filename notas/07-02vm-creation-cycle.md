# Ciclo de Creación y Verificación de VMs en ODA

## 1. Comando de Creación
```bash
# Sintaxis detallada
odacli create-vm \
  --name ol9vm \                      # Nombre de la VM
  --src /u01/kvm/images/OL9U5_x86_64-kvm-b253.qcow2 \  # Imagen origen
  --vms repotest \                    # Storage repository
  --memory 4G \                       # Memoria asignada
  --vcpu 2 \                          # CPUs virtuales
  --cpu_pool_name odb1 \             # Pool de CPU
  --extra-args /u01/kvm/images/config.iso \  # Config cloud-init
  --vnc_listen=0.0.0.0,port=5949     # Configuración VNC
```

## 2. Seguimiento del Proceso

### Verificación de Job
```bash
# Obtener ID del job
odacli describe-job -i d742328c-35e5-423f-9a7b-e56ac2bf5714

# Formato de salida:
Job ID: d742328c-35e5-423f-9a7b-e56ac2bf5714
Status: Success
Created: <timestamp>
Message: VM Creation Completed
```

### Estado de la VM
```bash
# Listar VMs
odacli list-vms

# Campos mostrados:
- Name: ol9vm
- VM Storage: repotest
- Current State: ONLINE
- Target State: ONLINE
- Created: 2025-02-07 10:55:04 ART
- Updated: 2025-02-07 10:55:04 ART
```

## 3. Verificaciones Post-Creación

### Estado de Recursos
```bash
# Detalles de la VM
odacli describe-vm --name ol9vm

# Verificar red
odacli list-networkinterfaces --name ol9vm

# Verificar storage
odacli describe-vm-storage --name ol9vm
```

### Acceso y Conectividad
```bash
# Verificar VNC
netstat -tulpn | grep 5949

# Verificar estado servicio
virsh list --all | grep ol9vm

# Test conectividad
ping <vm_ip>
```

## 4. Troubleshooting

### Logs Relevantes
```bash
# Logs de creación
tail -f /var/log/messages | grep -i vm
tail -f /var/log/odacli/dcs-agent.log

# Estado detallado
odacli describe-vm --name ol9vm --details
```

### Problemas Comunes
```bash
# VM no inicia
virsh start ol9vm --debug

# Problemas de red
odacli validate-networkconfig
virsh domiflist ol9vm

# Storage issues
odacli validate-storageconfiguration
virsh domblklist ol9vm
```