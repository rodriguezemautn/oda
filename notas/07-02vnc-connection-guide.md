# Conexión VNC a VMs en ODA

## 1. Configuración Inicial

### Verificar Puerto VNC
```bash
# Verificar configuración VNC de la VM
odacli describe-vm --name ol9vm | grep vnc

# Puerto configurado en creación
--vnc_listen=0.0.0.0,port=5949
```

### Preparar Acceso
```bash
# Verificar firewall
firewall-cmd --list-ports
firewall-cmd --permanent --add-port=5949/tcp
firewall-cmd --reload

# Verificar servicio
netstat -tulpn | grep 5949
```

## 2. Métodos de Conexión

### Túnel SSH (Recomendado)
```bash
# 1. Crear túnel SSH desde cliente local
ssh -L 5949:127.0.0.1:5949 oracle@oda-host

# 2. Conectar con vncviewer
vncviewer localhost:5949
```

### Conexión Directa (No recomendada)
```bash
# Conectar directamente
vncviewer oda-host:5949
```

## 3. Scripts de Conexión

### Script de Túnel SSH
```bash
#!/bin/bash
# connect_vm_vnc.sh

VM_NAME=$1
ODA_HOST=$2

# Obtener puerto VNC
PORT=$(odacli describe-vm --name $VM_NAME | grep vnc | awk -F'port=' '{print $2}')

# Crear túnel SSH
echo "Creando túnel SSH para $VM_NAME en puerto $PORT"
ssh -f -N -L $PORT:localhost:$PORT oracle@$ODA_HOST

# Iniciar VNC viewer
vncviewer localhost:$PORT
```

### Script de Verificación
```bash
#!/bin/bash
# check_vnc.sh

VM_NAME=$1

# Verificar estado VNC
echo "=== Estado VNC para $VM_NAME ==="
odacli describe-vm --name $VM_NAME | grep -A 2 "VNC Info"

# Verificar puerto
netstat -tulpn | grep vnc
```

## 4. Troubleshooting

### Problemas Comunes
```bash
# 1. Puerto bloqueado
iptables -L | grep 5949
ss -tulpn | grep 5949

# 2. Servicio VNC no responde
virsh dumpxml ol9vm | grep vnc
virsh reset ol9vm

# 3. Problemas de conexión
ping oda-host
telnet oda-host 5949
```

### Logs Relevantes
```bash
# Revisar logs
tail -f /var/log/messages | grep vnc
tail -f /var/log/secure | grep ssh

# Estado de la VM
odacli describe-vm --name ol9vm --details
```

## 5. Mejores Prácticas

### Seguridad
```plaintext
1. Siempre usar túnel SSH
2. Cambiar contraseña VNC regularmente
3. Limitar acceso por IP
4. Monitorear conexiones activas
```

### Performance
```plaintext
1. Ajustar calidad VNC según red
2. Cerrar sesiones inactivas
3. Limitar número de conexiones
```

## 6. Post-Conexión

### Verificación Sistema
```bash
# Dentro de la VM
df -h           # Espacio en disco
free -m         # Memoria
top             # Procesos
ip addr         # Red
```

### Configuración Inicial
```bash
# Setup básico
hostnamectl set-hostname ol9vm
timedatectl set-timezone America/Argentina/Buenos_Aires
yum update -y
```