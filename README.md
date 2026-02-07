# Magma Core Installation Guide: Orc8r + NMS + AGW
### (Local Deployment)

![Ubuntu](https://img.shields.io/badge/OS-Ubuntu_20.04-orange) ![Docker](https://img.shields.io/badge/Container-Docker-blue) ![Status](https://img.shields.io/badge/Status-Tested-success)

This guide documents the **exact process** to successfully deploy Magma Core, including the Orchestrator (Orc8r), Network Management System (NMS), and Access Gateway (AGW). 

---
## Index

1. [Introduction](#magma-core-installation-guide-orc8r--nms--agw)
2. [Prerequisites](#-prerequisites)
   - [Hardware Requirements](#hardware-requirements)
   - [Software Requirements](#software-requirements-host)
3. [Step-by-Step Installation](#-step-by-step-installation)
   - [Step 1: Host OS Preparation](#step-1-host-os-preparation)
   - [Step 2: Install Host Tools](#step-2-install-host-tools)
   - [Step 3: Create Virtual Machines](#step-3-create-virtual-machines)
   - [Step 4: Network Configuration](#step-4-network-configuration-critical)
   - [Step 5: Clone Repository](#step-5-clone-repositoryon-the-host-machine)
4. [Orc8r & NMS Setup (VM 1)](#-orc8r--nms-setup-vm-1)
   - [Step 6: System Update & Prerequisites](#step-6-system-update--prerequisites-inside-the-orc8r-vm)
   - [Step 7: Fix Fluentd Ruby Gem Compatibility](#step-7-fix-fluentd-ruby-gem-compatibility-critical)
   - [Step 8: Build and Run Orc8r](#step-8-build-and-run-orc8r-imagesbashcd-magmaorc8rclouddocker)
   - [Step 9: Build & Run NMS](#step-9-build--run-nms)
   - [Step 10: Set Admin Password](#step-10-set-admin-password-for-the-first-time-next-time)
   - [Step 11: Access Portal](#step-11-access-portal)
5. [Copy Root Certificate](#step-12-copy-root-certificate-from-orc8r-to-agw)
6. [Install AGW on VM2](#install-agw-on-vm2)
   - [Step 13: Download](#step-13-download)
   - [Step 15: Configure Hostname Resolution](#step-15-configure-hostname-resolutionedit)
   - [Step 16: Configure Control Proxy](#step-16-configure-control-proxycreate)
   - [Step 17: Start AGW Services](#step-17-start-agw-services)
7. [Integration & Verification](#-integration--verification)
   - [Step 18: Register AGW with Orc8r](#step-18register-agw-with-orc8rget-hardware-id-on-agw-vm)
   - [Step 19: Final Health Check](#step-19-final-health-checkrun-the-check-in-cli-on-the-agw-vm)
8. [Monitoring (Optional)](#optional-step-20--start-prometheus-and-grafana-for-metrics)
9. [Links](#links)
10. [Resources](#resources)


## 📋 Prerequisites

### Hardware Requirements
* **Host OS:** Windows (Dual-booted with Ubuntu 20.04) or native Linux machine.
* **Disk Space:** Minimum 150 GB allocated to Ubuntu partition.
* **RAM:** Sufficient to run Host OS + 3 VMs (approx. 16GB total RAM recommended).

### Software Requirements (Host)
* VirtualBox 6.1
* Docker & Docker Compose
* **Git**

> ❌ **Note:** Vagrant is **NOT** required for this specific setup.

---

## 🛠️ Step-by-Step Installation


### Step 1: Host OS Preparation

1.  Dual boot your system to **Ubuntu 20.04 LTS**.
2.  Ensure system updates are enabled.
    > ⚠️ **Warning:** Ubuntu 20.04 is strongly recommended for compatibility. Newer versions may cause dependency issues.


### Step 2: Install Host Tools

Run the following commands on your host Ubuntu system to verify installations:

```bash
docker --version
docker compose version
virtualbox --help
```


### Step 3: Create Virtual Machines

Virtual Machines in VirtualBox with the following specifications:
1. Orc8r + NMS 4 GB (Min) 40GB  Ubuntu 20.04
2. VM 3AGW 4 GB 30 GB Ubuntu 20.04

   
### Step 4: Network Configuration (Critical)

For EACH VM,/
configure two network adapters. This is mandatory for Orc8r–AGW communication.
```
Adapter 1 (Primary): Bridged Adapter OR Static Wi-Fi Configure via GUI
Adapter 2 (Secondary): NAT this will be used to update and install requirmenet's.
```

### Step 5: Clone RepositoryOn the Host Machine:

Fork the Magma repository to your GitHub account.Clone and configure upstream:Bashgit clone [https://github.com/YOUR_USERNAME/magma.git](https://github.com/YOUR_USERNAME/magma.git)
```bash
cd magma
git remote add upstream https://github.com/magma/magma.git
git remote -v
```


## ☁️ Orc8r & NMS Setup (VM 1)

### Step 6: System Update & Prerequisites (Inside the Orc8r VM)

```Bash
sudo apt update
sudo apt upgrade -y
```

## Install Docker, Docker Compose, Python3, and Git

### Step 7: Fix Fluentd Ruby Gem Compatibility (CRITICAL): 

Without this fix, the Orc8r build WILL FAIL.Navigate to the docker directory:
```Bash
cd magma/orc8r/cloud/docker/fluentd
```
Open the Dockerfile and replace the content with the following: 
Updated Dockerfile:Dockerfile 
```
FROM fluent/fluentd:v1.14.6-debian-1.0
USER root
  RUN gem install rubygems-update -v 3.2.33 --no-document && \
  update_rubygems && \
  gem install multi_json -v 1.15.0 --no-document && \
  gem install elasticsearch -v 7.13.0 --no-document && \
  gem install fluent-plugin-elasticsearch -v 5.2.1 --no-document && \
  gem install fluent-plugin-multi-format-parser -v 1.0.0 --no-document
USER fluent
```


### Step 8: Build and run Orc8r ImagesBashcd magma/orc8r/cloud/docker
```
./build.py --all
./run.py --metrics
```
Wait for the build to complete successfully.



### Step 9: Build & Run NMS 

``` bash
cd ../../../nms
COMPOSE_PROJECT_NAME=magmalte docker compose --compatibility build magmalte
docker compose --compatibility up -d
```


### Step 10: Set Admin Password (for the first time next time)

```Bash
docker compose exec magmalte \
yarn setAdminPassword host admin@magma.test password1234
```


### Step 11: Access Portal

Host Portal: [http://host.localhost:8081/](http://host.localhost:8081/)

hostLogin: admin@magma.test / password1234

Action: Create Organization (magma test1) 
with a super user and give acess to all internet 

        now create a super user in it with
        
        id :super@magma.test password: password1234

sudo vim /var/opt/magma/certs/rootCA.pem


### Step 12: copy root Certificate from Orc8r to AGW

**Note:**  I am using SSH 
```
# Locate Root CA on Orc8r VM
ls magma/.cache/test_certs/rootCA.pem

# Create certificate directory on AGW VM
ssh ubuntu@<AGW_IP>
sudo mkdir -p /var/opt/magma/certs
sudo chmod 755 /var/opt/magma/certs
exit

# Copy Root CA certificate from Orc8r to AGW
scp magma/.cache/test_certs/rootCA.pem ubuntu@<AGW_IP>:/tmp/rootCA.pem

# Move certificate to final location on AGW VM
ssh ubuntu@<AGW_IP>
sudo mv /tmp/rootCA.pem /var/opt/magma/certs/rootCA.pem
sudo chmod 644 /var/opt/magma/certs/rootCA.pem

# Verify
ls -l /var/opt/magma/certs/rootCA.pem
```
**Note:** certificates must be in /var/opt/magma/certs/rootCA.pem


## Install AGW on VM2

### Step 13: Download 

```
wget https://github.com/magma/magma/raw/v1.8/lte/gateway/deploy/agw_install_docker.sh
agw_install_docker.sh # require docker Compose
 ```
Reboot the VM when prompted.



### Step 15: Configure Hostname ResolutionEdit

``` /etc/hosts```
on the AGW VM to point to your Orc8r IP: 
```
sudo nano /etc/hosts
```
Add the following lines:
```
10.10.10.10 controller.magma.test  # this should be your orc8r ip 
10.10.10.10  bootstrapper-controller.magma.test
```
⚠️ Certificates strictly require .
magma.test domains.


### Step 16: Configure Control ProxyCreate 

the config file:
``` Bash
sudo nano /var/opt/magma/configs/control_proxy.yml
```
Paste the following configuration:
```
YAMLcloud_address: controller.magma.test
cloud_port: 7443
bootstrap_address: bootstrapper-controller.magma.test
bootstrap_port: 7444
fluentd_address: controller.magma.test
fluentd_port: 24224
rootca_cert: /var/opt/magma/certs/rootCA.pem

```


### Step 17: Start AGW Services

``` Bash
cd 
/var/opt/magma/docker
sudo docker compose up -d
```


### 🔗 Integration & Verification 

### Step 18:Register AGW with Orc8rGet Hardware ID (On AGW VM):

```Bash
docker exec magmad show_gateway_info.py
```
Copy the Hardware ID and Challenge Key.
Register (On NMS UI):
Go to Equipment -> Add Gateway.Paste the ID and Key.
Restart AGW (On AGW VM):
``` sudo docker compose up -d --force-recreate```


### Step 19: Final Health CheckRun the check-in CLI on the AGW VM:

```Bash
sudo docker exec magmad checkin_cli.py
```
✅ Success: If the output indicates a successful check-in, your Magma deployment is fully operational.


### Optional Step 20 : Start Prometheus and Grafana for Metrics

If you want to monitor metrics, start Prometheus and Grafana with:

```
docker-compose -f docker-compose.metrics.yml up -d
```
You might need to add this snippet to the bottom of docker-compose.metrics.yml to connect to the correct network:
```
networks:
  default:
    external:
      name: orc8r_default
```

## Second time starting
VM1 Orc8r + NMS
```
cd orc8r/cloud/docker 
./run.py --metrics

cd ../../../nms 
docker compose --compatibility up -d

Metrics (optional)
docker compose -f docker-compose.metrics.yml up -d
sudo docker exec magmad checkin_cli.py
```

VM2 AcessGateWay
```
cd /var/opt/magma/docker
sudo docker compose up -d
```

## Links 

NMS UI	[http://host.localhost:8081/host](http://host.localhost:8081/host)
```
ID: admin@magma.test
Password: password1234
```

```
Organization: magma test1   
ID: super@magma.test
Password: password1234
```

Orc8r Swagger	[https://localhost:9443/swagger/v1/ui](https://localhost:9443/swagger/v1/ui)
Promethius: [http://localhost:9090](http://localhost:9090)

## Resources 

Official Magma Documentation – Magma basics and introduction
https://magma.github.io/magma/docs/basics/introduction.html

Kidus’ Beginner-Friendly Guide – Git and Docker workflow for contributing to Magma’s NMS
https://medium.com/@pauloskidus48/a-beginner-friendly-git-and-docker-workflow-for-contributing-to-magmas-nms-fba541eb8443

Patrick’s Magma AGW Docker & VM Setup Guide – Step-by-step setup instructions
https://github.com/patricktomlin/magma-agw-docker-vm-guide/tree/main
                    
