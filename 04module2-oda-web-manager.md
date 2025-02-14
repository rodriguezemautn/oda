# Módulo 2: ODA Web Manager

## Documentación Oficial y Recursos

### Oracle Documentation
1. [Oracle Database Appliance Web Console](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/manage-web/)
2. [ODA Administration API](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/rest-api/)
3. [ODA Security Guide](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/security/)

### White Papers
1. "Best Practices for Oracle Database Appliance Web Console Management" (Doc ID 2144642.1)
2. "ODA Web Console Performance Tuning Guide" (Doc ID 2167692.1)
3. "Security Guidelines for ODA Web Console" (Doc ID 2178543.1)

### MOS Notes Relevantes
- Note 2521012.1: ODA Web Console Troubleshooting Guide
- Note 2558075.1: ODA Web Console Access Problems
- Note 2672356.1: ODA Web Console Best Practices

## 2.1 Arquitectura Detallada

### Componentes del Sistema
```bash
# Servicios Principales
/opt/oracle/dcs/                  # Database Configuration Services
    ./bin/                       # Binarios DCS
    ./conf/                      # Configuración
    ./java/                      # Java Runtime
    ./lib/                       # Librerías
    ./log/                       # Logs DCS
    
/opt/oracle/oak/                  # Oracle Appliance Kit
    ./bin/                       # Binarios OAK
    ./conf/                      # Configuración
    ./lib/                       # Librerías
    ./log/                       # Logs OAK
    
# Archivos de Configuración
/opt/oracle/dcs/conf/
    ./dcs.conf                   # Configuración principal DCS
    ./dcs-user.conf             # Configuración usuarios
    ./web-ssl.conf              # Configuración SSL
    ./api-keys.conf             # API keys
```

### Puertos y Protocolos
```plaintext
7093/TCP - HTTPS Web Console
7094/TCP - REST API
7095/TCP - Internal Services
7096/TCP - Metrics Collection
```

## 2.2 Seguridad Avanzada

### Configuración SSL/TLS
```bash
# Generación de certificado
openssl req -new -newkey rsa:4096 -nodes \
    -keyout oda.key -out oda.csr

# Instalación
/opt/oracle/dcs/bin/odacli configure-ssl \
    --key-file oda.key \
    --cert-file oda.crt
```

### Hardening Recomendado
```bash
# /opt/oracle/dcs/conf/security.conf
SSL_PROTOCOLS="TLSv1.2 TLSv1.3"
ALLOWED_CIPHERS="HIGH:!aNULL:!MD5:!RC4"
SESSION_TIMEOUT=1800
FAILED_LOGIN_ATTEMPTS=3
PASSWORD_COMPLEXITY="STRONG"
```

## 2.3 APIs REST Detalladas

### Autenticación
```python
import requests
import json

def get_token():
    url = "https://oda-host:7093/api/auth"
    headers = {
        'Content-Type': 'application/json'
    }
    data = {
        'username': 'admin',
        'password': 'password'
    }
    response = requests.post(url, headers=headers, 
                           data=json.dumps(data), verify=False)
    return response.json()['token']
```

### Endpoints Principales
```bash
# Database Operations
GET    /api/database/list
POST   /api/database/create
DELETE /api/database/{id}
PATCH  /api/database/{id}/update

# Storage Management
GET    /api/storage/diskgroups
POST   /api/storage/expand
GET    /api/storage/performance

# System Monitoring
GET    /api/system/metrics
GET    /api/system/alerts
POST   /api/system/patch
```

## 2.4 Monitorización Avanzada

### Métricas Custom
```sql
-- Crear vista personalizada
CREATE VIEW v_oda_metrics AS
SELECT 
    metric_name,
    value,
    collection_time,
    database_name
FROM 
    dcs_metrics
WHERE 
    metric_type = 'PERFORMANCE';

-- Consulta para dashboard
SELECT * FROM v_oda_metrics
WHERE collection_time > SYSDATE - 1/24
ORDER BY collection_time DESC;
```

### Alertas Personalizadas
```json
{
  "alert_rule": {
    "name": "High_CPU_Usage",
    "metric": "CPU_UTILIZATION",
    "threshold": 85,
    "duration": "5m",
    "severity": "CRITICAL",
    "notification": {
      "email": ["dba@company.com"],
      "webhook": "https://alerts.company.com/oda"
    }
  }
}
```

## 2.5 Scripts de Automatización

### Backup Automation
```python
# backup_automation.py
import requests
import schedule
import time

def backup_all_databases():
    token = get_token()
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }
    
    # Get all databases
    dbs = requests.get(
        'https://oda-host:7093/api/database/list',
        headers=headers, verify=False
    ).json()
    
    # Backup each database
    for db in dbs:
        backup_data = {
            'database_id': db['id'],
            'type': 'FULL',
            'compression': True
        }
        requests.post(
            'https://oda-host:7093/api/database/backup',
            headers=headers,
            json=backup_data,
            verify=False
        )

# Schedule daily backup
schedule.every().day.at("23:00").do(backup_all_databases)

while True:
    schedule.run_pending()
    time.sleep(60)
```

### Monitoring Script
```bash
#!/bin/bash
# monitor_oda.sh

# Config
ODA_HOST="oda-host"
API_PORT="7093"
AUTH_TOKEN=$(get_auth_token)

# Check System Health
check_system_health() {
    curl -k -X GET \
        -H "Authorization: Bearer $AUTH_TOKEN" \
        "https://$ODA_HOST:$API_PORT/api/system/health"
}

# Check Database Status
check_databases() {
    curl -k -X GET \
        -H "Authorization: Bearer $AUTH_TOKEN" \
        "https://$ODA_HOST:$API_PORT/api/database/list" \
        | jq '.[] | {name: .dbName, status: .status}'
}

# Main
main() {
    echo "=== ODA Health Check ==="
    check_system_health
    echo "=== Database Status ==="
    check_databases
}

main | tee -a /var/log/oda_monitor.log
```

## 2.6 Integración con Enterprise Manager

### Configuración
```bash
# /opt/oracle/dcs/conf/em_integration.conf
EM_URL="https://em.company.com:7802/em"
EM_USER="sysman"
EM_PASSWORD_FILE="/opt/oracle/dcs/conf/.em_pwd"
METRIC_UPLOAD_INTERVAL=300
```

### Métricas Custom
```xml
<!-- custom_metrics.xml -->
<metric_collection>
    <metrics>
        <metric>
            <name>custom_backup_status</name>
            <query>
                SELECT status, count(*) 
                FROM backup_info 
                GROUP BY status
            </query>
            <threshold>
                <critical>status = 'FAILED'</critical>
                <warning>status = 'WARNING'</warning>
            </threshold>
        </metric>
    </metrics>
</metric_collection>
```

## Referencias Adicionales

### Blogs Técnicos
1. [Oracle Database Insider](https://blogs.oracle.com/database/)
   - "ODA Web Console Best Practices"
   - "Security Hardening for ODA"

2. [Oracle DBA Blog](https://oracle-dba.com/oda/)
   - "ODA Web Console Performance Tuning"
   - "Automation Scripts for ODA"

### Comunidad
1. Oracle Community for ODA:
   - [Web Console Discussion](https://community.oracle.com/tech/apps-infra/categories/oracle_database_appliance_web_console)
   - [API Integration Forum](https://community.oracle.com/tech/apps-infra/categories/oda_api)

### Videos Técnicos
1. Oracle Learning Library:
   - "ODA Web Console Deep Dive"
   - "Security Best Practices for ODA"

2. Oracle Technology Network:
   - "ODA Performance Monitoring"
   - "Automation and Integration Tips"