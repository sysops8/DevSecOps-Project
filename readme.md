# DevSecOps Pipeline Project: Deploy Netflix Clone на Proxmox

## Описание проекта

Полная реализация DevSecOps пайплайна для развертывания Netflix Clone с использованием Jenkins, Kubernetes, Prometheus, Grafana и ArgoCD на домашней инфраструктуре Proxmox.

## Архитектура инфраструктуры

### Топология сети
```
Internet (Grey IP)
    ↓
Router TP-LINK (10.0.10.1)
    ↓
[10.0.10.0/24 Network]
    ├── Proxmox Host (10.0.10.200)
    │   ├── VM1: Jenkins Server (10.0.10.201)
    │   ├── VM2: Kubernetes Master (10.0.10.202)
    │   ├── VM3: Kubernetes Worker 1 (10.0.10.203)
    │   ├── VM4: Kubernetes Worker 2 (10.0.10.204)
    │   └── VM5: Monitoring Server (10.0.10.205)
    └── Windows 10 Workstation (DHCP)
```

### Спецификация виртуальных машин

| VM | Назначение | CPU | RAM | Disk | IP |
|---|---|---|---|---|---|
| VM1 | Jenkins + SonarQube + Docker | 4 vCPU | 8 GB | 100 GB | 10.0.10.201 |
| VM2 | Kubernetes Master | 4 vCPU | 8 GB | 80 GB | 10.0.10.202 |
| VM3 | Kubernetes Worker 1 | 4 vCPU | 8 GB | 80 GB | 10.0.10.203 |
| VM4 | Kubernetes Worker 2 | 4 vCPU | 8 GB | 80 GB | 10.0.10.204 |
| VM5 | Prometheus + Grafana | 2 vCPU | 4 GB | 50 GB | 10.0.10.205 |

**Итого ресурсов:** 18 vCPU, 36 GB RAM, 390 GB Disk

---

## Подготовка инфраструктуры

### 1. Terraform конфигурация для Proxmox

Создайте структуру проекта:
```bash
mkdir -p ~/devsecops-proxmox/{terraform,ansible,manifests}
cd ~/devsecops-proxmox/terraform
```

Создайте файл `terraform/provider.tf`:
```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

provider "proxmox" {
  pm_api_url      = "https://10.0.10.200:8006/api2/json"
  pm_user         = "root@pam"
  pm_password     = "your_proxmox_password"
  pm_tls_insecure = true
}
```

Создайте файл `terraform/variables.tf`:
```hcl
variable "ssh_public_key" {
  description = "SSH public key для доступа к VM"
  type        = string
}

variable "template_name" {
  description = "Имя шаблона Ubuntu в Proxmox"
  default     = "ubuntu-22.04-template"
}

variable "proxmox_node" {
  description = "Имя ноды Proxmox"
  default     = "pve"
}
```

Создайте файл `terraform/main.tf`:
```hcl
# Jenkins Server
resource "proxmox_vm_qemu" "jenkins" {
  name        = "jenkins-server"
  target_node = var.proxmox_node
  clone       = var.template_name
  
  cores   = 4
  memory  = 8192
  scsihw  = "virtio-scsi-pci"
  
  disk {
    size    = "100G"
    type    = "scsi"
    storage = "local-lvm"
  }
  
  network {
    model  = "virtio"
    bridge = "vmbr0"
  }
  
  ipconfig0 = "ip=10.0.10.201/24,gw=10.0.10.1"
  
  sshkeys = var.ssh_public_key
  
  lifecycle {
    ignore_changes = [network]
  }
}

# Kubernetes Master
resource "proxmox_vm_qemu" "k8s_master" {
  name        = "k8s-master"
  target_node = var.proxmox_node
  clone       = var.template_name
  
  cores   = 4
  memory  = 8192
  scsihw  = "virtio-scsi-pci"
  
  disk {
    size    = "80G"
    type    = "scsi"
    storage = "local-lvm"
  }
  
  network {
    model  = "virtio"
    bridge = "vmbr0"
  }
  
  ipconfig0 = "ip=10.0.10.202/24,gw=10.0.10.1"
  
  sshkeys = var.ssh_public_key
}

# Kubernetes Workers
resource "proxmox_vm_qemu" "k8s_worker" {
  count       = 2
  name        = "k8s-worker-${count.index + 1}"
  target_node = var.proxmox_node
  clone       = var.template_name
  
  cores   = 4
  memory  = 8192
  scsihw  = "virtio-scsi-pci"
  
  disk {
    size    = "80G"
    type    = "scsi"
    storage = "local-lvm"
  }
  
  network {
    model  = "virtio"
    bridge = "vmbr0"
  }
  
  ipconfig0 = "ip=10.0.10.${203 + count.index}/24,gw=10.0.10.1"
  
  sshkeys = var.ssh_public_key
}

# Monitoring Server
resource "proxmox_vm_qemu" "monitoring" {
  name        = "monitoring-server"
  target_node = var.proxmox_node
  clone       = var.template_name
  
  cores   = 2
  memory  = 4096
  scsihw  = "virtio-scsi-pci"
  
  disk {
    size    = "50G"
    type    = "scsi"
    storage = "local-lvm"
  }
  
  network {
    model  = "virtio"
    bridge = "vmbr0"
  }
  
  ipconfig0 = "ip=10.0.10.205/24,gw=10.0.10.1"
  
  sshkeys = var.ssh_public_key
}
```

Создайте файл `terraform/outputs.tf`:
```hcl
output "jenkins_ip" {
  value = proxmox_vm_qemu.jenkins.default_ipv4_address
}

output "k8s_master_ip" {
  value = proxmox_vm_qemu.k8s_master.default_ipv4_address
}

output "k8s_workers_ips" {
  value = proxmox_vm_qemu.k8s_worker[*].default_ipv4_address
}

output "monitoring_ip" {
  value = proxmox_vm_qemu.monitoring.default_ipv4_address
}
```

### 2. Создание Ubuntu Cloud-Init шаблона в Proxmox

На хосте Proxmox выполните:

```bash
# Скачать Ubuntu 22.04 Cloud Image
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Создать VM
qm create 9000 --name ubuntu-22.04-template --memory 2048 --net0 virtio,bridge=vmbr0

# Импортировать диск
qm importdisk 9000 jammy-server-cloudimg-amd64.img local-lvm

# Настроить VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1

# Изменить размер диска
qm resize 9000 scsi0 +30G

# Конвертировать в шаблон
qm template 9000
```

### 3. Создание SSH ключей и развертывание VM

На вашей Windows машине (в WSL или Git Bash):

