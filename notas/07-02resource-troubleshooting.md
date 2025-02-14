# Troubleshooting de Recursos en VMs

## 1. Error de Asignación de CPU

### Mensaje de Error
```plaintext
Status Code: 400
"Not enough hotpluggable vCPUs to reach the requested new vCPU count [2], live decrease first"
```

### Verificación de Recursos
```bash
# Verificar CPUs disponibles
odacli list-cpucores
odacli describe-cpupool

# Verificar asignación actual
odacli describe-vm --name <vm_name>
virsh vcpuinfo <vm_name>
```

### Resolución
```bash
# 1. Reducir CPUs primero
odacli modify-vm \
  --name <vm_name> \
  --vcpu 1

# 2. Esperar a que complete
odacli describe-job -i <job_id>

# 3. Realizar el scale up
odacli modify-vm \
  --name <vm_name> \
  --vcpu 2
```

## 2. Mejores Prácticas

### Planificación de Recursos
```plaintext
1. Verificar recursos antes de modificar
2. Realizar cambios incrementales
3. Mantener margen de CPU disponible
4. Documentar configuración óptima
```

### Monitoreo
```bash
# CPU usage
top -p $(pgrep -d',' qemu)

# Memory
free -m
vmstat 1 5

# Procesos VM
ps aux | grep <vm_name>
```

## 3. Prevención

### Validaciones Pre-Cambio
```bash
# 1. Verificar CPU pool
odacli list-cpupools

# 2. Revisar carga actual
odacli describe-vm --name <vm_name> --statistics

# 3. Validar configuración
odacli validate-system
```

### Documentación
```bash
# Registrar configuración
echo "$(date): Modificación VM $VM_NAME" >> /var/log/vm_changes.log
echo "CPU anterior: $OLD_CPU" >> /var/log/vm_changes.log
echo "CPU nuevo: $NEW_CPU" >> /var/log/vm_changes.log
```