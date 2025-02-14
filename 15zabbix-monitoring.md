# Monitoreo de ODA con Zabbix

## 1. Instalación y Configuración del Agente

### 1.1 Instalación del Agente Zabbix
```bash
# Repositorio Zabbix
rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/8/x86_64/zabbix-release-6.0-1.el8.noarch.rpm

# Instalación
dnf install zabbix-agent2

# Configuración principal
cat > /etc/zabbix/zabbix_agent2.conf << EOF
Server=zabbix.example.com
ServerActive=zabbix.example.com
Hostname=oda01.example.com
AllowKey=system.run[*]
Include=/etc/zabbix/zabbix_agent2.d/*.conf
EOF

# Iniciar servicio
systemctl enable --now zabbix-agent2
```

### 1.2 Configuración para ODA
```bash
# Scripts personalizados
cat > /etc/zabbix/scripts/oda_monitor.sh << EOF
#!/bin/bash

case "$1" in
    "system_status")
        odacli describe-system | grep Status
        ;;
    "storage_usage")
        odacli describe-storage | grep "Space Available"
        ;;
    "db_status")
        odacli list-databases | grep Status
        ;;
esac
EOF

chmod +x /etc/zabbix/scripts/oda_monitor.sh

# UserParameters
cat > /etc/zabbix/zabbix_agent2.d/oda.conf << EOF
UserParameter=oda.system_status,/etc/zabbix/scripts/oda_monitor.sh system_status
UserParameter=oda.storage_usage,/etc/zabbix/scripts/oda_monitor.sh storage_usage
UserParameter=oda.db_status,/etc/zabbix/scripts/oda_monitor.sh db_status
EOF
```

## 2. Templates Zabbix

### 2.1 Template ODA Base
```xml
<?xml version="1.0" encoding="UTF-8"?>
<zabbix_export>
    <version>6.0</version>
    <template>
        <template>Template ODA</template>
        <name>Oracle Database Appliance</name>
        <groups>
            <group>Templates/Databases</group>
        </groups>
        <items>
            <item>
                <name>System Status</name>
                <key>oda.system_status</key>
                <type>ZABBIX_ACTIVE</type>
                <value_type>TEXT</value_type>
                <triggers>
                    <trigger>
                        <expression>{str(ERROR)}=1</expression>
                        <name>ODA System Status Error</name>
                        <priority>HIGH</priority>
                    </trigger>
                </triggers>
            </item>
            <item>
                <name>Storage Usage</name>
                <key>oda.storage_usage</key>
                <type>ZABBIX_ACTIVE</type>
                <value_type>FLOAT</value_type>
                <triggers>
                    <trigger>
                        <expression>{last()}&gt;85</expression>
                        <name>Storage Usage High</name>
                        <priority>WARNING</priority>
                    </trigger>
                </triggers>
            </item>
        </items>
        <discovery_rules>
            <discovery_rule>
                <name>Database Discovery</name>
                <key>oda.discover.databases</key>
                <item_prototypes>
                    <item_prototype>
                        <name>Database {#DBNAME} Status</name>
                        <key>oda.db.status[{#DBNAME}]</key>
                    </item_prototype>
                </item_prototypes>
            </discovery_rule>
        </discovery_rules>
    </template>
</zabbix_export>
```

### 2.2 Template Database Monitoring
```yaml
# databases.yaml
zabbix_export:
  version: 6.0
  templates:
    - template: Template ODA Database
      name: ODA Database Monitoring
      items:
        - name: Database Tablespace Usage
          key: db.tablespace.usage[{$DBNAME}]
          type: ZABBIX_ACTIVE
          value_type: FLOAT
          triggers:
            - expression: {last()}>90
              name: Tablespace Nearly Full
              priority: HIGH
              
        - name: Database Performance
          key: db.performance[{$DBNAME}]
          type: ZABBIX_ACTIVE
          value_type: FLOAT
          triggers:
            - expression: {last()}>1000
              name: High Response Time
              priority: WARNING
```

## 3. Monitoreo Personalizado