```bash
# Создать SSH ключ
ssh-keygen -t rsa -b 4096 -f ~/.ssh/devsecops_rsa -N ""

# Создать terraform.tfvars
cat > terraform.tfvars <<EOF
ssh_public_key = "$(cat ~/.ssh/devsecops_rsa.pub)"
proxmox_node   = "pve"  # Замените на имя вашей ноды
EOF

# Развернуть инфраструктуру
terraform init
terraform plan
terraform apply -auto-approve
```

---

## PHASE 1: Установка и настройка Jenkins Server

### 1.1. Подключение к Jenkins VM

```bash
ssh -i ~/.ssh/devsecops_rsa ubuntu@10.0.10.201
```

### 1.2. Настройка hostname и обновление системы

```bash
sudo hostnamectl set-hostname jenkins-server
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget vim
```

### 1.3. Установка Docker

```bash
# Установка Docker
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Добавить пользователя в группу docker
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

### 1.4. Клонирование репозитория проекта

```bash
cd ~
git clone https://github.com/N4si/DevSecOps-Project.git
cd DevSecOps-Project
```

### 1.5. Получение TMDB API Key

1. Откройте браузер и перейдите на https://www.themoviedb.org/
2. Создайте аккаунт и войдите
3. Перейдите в Settings → API
4. Создайте новый API key (выберите Developer)
5. Заполните форму и получите API Key (v3 auth)

Сохраните ключ, он понадобится позже.

### 1.6. Тестовый запуск приложения в Docker

```bash
# Собрать образ с вашим API ключом
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .

# Запустить контейнер
docker run -d --name netflix-test -p 8081:80 netflix:latest

# Проверить работу
curl http://localhost:8081

# Остановить и удалить тестовый контейнер
docker stop netflix-test
docker rm netflix-test
```

Откройте браузер на Windows машине: http://10.0.10.201:8081

---

## PHASE 2: Security Tools (SonarQube и Trivy)

### 2.1. Установка SonarQube через Docker

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:lts-community

# Проверить статус
docker ps | grep sonarqube
```

Доступ: http://10.0.10.201:9000
- Логин: `admin`
- Пароль: `admin` (потребуется смена при первом входе)

### 2.2. Настройка SonarQube

1. Войдите в SonarQube
2. Смените пароль администратора
3. Создайте токен:
   - Administration → Security → Users → Administrator
   - Нажмите на токены → Generate Token
   - Имя: `jenkins-token`
   - Тип: `Global Analysis Token`
   - Скопируйте и сохраните токен

### 2.3. Установка Trivy

```bash
sudo apt-get install -y wget apt-transport-https gnupg lsb-release

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo gpg --dearmor -o /usr/share/keyrings/trivy.gpg

echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list

sudo apt-get update
sudo apt-get install -y trivy

# Проверка
trivy --version
```

### 2.4. Тестирование Trivy

```bash
# Сканирование образа
trivy image netflix:latest

# Сканирование файловой системы
trivy fs ~/DevSecOps-Project
```

---

## PHASE 3: CI/CD Setup - Jenkins

### 3.1. Установка Java 17

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre

java -version
```

### 3.2. Установка Jenkins

```bash
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt-get update
sudo apt-get install -y jenkins

sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

### 3.3. Получение начального пароля Jenkins

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Скопируйте пароль.

### 3.4. Первоначальная настройка Jenkins

1. Откройте браузер: http://10.0.10.201:8080
2. Вставьте начальный пароль
3. Выберите "Install suggested plugins"
4. Создайте admin пользователя:
   - Username: `admin`
   - Password: `<ваш-пароль>`
   - Full name: `Admin`
   - Email: `admin@localhost`
5. Jenkins URL: `http://10.0.10.201:8080/`

### 3.5. Установка необходимых плагинов

**Manage Jenkins → Plugins → Available Plugins**

Установите следующие плагины (без перезагрузки):
- Eclipse Temurin Installer
- SonarQube Scanner
- NodeJS Plugin
- Docker Pipeline
- Docker Plugin
- Docker Commons
- Docker API
- docker-build-step
- OWASP Dependency-Check
- Email Extension Plugin

После установки нажмите "Restart Jenkins when installation is complete"

### 3.6. Настройка Global Tool Configuration

**Manage Jenkins → Tools**

**JDK:**
- Name: `jdk17`
- Install automatically: ✓
- Version: `jdk-17.0.8+7`

**NodeJS:**
- Name: `node16`
- Install automatically: ✓
- Version: `NodeJS 16.20.2`

**SonarQube Scanner:**
- Name: `sonar-scanner`
- Install automatically: ✓
- Version: `SonarQube Scanner 5.0.1.3006`

**OWASP Dependency-Check:**
- Name: `DP-Check`
- Install automatically: ✓
- Install from github.com
- Version: `dependency-check 8.4.0`

**Docker:**
- Name: `docker`
- Install automatically: ✓
- Download from docker.com
- Docker version: `latest`

Нажмите **Save**

### 3.7. Настройка учетных данных

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

**1. SonarQube Token:**
- Kind: `Secret text`
- Secret: `<ваш-sonarqube-token>`
- ID: `Sonar-token`
- Description: `SonarQube Token`

**2. DockerHub Credentials:**
- Kind: `Username with password`
- Username: `<ваш-dockerhub-username>`
- Password: `<ваш-dockerhub-password>`
- ID: `docker`
- Description: `DockerHub Credentials`

### 3.8. Настройка SonarQube Server

**Manage Jenkins → Configure System → SonarQube servers**

- Name: `sonar-server`
- Server URL: `http://10.0.10.201:9000`
- Server authentication token: выберите `Sonar-token`

Нажмите **Save**

### 3.9. Настройка Webhook в SonarQube

1. Войдите в SonarQube: http://10.0.10.201:9000
2. Administration → Configuration → Webhooks
3. Create:
   - Name: `jenkins`
   - URL: `http://10.0.10.201:8080/sonarqube-webhook/`
4. Create

### 3.10. Добавление Jenkins в группу Docker

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

### 3.11. Создание Jenkins Pipeline

**Jenkins Dashboard → New Item**

- Name: `Netflix-DevSecOps`
- Type: `Pipeline`
- OK

**Pipeline Configuration:**

