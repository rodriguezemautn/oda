# Módulo 13: Automatización Empresarial de ODA

## 13.1 Ansible para ODA

### Estructura del Proyecto
```yaml
oda_automation/
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── oda_servers.yml
├── roles/
│   ├── oda_setup/
│   ├── oda_database/
│   ├── oda_network/
│   └── oda_monitoring/
└── playbooks/
    ├── setup_oda.yml
    ├── manage_databases.yml
    └── configure_network.yml
```

### Inventario y Variables
```yaml
# inventory/hosts
[oda_servers]
oda1.example.com
oda2.example.com

# inventory/group_vars/oda_servers.yml
oda_version: "19.12"
default_database_version: "19.12.0.0"
backup_location: "/backup"
monitoring_config:
  metrics_retention: 30
  alert_email: "dba@example.com"
```

### Roles y Tasks

1. ODA Setup Role
```yaml
# roles/oda_setup/tasks/main.yml
- name: Validate ODA System
  oda_system:
    state: validated
  register: system_status

- name: Configure Storage
  oda_storage:
    diskgroup: DATA
    redundancy: HIGH
    disks: 
      - /dev/sde
      - /dev/sdf
  when: system_status.changed

- name: Configure Network
  oda_network:
    name: "{{ item.name }}"
    type: "{{ item.type }}"
    ip: "{{ item.ip }}"
    netmask: "{{ item.netmask }}"
  loop: "{{ network_configs }}"
```

2. Database Management
```yaml
# roles/oda_database/tasks/main.yml
- name: Create Database
  oda_database:
    name: "{{ db_name }}"
    version: "{{ db_version }}"
    shape: "{{ db_shape }}"
    class: "{{ db_class }}"
    charset: AL32UTF8
    state: present
  register: db_creation

- name: Configure Backup
  oda_backup:
    database: "{{ db_name }}"
    policy: "{{ backup_policy }}"
    location: "{{ backup_location }}"
  when: db_creation.changed
```

### Playbooks

1. Setup Completo
```yaml
# playbooks/setup_oda.yml
---
- hosts: oda_servers
  become: yes
  roles:
    - role: oda_setup
    - role: oda_database
    - role: oda_network
    - role: oda_monitoring

  tasks:
    - name: Verify Setup
      oda_system:
        state: verified
      register: setup_verification

    - name: Send Notification
      mail:
        to: "{{ admin_email }}"
        subject: "ODA Setup Complete"
        body: "Setup completed successfully"
      when: setup_verification.changed
```

2. Gestión de Bases de Datos
```yaml
# playbooks/manage_databases.yml
---
- hosts: oda_servers
  become: yes
  vars_prompt:
    - name: "operation"
      prompt: "Enter operation (create/backup/patch)"
      private: no

  tasks:
    - name: Database Operations
      include_role:
        name: oda_database
        tasks_from: "{{ operation }}.yml"
```

## 13.2 Terraform Provider para ODA

### Provider Configuration
```hcl
# provider.tf
terraform {
  required_providers {
    oda = {
      source = "oracle/oda"
      version = "~> 2.0"
    }
  }
}

provider "oda" {
  endpoint = "https://oda.example.com"
  username = var.oda_username
  password = var.oda_password
}
```

### Resource Definitions

1. Database Resources
```hcl
# database.tf
resource "oda_database" "prod" {
  name    = "PRODDB"
  version = "19.12.0.0"
  shape   = "odb1"
  class   = "OLTP"

  storage_config {
    data_size    = "1024G"
    recovery_size = "512G"
  }

  backup_config {
    policy    = "BASIC"
    schedule  = "daily"
    retention = 30
  }
}

resource "oda_database_backup" "prod_backup" {
  database_id = oda_database.prod.id
  type        = "FULL"
  tag         = "monthly"
  
  depends_on = [oda_database.prod]
}
```

2. Network Resources
```hcl
# network.tf
resource "oda_network" "public" {
  name    = "public_network"
  type    = "PUBLIC"
  ip      = "192.168.1.10"
  netmask = "255.255.255.0"
  gateway = "192.168.1.1"
}

resource "oda_network" "private" {
  name    = "private_network"
  type    = "PRIVATE"
  ip      = "10.0.0.10"
  netmask = "255.255.255.0"
}
```

3. Storage Resources
```hcl
# storage.tf
resource "oda_diskgroup" "data" {
  name        = "DATA"
  redundancy  = "HIGH"
  disk_paths  = ["/dev/sde", "/dev/sdf"]
}

resource "oda_diskgroup" "reco" {
  name        = "RECO"
  redundancy  = "HIGH"
  disk_paths  = ["/dev/sdg", "/dev/sdh"]
}
```

### Módulos

1. Database Module
```hcl
# modules/database/main.tf
variable "db_name" {}
variable "db_version" {}
variable "db_class" {}

resource "oda_database" "db" {
  name    = var.db_name
  version = var.db_version
  class   = var.db_class
}

output "database_id" {
  value = oda_database.db.id
}
```

