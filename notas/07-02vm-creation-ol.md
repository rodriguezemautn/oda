# Creación de VMs con Oracle Linux en ODA

## 1. Obtención de Imágenes Oracle Linux

### Fuentes Oficiales
```plaintext
1. ISO images:
   - Oracle Linux Installation Media (x86_64 y ARM)
   - https://yum.oracle.com/oracle-linux-downloads.html

2. Cloud Images:
   - Formatos: .ova y .qcow2
   - URL: https://yum.oracle.com/templates/

3. Container Images:
   - Oracle Container Registry
   - GitHub Container Registry
   - Docker Hub
```

### Descarga de Imagen QCOW2
```bash
# Crear directorio
cd /u01/kvm/images

# Descargar imagen
wget https://yum.oracle.com/templates/OracleLinux/OL9/u5/x86_64/OL9U5_x86_64-kvm-b253.qcow2

# Verificar integridad
sha256sum OL9U5_x86_64-kvm-b253.qcow2
```

## 2. Configuración Cloud-Init

### Preparación de Archivos
```bash
# user-data
cat > user-data << 'EOF'
#cloud-config
user: root
password: MyNewPassword
chpasswd: { expire: False }
ssh_pwauth: True
EOF

# meta-data
cat > meta-data << 'EOF'
instance-id: local
EOF

# Generar config.iso
genisoimage -output /u01/kvm/images/config.iso \
  -volid cidata \
  -joliet \
  -rock /u01/kvm/images/user-data /u01/kvm/images/meta-data
```

## 3. Creación de VM

### Usando ODACLI
```bash
# Crear VM
odacli create-vm \
  --name ol9vm \
  --src /u01/kvm/images/OL9U5_x86_64-kvm-b253.qcow2 \
  --vm-name ol9vm1 \
  --memory 4G \
  --vcpu 2 \
  --extra-args /u01/kvm/images/config.iso \
  --vnc-listen=0.0.0.0,port=5949

# Verificar estado
odacli describe-vm --name ol9vm1
```

### Post-Configuración
```bash
# Iniciar VM
odacli start-vm --name ol9vm1

# Verificar acceso
# 1. Usando VNC
vncviewer <oda-host>:5949

# 2. Usando SSH (una vez iniciada)
ssh root@<vm-ip>
```

## 4. Limpieza y Finalización

### Eliminar Archivos Temporales
```bash
# Eliminar archivos cloud-init
rm /u01/kvm/images/user-data
rm /u01/kvm/images/meta-data
rm /u01/kvm/images/config.iso

# Limpiar XML
virsh dumpxml ol9vm > ol9vm.xml
# Editar y eliminar sección:
#<disk type='file' device='cdrom'>
#  <source file='/u01/kvm/images/config.iso'/>
#  <target dev='hda' bus='ide'/>
#</disk>

virsh undefine ol9vm.xml
```

## 5. Opciones de Imagen Oracle Linux

### Tipos Disponibles
```plaintext
1. ISO Installation Media
   - Full instalación
   - Personalización completa
   - Ideal para entornos específicos

2. Cloud Images (.qcow2)
   - Pre-configuradas
   - Inicio rápido
   - Ideal para despliegue automatizado

3. Vagrant Boxes
   - Desarrollo y testing
   - Entornos consistentes
```

### Selección de Imagen
```plaintext
Criterios:
1. Uso previsto
   - Producción: ISO o qcow2
   - Desarrollo: Vagrant
   - Containers: Registry images

2. Requisitos
   - Performance
   - Seguridad
   - Personalización

3. Mantenimiento
   - Actualizaciones
   - Parches
   - Backup
```

## 6. Automatización

### Script de Despliegue
```bash
#!/bin/bash
# deploy_ol_vm.sh

VM_NAME=$1
QCOW2_URL="https://yum.oracle.com/templates/OracleLinux/OL9/u5/x86_64/OL9U5_x86_64-kvm-b253.qcow2"
MEMORY="4G"
VCPU="2"

# Descarga imagen
wget $QCOW2_URL -O /u01/kvm/images/${VM_NAME}.qcow2

# Crear archivos cloud-init
create_cloud_init_files

# Generar config.iso
generate_config_iso

# Crear VM
odacli create-vm \
  --name $VM_NAME \
  --src /u01/kvm/images/${VM_NAME}.qcow2 \
  --memory $MEMORY \
  --vcpu $VCPU \
  --extra-args /u01/kvm/images/config.iso

# Limpieza
cleanup_temp_files
```

## 7. Troubleshooting

### Problemas Comunes
```bash
# Error en cloud-init
tail -f /var/log/cloud-init.log

# Problemas de red
ip addr show
nmcli connection show

# Issues de storage
df -h
lvs
pvs
```

### Verificaciones
```bash
# Estado VM
odacli describe-vm --name ol9vm1

# Logs sistema
journalctl -u cloud-init
dmesg | grep -i error

# Network
ping -c 4 <vm-ip>
telnet <vm-ip> 22
```