В разделе Pipeline выберите:
- Definition: `Pipeline script`
- Script:

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TMDB_API_KEY = '<YOUR_TMDB_API_KEY>'
        DOCKER_IMAGE = '<your-dockerhub-username>/netflix'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', 
                    odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh """
                            docker build --build-arg TMDB_V3_API_KEY=${TMDB_API_KEY} -t ${DOCKER_IMAGE}:latest .
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_IMAGE}:latest > trivyimage.txt"
            }
        }
        stage('Deploy to Container') {
            steps {
                sh """
                    docker stop netflix || true
                    docker rm netflix || true
                    docker run -d --name netflix -p 8081:80 ${DOCKER_IMAGE}:latest
                """
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: '*.txt', allowEmptyArchive: true
        }
    }
}
```

**Важно:** Замените:
- `<YOUR_TMDB_API_KEY>` на ваш TMDB API ключ
- `<your-dockerhub-username>` на ваш username в DockerHub

Нажмите **Save** и затем **Build Now**

---

## PHASE 4: Monitoring (Prometheus & Grafana)

### 4.1. Подключение к Monitoring Server

```bash
ssh -i ~/.ssh/devsecops_rsa ubuntu@10.0.10.205
```

### 4.2. Установка Prometheus

```bash
# Создать пользователя для Prometheus
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Скачать Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/

# Создать директории
sudo mkdir -p /data /etc/prometheus

# Переместить бинарники и конфиги
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

# Установить права
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

### 4.3. Настройка Prometheus

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Содержимое:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['10.0.10.201:8080']

  - job_name: 'kubernetes-nodes'
    static_configs:
      - targets: 
        - '10.0.10.202:9100'
        - '10.0.10.203:9100'
        - '10.0.10.204:9100'
```

### 4.4. Создание systemd сервиса для Prometheus

```bash
sudo nano /etc/systemd/system/prometheus.service
```

Содержимое:
```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

```bash
# Запустить Prometheus
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Доступ: http://10.0.10.205:9090

### 4.5. Установка Node Exporter

```bash
# Создать пользователя
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Скачать Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz

# Переместить бинарник
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
```

### 4.6. Создание systemd сервиса для Node Exporter

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Содержимое:
```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

```bash
# Запустить Node Exporter
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

### 4.7. Установка Node Exporter на остальных серверах

Повторите шаги 4.5 и 4.6 на следующих серверах:
- Jenkins Server (10.0.10.201)
- K8s Master (10.0.10.202)
- K8s Worker 1 (10.0.10.203)
- K8s Worker 2 (10.0.10.204)

### 4.8. Установка Grafana

```bash
# Установить зависимости
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common

# Добавить GPG ключ
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Добавить репозиторий
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Установить Grafana
sudo apt-get update
sudo apt-get install -y grafana

# Запустить Grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

Доступ: http://10.0.10.205:3000
- Логин: `admin`
- Пароль: `admin` (потребуется смена)

### 4.9. Настройка Grafana

1. Добавить Data Source:
   - Configuration (⚙️) → Data Sources → Add data source
   - Выбрать Prometheus
   - URL: `http://localhost:9090`
   - Save & Test

2. Импортировать Dashboard:
   - Create (+) → Import
   - Dashboard ID: `1860` (Node Exporter Full)
   - Load → Select Prometheus → Import
   
3. Дополнительные полезные дашборды:
   - `3662` - Prometheus 2.0 Overview
   - `13332` - Kubernetes Cluster Monitoring
   - `9964` - Jenkins Performance and Health Overview

---

## PHASE 5: Kubernetes Cluster

### 5.1. Подготовка всех нод (Master и Workers)

Выполните на **всех** нодах Kubernetes (10.0.10.202, 10.0.10.203, 10.0.10.204):

```bash
# Обновить систему
sudo apt update && sudo apt upgrade -y

# Отключить swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Загрузить модули ядра
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Настроить sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Установить containerd
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# Установить зависимости для Kubernetes
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Добавить ключ и репозиторий Kubernetes
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Установить kubeadm, kubelet и kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 5.2. Инициализация Master ноды

На **Master** (10.0.10.202):

```bash
# Установить hostname
sudo hostnamectl set-hostname k8s-master

# Инициализировать кластер
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=10.0.10.202 \
  --control-plane-endpoint=10.0.10.202

# Настроить kubectl для обычного пользователя
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Проверить статус
kubectl get nodes
kubectl get pods -A
```

**ВАЖНО:** Сохраните команду `kubeadm join`, которая появится в выводе. Она будет выглядеть примерно так:

```bash
kubeadm join 10.0.10.202:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### 5.3. Установка сетевого плагина (Flannel)

На **Master**:

```bash
# Установить Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Подождать пока все поды запустятся
kubectl get pods -n kube-flannel
kubectl get pods -n kube-system
```

### 5.4. Присоединение Worker нод

На **каждом Worker** (10.0.10.203, 10.0.10.204):

```bash
# Worker 1
sudo hostnamectl set-hostname k8s-worker-1

# Worker 2
sudo hostnamectl set-hostname k8s-worker-2

# Выполнить команду join (замените на вашу команду из вывода kubeadm init)
sudo kubeadm join 10.0.10.202:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### 5.5. Проверка кластера

На **Master**:

```bash
# Проверить ноды
kubectl get nodes

# Должно показать:
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-master     Ready    control-plane   5m    v1.28.x
# k8s-worker-1   Ready    <none>          2m    v1.28.x
# k8s-worker-2   Ready    <none>          2m    v1.28.x

# Проверить все поды
kubectl get pods -A
```

### 5.6. Установка Helm

На **Master**:

```bash
# Скачать и установить Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Проверить версию
helm version
```

### 5.7. Установка Metrics Server

```bash
# Установить Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Пропатчить для работы с самоподписанными сертификатами
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Проверить
kubectl top nodes
```

---

## PHASE 6: ArgoCD

### 6.1. Установка ArgoCD

На **Master**:

```bash
# Создать namespace
kubectl create namespace argocd

# Установить ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Подождать пока все поды запустятся
kubectl get pods -n argocd -w
```

### 6.2. Доступ к ArgoCD UI

```bash
# Изменить тип сервиса на NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Получить порт
kubectl get svc argocd-server -n argocd

# Получить начальный пароль
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Доступ: http://10.0.10.202:<NodePort>
- Username: `admin`
- Password: (из команды выше)

### 6.3. Установка ArgoCD CLI (опционально)

```bash
# Скачать ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Логин
argocd login 10.0.10.202:<NodePort> --insecure

# Сменить пароль
argocd account update-password
```

### 6.4. Создание Kubernetes манифестов для Netflix приложения

На **Master** создайте директорию для манифестов:

```bash
mkdir -p ~/netflix-k8s-manifests
cd ~/netflix-k8s-manifests
```

Создайте файл `namespace.yaml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netflix
```

Создайте файл `deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netflix-app
  namespace: netflix
  labels:
    app: netflix
spec:
  replicas: 2
  selector:
    matchLabels:
      app: netflix
  template:
    metadata:
      labels:
        app: netflix
    spec:
      containers:
      - name: netflix
        image: <your-dockerhub-username>/netflix:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

Создайте файл `service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: netflix-service
  namespace: netflix
spec:
  type: NodePort
  selector:
    app: netflix
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30007
```

### 6.5. Создание Git репозитория для манифестов

**Вариант 1: Использовать GitHub**

1. Создайте новый репозиторий на GitHub: `netflix-k8s-deployment`
2. Загрузите манифесты:

```bash
cd ~/netflix-k8s-manifests
git init
git add .
git commit -m "Initial Netflix K8s manifests"
git branch -M main
git remote add origin https://github.com/<your-username>/netflix-k8s-deployment.git
git push -u origin main
```

**Вариант 2: Использовать локальный Git (если нет доступа к GitHub)**

```bash
# На Jenkins сервере создать bare репозиторий
ssh ubuntu@10.0.10.201
sudo mkdir -p /opt/git/netflix-k8s-deployment.git
cd /opt/git/netflix-k8s-deployment.git
sudo git init --bare
sudo chown -R ubuntu:ubuntu /opt/git/netflix-k8s-deployment.git

# На Master ноде
cd ~/netflix-k8s-manifests
git init
git add .
git commit -m "Initial Netflix K8s manifests"
git remote add origin ubuntu@10.0.10.201:/opt/git/netflix-k8s-deployment.git
git push -u origin master
```

### 6.6. Настройка ArgoCD Application

**Через UI:**

1. Откройте ArgoCD UI
2. Нажмите "+ NEW APP"
3. Заполните:
   - Application Name: `netflix`
   - Project: `default`
   - Sync Policy: `Automatic`
   - ✓ Prune Resources
   - ✓ Self Heal
   - Repository URL: `https://github.com/<your-username>/netflix-k8s-deployment.git`
   - Path: `.`
   - Cluster URL: `https://kubernetes.default.svc`
   - Namespace: `netflix`
4. Нажмите **CREATE**

**Через CLI:**

```bash
argocd app create netflix \
  --repo https://github.com/<your-username>/netflix-k8s-deployment.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace netflix \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 6.7. Синхронизация приложения

```bash
# Создать namespace
kubectl apply -f ~/netflix-k8s-manifests/namespace.yaml

# Синхронизировать через CLI
argocd app sync netflix

# Или через UI: нажать "SYNC" в приложении

# Проверить статус
kubectl get all -n netflix
argocd app get netflix
```

### 6.8. Доступ к приложению

```bash
# Проверить NodePort
kubectl get svc -n netflix

# Открыть в браузере
# http://10.0.10.202:30007  или
# http://10.0.10.203:30007  или
# http://10.0.10.204:30007
```

---

## PHASE 7: Интеграция Jenkins с Kubernetes

### 7.1. Обновление Jenkins Pipeline для деплоя в Kubernetes

На **Jenkins Server** обновите Pipeline:

**Вариант 1: Прямой деплой через kubectl (простой)**

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        TMDB_API_KEY = '<YOUR_TMDB_API_KEY>'
        DOCKER_IMAGE = '<your-dockerhub-username>/netflix'
        KUBECONFIG = '/home/ubuntu/.kube/config'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    '''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', 
                    odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh """
                            docker build --build-arg TMDB_V3_API_KEY=${TMDB_API_KEY} -t ${DOCKER_IMAGE}:${BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest
                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${DOCKER_IMAGE}:latest > trivyimage.txt"
            }
        }
        stage('Update K8s Manifests') {
            steps {
                script {
                    sh """
                        git clone https://github.com/<your-username>/netflix-k8s-deployment.git
                        cd netflix-k8s-deployment
                        sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|g' deployment.yaml
                        git config user.email "jenkins@localhost"
                        git config user.name "Jenkins"
                        git add deployment.yaml
                        git commit -m "Update image to ${BUILD_NUMBER}"
                        git push https://<github-token>@github.com/<your-username>/netflix-k8s-deployment.git main
                    """
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: '*.txt', allowEmptyArchive: true
        }
    }
}
```

### 7.2. Настройка SSH доступа к K8s Master для Jenkins

На **Jenkins Server**:

```bash
# Создать SSH ключ для Jenkins
sudo -u jenkins ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""

