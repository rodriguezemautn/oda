# Conexión VNC a VMs en ODA

## 1. Configuración en ODA

### Verificar puerto VNC de la VM
```bash
# Listar VMs y sus puertos VNC
virsh dumpxml VM_NAME | grep vnc
virsh domdisplay VM_NAME

# Típicamente el puerto será 5900 + display número
# Ejemplo: :1 = 5901
```

### Configurar Firewall
```bash
# Abrir puerto VNC
firewall-cmd --permanent --add-port=5901/tcp
firewall-cmd --reload

# Verificar
firewall-cmd --list-ports
```

## 2. Acceso Seguro

### Método 1: Túnel SSH
```bash
# En cliente local
ssh -L 5901:localhost:5901 oracle@oda-host

# Conectar con vncviewer
vncviewer localhost:5901
```

### Método 2: Acceso Directo (No recomendado)
```bash
# Conectar directamente
vncviewer oda-host:5901
```

## 3. Scripts de Conexión

### Script de Túnel SSH
```bash
#!/bin/bash
# connect_vm_vnc.sh

VM_NAME=$1
ODA_HOST=$2

# Obtener puerto VNC
PORT=$(ssh oracle@$ODA_HOST "virsh domdisplay $VM_NAME" | cut -d: -f2)
PORT=$((5900 + $PORT))

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
virsh dumpxml $VM_NAME | grep -A 10 "graphics type='vnc'"

# Verificar conectividad
netstat -tulpn | grep vnc
```

## 4. Mejores Prácticas

### Seguridad
```plaintext
1. Usar siempre túnel SSH
2. Cambiar contraseña VNC regularmente
3. Limitar acceso por IP
4. Monitorear conexiones
5. Usar VLAN dedicada
```

### Resolución de Problemas
```bash
# Verificar servicio VNC
ps aux | grep vnc

# Verificar conectividad
telnet localhost 5901

# Logs
tail -f /var/log/messages | grep vnc
```

# Para establecer un túnel SSH y acceder a VNC de manera segura:
```bash
# 1. Crear túnel SSH (desde tu máquina local)
ssh -N root@10.30.6.51 -L 5900/127.0.0.1:5900

# 2. En otra terminal, conectar vncviewer a localhost
vncviewer localhost:5900

# Alternativa usando un solo comando
ssh -L 5900:127.0.0.1:5900 root@10.30.6.51 'vncviewer :0'
```
## Puntos clave:

El puerto 5900 es el estándar para VNC
Usar 127.0.0.1 asegura que solo se acepta tráfico local
La opción -N evita ejecutar un shell remoto
Asegúrate que el servicio VNC esté corriendo en el host remoto