### 3.1 Scripts de Monitoreo
```python
#!/usr/bin/env python3
# oda_monitor.py

import subprocess
import json

class ODAMonitor:
    def collect_metrics(self):
        metrics = {
            'system': self._get_system_metrics(),
            'storage': self._get_storage_metrics(),
            'databases': self._get_database_metrics()
        }
        return json.dumps(metrics)
    
    def _get_system_metrics(self):
        cmd = "odacli describe-system"
        result = subprocess.run(cmd.split(), 
                              capture_output=True, 
                              text=True)
        return self._parse_system_output(result.stdout)
    
    def _get_storage_metrics(self):
        cmd = "odacli describe-storage"
        result = subprocess.run(cmd.split(), 
                              capture_output=True, 
                              text=True)
        return self._parse_storage_output(result.stdout)

    def send_to_zabbix(self, metrics):
        for key, value in metrics.items():
            cmd = f"zabbix_sender -c /etc/zabbix/zabbix_agent2.conf -k {key} -o '{value}'"
            subprocess.run(cmd, shell=True)
```

### 3.2 Discovery Rules
```bash
#!/bin/bash
# discover_databases.sh

databases=$(odacli list-databases | grep -v "^$" | tail -n +2)

echo '{"data":['

first=true
while read -r line; do
    if [ "$first" = true ]; then
        first=false
    else
        echo ','
    fi
    
    db_name=$(echo $line | awk '{print $2}')
    echo "{ \"{#DBNAME}\": \"$db_name\" }"
done <<< "$databases"

echo ']}'
```

## 4. Alertas y Notificaciones

### 4.1 Media Types
```yaml
# email_config.yaml
media_types:
  - name: ODA Alerts
    type: EMAIL
    smtp_server: smtp.example.com
    smtp_port: 587
    smtp_helo: example.com
    smtp_email: zabbix@example.com
    content_type: text/html
    message_format: |
      <b>Problem</b>: {TRIGGER.NAME}
      <b>Host</b>: {HOST.NAME}
      <b>Severity</b>: {TRIGGER.SEVERITY}
      <b>Time</b>: {EVENT.TIME}
      
      {TRIGGER.DESCRIPTION}
```

### 4.2 Action Rules
```javascript
// alert_conditions.js
{
    "conditions": [
        {
            "type": "trigger_severity",
            "operator": ">=",
            "severity": "HIGH"
        },
        {
            "type": "host_group",
            "operator": "=",
            "value": "Oracle Database Appliance"
        }
    ],
    "operations": [
        {
            "type": "send_message",
            "media_type": "ODA Alerts",
            "send_to": "dba_team"
        }
    ]
}
```

## 5. Dashboards

### 5.1 System Overview
```json
{
    "name": "ODA System Overview",
    "widgets": [
        {
            "type": "graph",
            "name": "System CPU Usage",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "item_id": "system.cpu.util"
        },
        {
            "type": "graph",
            "name": "Storage Usage",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "item_id": "oda.storage_usage"
        }
    ]
}
```

### 5.2 Database Performance
```json
{
    "name": "Database Performance",
    "widgets": [
        {
            "type": "graph",
            "name": "Response Time",
            "item_id": "db.performance"
        },
        {
            "type": "plain_text",
            "name": "Active Sessions",
            "item_id": "db.active_sessions"
        }
    ]
}
```

## 6. Mantenimiento

### 6.1 Backup Configuración
```bash
#!/bin/bash
# backup_zabbix_config.sh

backup_dir="/backup/zabbix"
date=$(date +%Y%m%d)

# Backup configuración
tar czf $backup_dir/zabbix_config_$date.tar.gz \
    /etc/zabbix/zabbix_agent2.conf \
    /etc/zabbix/zabbix_agent2.d/ \
    /etc/zabbix/scripts/

# Limpiar backups antiguos
find $backup_dir -type f -mtime +30 -delete
```

### 6.2 Actualización Templates
```python
#!/usr/bin/env python3
# update_templates.py

import requests
import json

def update_template(template_id, template_file):
    with open(template_file) as f:
        template_data = json.load(f)
    
    response = requests.post(
        'http://zabbix.example.com/api_jsonrpc.php',
        json={
            'jsonrpc': '2.0',
            'method': 'template.update',
            'params': template_data,
            'auth': 'your_auth_token',
            'id': 1
        }
    )
    
    return response.json()
```

## 7. Troubleshooting

### 7.1 Verificación de Agente
```bash
# Test conexión
zabbix_agent2 -t oda.system_status
zabbix_agent2 -p

# Logs
tail -f /var/log/zabbix/zabbix_agent2.log

# Debug mode
zabbix_agent2 -d
```

### 7.2 Validación de Datos
```sql
-- Verificar datos históricos
SELECT itemid, clock, value
FROM history_text
WHERE itemid IN (
    SELECT itemid
    FROM items
    WHERE key_ LIKE 'oda%'
)
ORDER BY clock DESC
LIMIT 100;
```