# Скопировать публичный ключ
sudo cat /var/lib/jenkins/.ssh/id_rsa.pub
```

На **K8s Master**:

```bash
# Добавить публичный ключ в authorized_keys
echo "<jenkins-public-key>" >> ~/.ssh/authorized_keys

# Скопировать kubeconfig для Jenkins
cat ~/.kube/config
```

На **Jenkins Server**:

```bash
# Создать kubeconfig для Jenkins
sudo -u jenkins mkdir -p /var/lib/jenkins/.kube
sudo -u jenkins nano /var/lib/jenkins/.kube/config
# Вставить содержимое kubeconfig с K8s Master

# Тест подключения
sudo -u jenkins kubectl get nodes
```

---

## PHASE 8: Мониторинг Kubernetes

### 8.1. Установка Prometheus Node Exporter в Kubernetes

На **K8s Master**:

```bash
# Добавить Helm репозиторий
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Создать namespace
kubectl create namespace monitoring

# Установить Node Exporter
helm install node-exporter prometheus-community/prometheus-node-exporter \
  --namespace monitoring
```

### 8.2. Установка kube-state-metrics

```bash
# Установить kube-state-metrics
helm install kube-state-metrics prometheus-community/kube-state-metrics \
  --namespace monitoring
```

### 8.3. Настройка Prometheus для сбора метрик из Kubernetes

На **Monitoring Server** (10.0.10.205):

```bash
sudo nano /etc/prometheus/prometheus.yml
```

Добавьте следующие job'ы:

```yaml
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
      api_server: http://10.0.10.202:6443
      tls_config:
        insecure_skip_verify: true
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true

  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
    - role: service
      api_server: http://10.0.10.202:6443
      tls_config:
        insecure_skip_verify: true

  - job_name: 'netflix-app'
    static_configs:
      - targets: 
        - '10.0.10.202:30007'
        - '10.0.10.203:30007'
        - '10.0.10.204:30007'
```

```bash
# Перезагрузить конфигурацию
curl -X POST http://localhost:9090/-/reload

