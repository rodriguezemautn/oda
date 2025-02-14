# Creación de VMs en ODA

## 1. Creación desde Template

### Preparación del Template
```bash
# Listar templates disponibles
odacli list-vmtemplates

# Importar template OVA
odacli import-vmtemplate \
  --name OL8_Template \
  --file /tmp/OL8_template.ova

# Validar template
odacli describe-vmtemplate --name OL8_Template
```

### Creación VM desde OVA
```bash
# Método CLI
odacli create-vm \
  --name prodvm1 \
  --template OL8_Template \
  --memory 16384 \
  --vcpu 4 \
  --network "['pubnet']" \
  --domain example.com \
  --disk-size 100G

# Verificación
odacli describe-vm --name prodvm1
```

### Creación desde QCOW2
```bash
# Importar imagen QCOW2
odacli import-vmtemplate \
  --name Custom_Template \
  --file /tmp/custom.qcow2 \
  --type QCOW2

# Crear VM
odacli create-vm \
  --name customvm1 \
  --template Custom_Template \
  --memory 8192 \
  --vcpu 2
```

## 2. Creación desde ISO

### Preparación ISO
```bash
# Copiar ISO al ODA
scp OracleLinux-R8-U6-x86_64.iso root@oda:/tmp/

# Registrar ISO
odacli register-iso \
  --name OL8_ISO \
  --iso-path /tmp/OracleLinux-R8-U6-x86_64.iso
```

### Instalación desde ISO
```bash
# Crear VM vacía
odacli create-vm \
  --name newvm1 \
  --memory 8192 \
  --vcpu 2 \
  --disk-size 50G \
  --boot-iso OL8_ISO

# Configurar consola VNC
odacli describe-vm --name newvm1 | grep vnc
```

## 3. Post-Configuración

### Network Setup
```bash
# Configurar red
odacli modify-vm \
  --name prodvm1 \
  --network "['pubnet','privnet']"

# Agregar vNIC
odacli modify-vm \
  --name prodvm1 \
  --add-vnic pubnet
```

### Storage Management
```bash
# Agregar disco
odacli modify-vm \
  --name prodvm1 \
  --add-disk 100G

# Configurar almacenamiento
odacli modify-vm \
  --name prodvm1 \
  --disk-size 200G
```

## 4. Templates Personalizados

### Crear Template desde VM
```bash
# Preparar VM
odacli create-vmsnapshot \
  --name prodvm1 \
  --snapshot_name clean_state

# Crear template
odacli create-vmtemplate \
  --name Custom_OL8 \
  --vm prodvm1 \
  --snapshot clean_state
```

### Gestión de Templates
```bash
# Listar templates
odacli list-vmtemplates

# Eliminar template
odacli delete-vmtemplate --name old_template

# Actualizar template
odacli update-vmtemplate \
  --name OL8_Template \
  --version 8.6
```

## 5. Mejores Prácticas

### Planificación
```plaintext
1. Dimensionamiento
   - CPU/Memory según workload
   - Storage con room para crecimiento
   - Network bandwidth requerido

2. Seguridad
   - Hardening base
   - Network isolation
   - Access control
```

### Performance
```bash
# Monitoreo
odacli describe-vm --name prodvm1 --statistics

# CPU pinning
virsh vcpupin prodvm1 0 0-1
virsh vcpupin prodvm1 1 2-3

# Memory tuning
virsh setmem prodvm1 16G
```

## 6. Troubleshooting

### Problemas Comunes
```bash
# VM no inicia
odacli describe-vm --name prodvm1 --details
virsh start --debug prodvm1

# Problemas de red
odacli validate-networkconfig
virsh domiflist prodvm1

# Storage issues
odacli validate-storageconfiguration
virsh domblklist prodvm1
```

### Logs y Diagnóstico
```bash
# Logs VM
tail -f /var/log/messages
virsh dumpxml prodvm1

# Estado sistema
odacli describe-system
odacli list-jobs
```

## 7. Automatización

### Scripts de Deployment
```python
#!/usr/bin/python3
# deploy_vm.py

def create_vm_from_template(name, template, specs):
    cmd = f"""odacli create-vm \
        --name {name} \
        --template {template} \
        --memory {specs['memory']} \
        --vcpu {specs['vcpu']} \
        --network "{specs['networks']}" \
        --disk-size {specs['disk']}"""
    
    return subprocess.run(cmd, shell=True)

def post_deploy_config(name):
    # Network setup
    # Storage config
    # Security hardening
    pass
```