2. Networking Module
```hcl
# modules/network/main.tf
module "network" {
  source = "./modules/network"

  network_configs = {
    public = {
      name    = "public_net"
      type    = "PUBLIC"
      ip      = "192.168.1.10"
    }
    private = {
      name    = "private_net"
      type    = "PRIVATE"
      ip      = "10.0.0.10"
    }
  }
}
```

## 13.3 APIs para Integración Empresarial

### REST API Client
```python
# oda_api_client.py
import requests
import json

class ODAClient:
    def __init__(self, host, username, password):
        self.base_url = f"https://{host}:7093/api"
        self.session = requests.Session()
        self.authenticate(username, password)
    
    def authenticate(self, username, password):
        auth_url = f"{self.base_url}/auth"
        response = self.session.post(
            auth_url,
            json={"username": username, "password": password},
            verify=False
        )
        self.session.headers["Authorization"] = f"Bearer {response.json()['token']}"
    
    def create_database(self, config):
        return self.session.post(
            f"{self.base_url}/database",
            json=config
        ).json()
    
    def get_system_status(self):
        return self.session.get(
            f"{self.base_url}/system/status"
        ).json()
```

### Integration Examples

1. ServiceNow Integration
```python
# servicenow_integration.py
from pysnow import Client
from oda_api_client import ODAClient

class ServiceNowIntegration:
    def __init__(self, oda_client, snow_client):
        self.oda = oda_client
        self.snow = snow_client
    
    def create_incident(self, alert):
        incident = {
            'short_description': f"ODA Alert: {alert['message']}",
            'description': json.dumps(alert, indent=2),
            'urgency': self._map_urgency(alert['severity']),
            'assignment_group': 'Database Team'
        }
        return self.snow.resource('incident').create(payload=incident)
    
    def update_cmdb(self, system_info):
        ci = {
            'name': system_info['name'],
            'version': system_info['version'],
            'status': system_info['status']
        }
        return self.snow.resource('cmdb_ci_oracle').update(query={
            'name': system_info['name']
        }, payload=ci)
```

2. Monitoring Integration
```python
# prometheus_integration.py
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

class PrometheusExporter:
    def __init__(self, oda_client, push_gateway):
        self.oda = oda_client
        self.gateway = push_gateway
        self.registry = CollectorRegistry()
        
        self.metrics = {
            'cpu_usage': Gauge('oda_cpu_usage', 
                             'ODA CPU Usage', 
                             registry=self.registry),
            'memory_usage': Gauge('oda_memory_usage', 
                                'ODA Memory Usage', 
                                registry=self.registry)
        }
    
    def collect_metrics(self):
        status = self.oda.get_system_status()
        
        self.metrics['cpu_usage'].set(status['cpu_usage'])
        self.metrics['memory_usage'].set(status['memory_usage'])
        
        push_to_gateway(self.gateway, 
                       job='oda_metrics', 
                       registry=self.registry)
```

### Automatización de Workflows

1. Jenkins Pipeline
```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        ODA_CREDS = credentials('oda-credentials')
    }
    
    stages {
        stage('Validate') {
            steps {
                sh 'odacli validate-system'
            }
        }
        
        stage('Backup') {
            steps {
                sh """
                    odacli create-backup \
                        --database ${params.DATABASE} \
                        --level FULL
                """
            }
        }
        
        stage('Patch') {
            steps {
                sh """
                    odacli update-server \
                        --version ${params.PATCH_VERSION}
                """
            }
        }
    }
    
    post {
        always {
            notifyTeams("Pipeline completed: ${currentBuild.result}")
        }
    }
}
```

2. GitHub Actions
```yaml
# .github/workflows/oda-automation.yml
name: ODA Automation

on:
  push:
    branches: [ main ]
  schedule:
    - cron: '0 0 * * *'

jobs:
  database-maintenance:
    runs-on: self-hosted
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Run System Validation
        run: odacli validate-system
        
      - name: Update Statistics
        run: |
          sqlplus / as sysdba <<EOF
          EXEC DBMS_STATS.GATHER_SCHEMA_STATS('SCHEMA_NAME');
          EXIT;
          EOF
```

## 13.4 Mejores Prácticas

### Seguridad
```plaintext
1. Gestión de Credenciales
   - Usar vault para secretos
   - Rotación regular de credenciales
   - Auditoría de accesos

2. Control de Acceso
   - RBAC para APIs
   - Network segmentation
   - SSL/TLS para comunicaciones
```

### Monitoreo
```plaintext
1. Logging
   - Centralizar logs
   - Retención apropiada
   - Alerting basado en patrones

2. Métricas
   - KPIs críticos
   - Dashboards automatizados
   - Trending y análisis
```

## Referencias

### Documentation
1. [ODA REST API Reference](https://docs.oracle.com/en/engineered-systems/oracle-database-appliance/19.12/rest-api/)
2. [Ansible Documentation](https://docs.ansible.com/ansible/latest/index.html)
3. [Terraform Provider Development](https://www.terraform.io/docs/extend/index.html)

### Ejemplos y Recursos
1. [GitHub - ODA Automation Examples](https://github.com/oracle/oda-automation)
2. [Ansible Galaxy - ODA Roles](https://galaxy.ansible.com/oracle/oda)
3. [Terraform Registry - ODA Provider](https://registry.terraform.io/providers/oracle/oda)