# Или перезапустить сервис
sudo systemctl restart prometheus
```

---

## PHASE 9: Email уведомления в Jenkins

### 9.1. Настройка Email в Jenkins

**Manage Jenkins → Configure System → Extended E-mail Notification**

Если у вас есть SMTP сервер (например, Gmail):

- SMTP server: `smtp.gmail.com`
- SMTP Port: `587`
- Use SSL: ✓
- Credentials: Добавьте email и app password
- Default user E-mail suffix: `@gmail.com`

**Email Notification (классический)**
- SMTP server: `smtp.gmail.com`
- Use SMTP Authentication: ✓
- User Name: `your-email@gmail.com`
- Password: `<app-password>`
- Use SSL: ✓
- SMTP Port: `465`

### 9.2. Обновление Pipeline с уведомлениями

Добавьте в Pipeline:

```groovy
    post {
        always {
            archiveArtifacts artifacts: '*.txt', allowEmptyArchive: true
        }
        success {
            emailext(
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                    <p>УСПЕШНО: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                    <p>Проверьте консоль: <a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>
                """,
                to: 'your-email@example.com',
                mimeType: 'text/html'
            )
        }
        failure {
            emailext(
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """
                    <p>ОШИБКА: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                    <p>Проверьте консоль: <a href="${env.BUILD_URL}">${env.JOB_NAME} [${env.BUILD_NUMBER}]</a></p>
                """,
                to: 'your-email@example.com',
                mimeType: 'text/html'
            )
        }
    }
```

---

## PHASE 10: Финальная проверка и тестирование

### 10.1. Проверочный чек-лист

**Jenkins (10.0.10.201:8080):**
- [ ] Jenkins доступен
- [ ] Pipeline успешно выполняется
- [ ] SonarQube анализ работает
- [ ] Docker образы пушатся в DockerHub
- [ ] Trivy сканирование работает

**SonarQube (10.0.10.201:9000):**
- [ ] SonarQube доступен
- [ ] Проекты отображаются
- [ ] Quality Gates настроены

**Kubernetes (10.0.10.202):**
- [ ] Все ноды в статусе Ready
- [ ] Все системные поды запущены
- [ ] Netflix приложение развернуто в namespace netflix
- [ ] Service доступен на NodePort 30007

**ArgoCD (10.0.10.202:NodePort):**
- [ ] ArgoCD UI доступен
- [ ] Netflix приложение синхронизировано
- [ ] Auto-sync работает

**Monitoring (10.0.10.205):**
- [ ] Prometheus доступен на :9090
- [ ] Grafana доступен на :3000
- [ ] Все targets в Prometheus "UP"
- [ ] Dashboards отображают метрики

**Netflix App:**
- [ ] http://10.0.10.202:30007 - доступен
- [ ] http://10.0.10.203:30007 - доступен
- [ ] http://10.0.10.204:30007 - доступен
- [ ] Приложение корректно отображает фильмы

### 10.2. Тестирование CI/CD процесса

1. **Внесите изменение в код:**
```bash
# На вашей Windows машине
cd ~/DevSecOps-Project
# Измените что-то в src/App.jsx
git add .
git commit -m "Test CI/CD"
git push
```

2. **Запустите Pipeline в Jenkins:**
   - Откройте Jenkins
   - Нажмите "Build Now"
   - Следите за выполнением

3. **Проверьте ArgoCD:**
   - Откройте ArgoCD UI
   - Убедитесь что новая версия задеплоилась
   - Проверьте что приложение обновилось

### 10.3. Команды для диагностики

**Kubernetes:**
```bash
# Статус кластера
kubectl cluster-info
kubectl get nodes
kubectl get pods -A

# Netflix приложение
kubectl get all -n netflix
kubectl describe deployment netflix-app -n netflix
kubectl logs -n netflix -l app=netflix

# ArgoCD
kubectl get all -n argocd
argocd app list
argocd app get netflix
```

**Docker:**
```bash
# На Jenkins сервере
docker ps
docker images
docker logs netflix
```

**Prometheus:**
```bash
# Проверить targets
curl http://10.0.10.205:9090/api/v1/targets | jq

# Проверить метрики
curl http://10.0.10.205:9090/api/v1/query?query=up
```

---

## PHASE 11: Доступ извне (опционально)

Если хотите получить доступ к сервисам извне вашей домашней сети:

### 11.1. Вариант 1: Port Forwarding на роутере

На роутере TP-LINK настройте проброс портов:

| Внешний порт | Внутренний IP | Внутренний порт | Описание |
|---|---|---|---|
| 8080 | 10.0.10.201 | 8080 | Jenkins |
| 9000 | 10.0.10.201 | 9000 | SonarQube |
| 3000 | 10.0.10.205 | 3000 | Grafana |
| 9090 | 10.0.10.205 | 9090 | Prometheus |
| 30007 | 10.0.10.202 | 30007 | Netflix App |

### 11.2. Вариант 2: Использование Cloudflare Tunnel (бесплатно)

На **Jenkins Server**:

```bash
# Скачать cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Авторизоваться
cloudflared tunnel login

# Создать туннель
cloudflared tunnel create netflix-devsecops

# Настроить конфиг
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

Содержимое `config.yml`:
```yaml
tunnel: <tunnel-id>
credentials-file: /home/ubuntu/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: jenkins.yourdomain.com
    service: http://10.0.10.201:8080
  - hostname: sonar.yourdomain.com
    service: http://10.0.10.201:9000
  - hostname: grafana.yourdomain.com
    service: http://10.0.10.205:3000
  - hostname: netflix.yourdomain.com
    service: http://10.0.10.202:30007
  - service: http_status:404
```

```bash
# Запустить туннель как сервис
sudo cloudflared service install
sudo systemctl start cloudflared
sudo systemctl enable cloudflared
```

---

## PHASE 12: Cleanup и управление ресурсами

### 12.1. Остановка сервисов

```bash
# Остановить Jenkins
sudo systemctl stop jenkins

# Остановить Docker контейнеры
docker stop $(docker ps -aq)

# Остановить Kubernetes поды
kubectl scale deployment --all --replicas=0 -n netflix

# Остановить Prometheus/Grafana
sudo systemctl stop prometheus
sudo systemctl stop grafana-server
```

### 12.2. Полное удаление VM через Terraform

```bash
cd ~/devsecops-proxmox/terraform
terraform destroy -auto-approve
```

### 12.3. Бэкап важных данных

```bash
# Бэкап Jenkins
sudo tar -czf jenkins-backup.tar.gz /var/lib/jenkins

# Бэкап Kubernetes манифестов
kubectl get all -n netflix -o yaml > netflix-backup.yaml

# Бэкап Prometheus данных
sudo tar -czf prometheus-backup.tar.gz /data

# Бэкап Grafana
sudo tar -czf grafana-backup.tar.gz /var/lib/grafana
```

---

## Приложения

### A. Полезные ссылки

- **Документация Kubernetes:** https://kubernetes.io/docs/
- **ArgoCD Docs:** https://argo-cd.readthedocs.io/
- **Jenkins Docs:** https://www.jenkins.io/doc/
- **Prometheus Docs:** https://prometheus.io/docs/
- **Grafana Dashboards:** https://grafana.com/grafana/dashboards/

### B. Troubleshooting

**Проблема: Ноды Kubernetes не Ready**
```bash
kubectl describe node <node-name>
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

**Проблема: Поды в статусе Pending**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl get events -n <namespace>
```

**Проблема: Jenkins не может подключиться к Docker**
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

**Проблема: ArgoCD не синхронизирует приложение**
```bash
argocd app get netflix
argocd app sync netflix --force
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

### C. Расширенные возможности

**Автоматическое масштабирование:**
```bash
kubectl autoscale deployment netflix-app -n netflix --min=2 --max=10 --cpu-percent=80
```

**Ingress Controller:**
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

**Cert-Manager для SSL:**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

---

## Заключение

Вы создали полноценный DevSecOps пайплайн с:
- ✅ Continuous Integration (Jenkins)
- ✅ Security Scanning (SonarQube, Trivy, OWASP)
- ✅ Container Registry (DockerHub)
- ✅ Kubernetes Orchestration
- ✅ GitOps (ArgoCD)
- ✅ Monitoring (Prometheus + Grafana)
- ✅ Infrastructure as Code (Terraform)

Этот проект отлично демонстрирует современные практики DevSecOps и будет ценным дополнением к вашему резюме!

**Следующие шаги:**
1. Документируйте проект в вашем GitHub
2. Создайте README с архитектурой и скриншотами
3. Снимите демо-видео работы пайплайна
4. Добавьте проект в портфолио/резюме

---

## Дополнительные материалы

### D. Пошаговый план выполнения проекта

**День 1: Подготовка инфраструктуры (2-3 часа)**
1. Создание Cloud-Init шаблона в Proxmox
2. Генерация SSH ключей
3. Настройка Terraform конфигурации
4. Развертывание VM через Terraform
5. Проверка доступности всех VM

**День 2: Jenkins и Security Tools (3-4 часа)**
1. Установка Docker на Jenkins Server
2. Установка и настройка Jenkins
3. Установка SonarQube через Docker
4. Установка Trivy
5. Клонирование репозитория проекта
6. Получение TMDB API ключа
7. Тестовый запуск приложения

**День 3: CI/CD Pipeline (3-4 часа)**
1. Установка плагинов Jenkins
2. Настройка Global Tool Configuration
3. Настройка учетных данных
4. Интеграция с SonarQube
5. Интеграция с DockerHub
6. Создание и тестирование Pipeline
7. Проверка работы всех stages

**День 4: Monitoring Stack (2-3 часа)**
1. Установка Prometheus
2. Установка Node Exporter на всех серверах
3. Установка Grafana
4. Настройка Data Sources
5. Импорт дашбордов
6. Проверка метрик со всех узлов

**День 5: Kubernetes Cluster (4-5 часов)**
1. Подготовка всех нод (containerd, kubeadm)
2. Инициализация Master ноды
3. Установка сетевого плагина
4. Присоединение Worker нод
5. Установка Helm
6. Установка Metrics Server
7. Проверка работоспособности кластера

**День 6: ArgoCD и GitOps (2-3 часа)**
1. Установка ArgoCD в кластер
2. Создание Kubernetes манифестов
3. Создание Git репозитория для манифестов
4. Настройка ArgoCD Application
5. Тестирование автоматической синхронизации

**День 7: Интеграция и тестирование (3-4 часа)**
1. Обновление Jenkins Pipeline для K8s
2. Настройка SSH доступа к K8s Master
3. Настройка мониторинга Kubernetes
4. Настройка email уведомлений
5. Полное end-to-end тестирование
6. Документирование проекта

**Итого:** 19-26 часов чистого времени работы

---

### E. Скрипты автоматизации

#### Скрипт автоматической установки Node Exporter

Создайте файл `install-node-exporter.sh`:

```bash
#!/bin/bash

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

echo -e "${GREEN}=== Установка Node Exporter ===${NC}"

# Создать пользователя
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Скачать Node Exporter
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz

# Установить
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*

# Создать systemd сервис
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
EOF

# Запустить
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

# Проверить статус
if systemctl is-active --quiet node_exporter; then
    echo -e "${GREEN}Node Exporter успешно установлен и запущен!${NC}"
    sudo systemctl status node_exporter --no-pager
else
    echo -e "${RED}Ошибка при установке Node Exporter${NC}"
    exit 1
fi
```

Использование:
```bash
chmod +x install-node-exporter.sh
./install-node-exporter.sh
```

#### Скрипт проверки состояния всей инфраструктуры

Создайте файл `check-infrastructure.sh`:

```bash
#!/bin/bash

GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Список серверов
JENKINS_SERVER="10.0.10.201"
K8S_MASTER="10.0.10.202"
K8S_WORKER1="10.0.10.203"
K8S_WORKER2="10.0.10.204"
MONITORING="10.0.10.205"

echo -e "${YELLOW}=== Проверка инфраструктуры DevSecOps ===${NC}\n"

# Функция проверки доступности хоста
check_host() {
    local host=$1
    local name=$2
    if ping -c 1 -W 2 $host &> /dev/null; then
        echo -e "${GREEN}✓${NC} $name ($host) - доступен"
        return 0
    else
        echo -e "${RED}✗${NC} $name ($host) - недоступен"
        return 1
    fi
}

# Функция проверки HTTP сервиса
check_http() {
    local url=$1
    local name=$2
    if curl -s -o /dev/null -w "%{http_code}" $url | grep -q "200\|302"; then
        echo -e "${GREEN}✓${NC} $name - работает"
        return 0
    else
        echo -e "${RED}✗${NC} $name - не отвечает"
        return 1
    fi
}

# Проверка хостов
echo -e "${YELLOW}Проверка доступности серверов:${NC}"
check_host $JENKINS_SERVER "Jenkins Server"
check_host $K8S_MASTER "K8s Master"
check_host $K8S_WORKER1 "K8s Worker 1"
check_host $K8S_WORKER2 "K8s Worker 2"
check_host $MONITORING "Monitoring Server"

echo ""

# Проверка сервисов
echo -e "${YELLOW}Проверка веб-сервисов:${NC}"
check_http "http://$JENKINS_SERVER:8080" "Jenkins UI"
check_http "http://$JENKINS_SERVER:9000" "SonarQube UI"
check_http "http://$MONITORING:9090" "Prometheus UI"
check_http "http://$MONITORING:3000" "Grafana UI"
check_http "http://$K8S_MASTER:30007" "Netflix App (Master)"
check_http "http://$K8S_WORKER1:30007" "Netflix App (Worker 1)"
check_http "http://$K8S_WORKER2:30007" "Netflix App (Worker 2)"

echo ""

# Проверка Kubernetes (если есть доступ)
if command -v kubectl &> /dev/null; then
    echo -e "${YELLOW}Проверка Kubernetes:${NC}"
    
    nodes_ready=$(kubectl get nodes --no-headers 2>/dev/null | grep -c "Ready")
    nodes_total=$(kubectl get nodes --no-headers 2>/dev/null | wc -l)
    
    if [ $nodes_ready -eq $nodes_total ] && [ $nodes_total -gt 0 ]; then
        echo -e "${GREEN}✓${NC} Kubernetes Nodes: $nodes_ready/$nodes_total Ready"
    else
        echo -e "${RED}✗${NC} Kubernetes Nodes: $nodes_ready/$nodes_total Ready"
    fi
    
    pods_running=$(kubectl get pods -n netflix --no-headers 2>/dev/null | grep -c "Running")
    pods_total=$(kubectl get pods -n netflix --no-headers 2>/dev/null | wc -l)
    
    if [ $pods_running -eq $pods_total ] && [ $pods_total -gt 0 ]; then
        echo -e "${GREEN}✓${NC} Netflix Pods: $pods_running/$pods_total Running"
    else
        echo -e "${RED}✗${NC} Netflix Pods: $pods_running/$pods_total Running"
    fi
    
    argocd_healthy=$(kubectl get pods -n argocd --no-headers 2>/dev/null | grep -c "Running")
    argocd_total=$(kubectl get pods -n argocd --no-headers 2>/dev/null | wc -l)
    
    if [ $argocd_healthy -eq $argocd_total ] && [ $argocd_total -gt 0 ]; then
        echo -e "${GREEN}✓${NC} ArgoCD Pods: $argocd_healthy/$argocd_total Running"
    else
        echo -e "${RED}✗${NC} ArgoCD Pods: $argocd_healthy/$argocd_total Running"
    fi
fi

echo ""
echo -e "${YELLOW}=== Проверка завершена ===${NC}"
```

Использование:
```bash
chmod +x check-infrastructure.sh
./check-infrastructure.sh
```

---

### F. Ansible Playbooks для автоматизации

Если хотите автоматизировать установку на все серверы:

#### Создание Ansible inventory

Файл `inventory/hosts.ini`:
```ini
[jenkins]
jenkins ansible_host=10.0.10.201 ansible_user=ubuntu

[k8s_master]
k8s-master ansible_host=10.0.10.202 ansible_user=ubuntu

[k8s_workers]
k8s-worker-1 ansible_host=10.0.10.203 ansible_user=ubuntu
k8s-worker-2 ansible_host=10.0.10.204 ansible_user=ubuntu

[monitoring]
monitoring ansible_host=10.0.10.205 ansible_user=ubuntu

[k8s:children]
k8s_master
k8s_workers

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_private_key_file=~/.ssh/devsecops_rsa
```

#### Playbook для установки Node Exporter на все серверы

Файл `playbooks/install-node-exporter.yml`:
```yaml
---
- name: Install Node Exporter on all servers
  hosts: all
  become: yes
  tasks:
    - name: Create node_exporter user
      user:
        name: node_exporter
        system: yes
        shell: /bin/false
        create_home: no

    - name: Download Node Exporter
      get_url:
        url: https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
        dest: /tmp/node_exporter.tar.gz

    - name: Extract Node Exporter
      unarchive:
        src: /tmp/node_exporter.tar.gz
        dest: /tmp/
        remote_src: yes

    - name: Copy Node Exporter binary
      copy:
        src: /tmp/node_exporter-1.6.1.linux-amd64/node_exporter
        dest: /usr/local/bin/node_exporter
        mode: '0755'
        remote_src: yes

    - name: Create systemd service file
      copy:
        dest: /etc/systemd/system/node_exporter.service
        content: |
          [Unit]
          Description=Node Exporter
          Wants=network-online.target
          After=network-online.target

          [Service]
          User=node_exporter
          Group=node_exporter
          Type=simple
          ExecStart=/usr/local/bin/node_exporter

          [Install]
          WantedBy=multi-user.target

    - name: Start and enable Node Exporter
      systemd:
        name: node_exporter
        state: started
        enabled: yes
        daemon_reload: yes

    - name: Cleanup
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /tmp/node_exporter.tar.gz
        - /tmp/node_exporter-1.6.1.linux-amd64
```

Запуск:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/install-node-exporter.yml
```

---

### G. Диаграммы архитектуры

#### Схема сетевой топологии

```
┌─────────────────────────────────────────────────────────────┐
│                    Internet (Grey IP)                        │
└────────────────────────┬────────────────────────────────────┘
                         │
┌────────────────────────┴────────────────────────────────────┐
│              TP-LINK Router (10.0.10.1)                      │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │     Network: 10.0.10.0/24       │
        └─────────────────────────────────┘
                         │
        ┌────────────────┼───────────────────────────┐
        │                │                           │
┌───────┴────────┐  ┌───┴──────┐          ┌────────┴─────────┐
│   Proxmox      │  │ Windows  │          │  Other Devices   │
│  10.0.10.200   │  │  DHCP    │          │      DHCP        │
└────────────────┘  └──────────┘          └──────────────────┘
        │
        │ (Virtual Machines)
        │
        ├─── Jenkins Server (10.0.10.201)
        │     ├─ Jenkins :8080
        │     ├─ SonarQube :9000
        │     ├─ Docker
        │     └─ Node Exporter :9100
        │
        ├─── K8s Master (10.0.10.202)
        │     ├─ API Server :6443
        │     ├─ ArgoCD :NodePort
        │     ├─ Netflix App :30007
        │     └─ Node Exporter :9100
        │
        ├─── K8s Worker 1 (10.0.10.203)
        │     ├─ Netflix Pods
        │     └─ Node Exporter :9100
        │
        ├─── K8s Worker 2 (10.0.10.204)
        │     ├─ Netflix Pods
        │     └─ Node Exporter :9100
        │
        └─── Monitoring (10.0.10.205)
              ├─ Prometheus :9090
              ├─ Grafana :3000
              └─ Node Exporter :9100
```

#### Схема CI/CD Pipeline

```
┌──────────────┐
│ Developer    │
│ Git Push     │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────┐
│              Jenkins Pipeline                     │
│                                                   │
│  ┌─────────────────────────────────────────┐    │
│  │ 1. Checkout Code from GitHub            │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 2. SonarQube Analysis                   │    │
│  │    - Code Quality Check                 │    │
│  │    - Security Vulnerabilities           │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 3. Install NPM Dependencies             │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 4. OWASP Dependency Check               │    │
│  │    - Check vulnerable dependencies      │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 5. Trivy Filesystem Scan                │    │
│  │    - Scan source code for issues        │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 6. Docker Build & Push                  │    │
│  │    - Build image with TMDB API key      │    │
│  │    - Tag: username/netflix:BUILD_NUMBER │    │
│  │    - Push to DockerHub                  │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 7. Trivy Image Scan                     │    │
│  │    - Scan Docker image for CVEs         │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
│            ▼                                      │
│  ┌─────────────────────────────────────────┐    │
│  │ 8. Update K8s Manifests in Git          │    │
│  │    - Update image tag                   │    │
│  │    - Commit & Push to GitOps repo       │    │
│  └─────────┬───────────────────────────────┘    │
│            │                                      │
└────────────┼──────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────┐
│              ArgoCD (GitOps)                   │
│                                                 │
│  ┌─────────────────────────────────────────┐  │
│  │ 1. Detect changes in Git repository     │  │
│  └─────────┬───────────────────────────────┘  │
│            │                                    │
│            ▼                                    │
│  ┌─────────────────────────────────────────┐  │
│  │ 2. Compare desired state vs actual      │  │
│  └─────────┬───────────────────────────────┘  │
│            │                                    │
│            ▼                                    │
│  ┌─────────────────────────────────────────┐  │
│  │ 3. Auto-sync (if enabled)                │  │
│  │    - Apply new manifests                 │  │
│  │    - Rolling update deployment           │  │
│  └─────────┬───────────────────────────────┘  │
│            │                                    │
└────────────┼────────────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────────┐
│         Kubernetes Cluster                     │
│                                                 │
│  ┌──────────────────────────────────────────┐ │
│  │ Namespace: netflix                       │ │
│  │                                          │ │
│  │  ┌────────────┐  ┌────────────┐        │ │
│  │  │  Pod 1     │  │  Pod 2     │        │ │
│  │  │  netflix   │  │  netflix   │        │ │
│  │  └────────────┘  └────────────┘        │ │
│  │         │               │               │ │
│  │         └───────┬───────┘               │ │
│  │                 │                       │ │
│  │         ┌───────▼──────┐                │ │
│  │         │   Service    │                │ │
│  │         │ NodePort:    │                │ │
│  │         │   30007      │                │ │
│  │         └──────────────┘                │ │
│  └──────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
             │
             ▼
     ┌───────────────┐
     │  End Users    │
     │  :30007       │
     └───────────────┘
```

---

### H. Конфигурация для GitHub README

Создайте файл `README.md` для вашего GitHub репозитория:

```markdown
# Netflix Clone - DevSecOps Pipeline на Proxmox

![DevSecOps](./images/architecture.png)

## 📋 Описание проекта

Полноценный DevSecOps пайплайн для развертывания Netflix Clone с использованием современных инструментов и практик на домашней инфраструктуре Proxmox.

## 🏗️ Архитектура

- **Jenkins** - CI/CD автоматизация
- **SonarQube** - Анализ качества кода
- **Trivy** - Сканирование безопасности
- **Docker** - Контейнеризация
- **Kubernetes** - Оркестрация контейнеров
- **ArgoCD** - GitOps CD
- **Prometheus** - Мониторинг
- **Grafana** - Визуализация метрик
- **Terraform** - Infrastructure as Code

## 🚀 Компоненты инфраструктуры

| Компонент | IP | Specs | Роль |
|-----------|-----|-------|------|
| Jenkins Server | 10.0.10.201 | 4 CPU, 8GB RAM | CI/CD, SonarQube |
| K8s Master | 10.0.10.202 | 4 CPU, 8GB RAM | Control Plane |
| K8s Worker 1 | 10.0.10.203 | 4 CPU, 8GB RAM | Workload |
| K8s Worker 2 | 10.0.10.204 | 4 CPU, 8GB RAM | Workload |
| Monitoring | 10.0.10.205 | 2 CPU, 4GB RAM | Prometheus, Grafana |

## 🔒 Security Features

- ✅ SonarQube code quality analysis
- ✅ OWASP Dependency Check
- ✅ Trivy filesystem scanning
- ✅ Trivy container image scanning
- ✅ Quality Gates
- ✅ Automated security reports

## 📊 Мониторинг

- Real-time metrics с Prometheus
- Grafana dashboards для визуализации
- Node Exporter на всех серверах
- Kubernetes cluster monitoring
- Application performance metrics

## 🔄 CI/CD Pipeline

1. **Code Checkout** - Получение кода из Git
2. **Static Analysis** - SonarQube сканирование
3. **Quality Gate** - Проверка качества кода
4. **Dependencies** - Установка зависимостей
5. **Security Scan** - OWASP + Trivy
6. **Build** - Docker image сборка
7. **Push** - Загрузка в registry
8. **Deploy** - Автоматический деплой через ArgoCD

## 🛠️ Установка

Подробная инструкция в [INSTALLATION.md](./INSTALLATION.md)

### Быстрый старт

```bash
# 1. Клонировать репозиторий
git clone https://github.com/yourusername/netflix-devsecops.git

# 2. Развернуть инфраструктуру через Terraform
cd terraform
terraform init
terraform apply

# 3. Следовать пошаговой инструкции в INSTALLATION.md
```

## 📸 Screenshots

### Pipeline Execution
![Pipeline](./images/pipeline.png)

### SonarQube Analysis
![SonarQube](./images/sonarqube.png)

### ArgoCD Dashboard
![ArgoCD](./images/argocd.png)

### Grafana Monitoring
![Grafana](./images/grafana.png)

### Netflix Application
![Netflix](./images/netflix-app.png)

## 📝 Использованные технологии

- **Infrastructure**: Proxmox, Terraform
- **CI/CD**: Jenkins, ArgoCD
- **Security**: SonarQube, Trivy, OWASP Dependency-Check
- **Containers**: Docker, Kubernetes
- **Monitoring**: Prometheus, Grafana, Node Exporter
- **Frontend**: React, Vite
- **API**: TMDB (The Movie Database)

## 🎓 Чему я научился

- Построение end-to-end DevSecOps пайплайна
- Infrastructure as Code с Terraform
- Kubernetes кластер с нуля
- GitOps практики с ArgoCD
- Security scanning интеграция
- Monitoring и observability
- CI/CD автоматизация

## 📞 Контакты

- LinkedIn: [Your Profile]
- Email: your.email@example.com
- Portfolio: [Your Website]

## 📄 Лицензия

MIT License

---

⭐ Star this repo if you find it helpful!
```

---

## Итоговая сводка

### ✅ Что вы получите после выполнения проекта:

1. **Практический опыт** с современным стеком DevSecOps инструментов
2. **Portfolio проект** для демонстрации работодателям
3. **Реальная инфраструктура** на собственном оборудовании
4. **Документация** всех процессов и конфигураций
5. **Навыки автоматизации** CI/CD пайплайнов
6. **Знания безопасности** при разработке и деплое
7. **Опыт с Kubernetes** и container orchestration
8. **Понимание GitOps** методологии

### 💡 Дополнительные идеи для расширения:

1. **Helm Charts** - упаковать приложение в Helm chart
2. **Ingress Controller** - настроить nginx-ingress для доменных имен
3. **SSL/TLS** - добавить cert-manager для автоматических сертификатов
4. **Horizontal Pod Autoscaling** - автомасштабирование на основе нагрузки
5. **Backup & Restore** - Velero для бэкапов Kubernetes
6. **Service Mesh** - Istio или Linkerd для advanced networking
7. **Logging Stack** - ELK или Loki для централизованных логов
8. **Chaos Engineering** - Chaos Mesh для тестирования отказоустойчивости

Удачи в реализации проекта! 🚀
