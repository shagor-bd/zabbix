# Zabbix Installation, Configuration, and Maintenance Guide

This guide provides comprehensive instructions for installing, configuring, and maintaining Zabbix monitoring system.

## Table of Contents

- [Zabbix Installation, Configuration, and Maintenance Guide](#zabbix-installation-configuration-and-maintenance-guide)
  - [Table of Contents](#table-of-contents)
  - [Prerequisites](#prerequisites)
    - [System Requirements](#system-requirements)
    - [Software Dependencies](#software-dependencies)
  - [Installation Methods](#installation-methods)
    - [Method 1: Docker Installation (Recommended)](#method-1-docker-installation-recommended)
    - [Method 2: Package Installation](#method-2-package-installation)
  - [Database Setup](#database-setup)
    - [MySQL Database Configuration](#mysql-database-configuration)
  - [Zabbix Server Configuration](#zabbix-server-configuration)
    - [Main Configuration File: `/etc/zabbix/zabbix_server.conf`](#main-configuration-file-etczabbixzabbix_serverconf)
    - [Create Required Directories](#create-required-directories)
  - [Zabbix Agent Installation](#zabbix-agent-installation)
    - [Copy RPM file to destination server](#copy-rpm-file-to-destination-server)
    - [📘 Zabbix – Management Interface Routing Guidelines](#-zabbix--management-interface-routing-guidelines)
      - [🔧 Required Configuration by Platform](#-required-configuration-by-platform)
        - [**For OpenStack Platform:**](#for-openstack-platform)
        - [**For VMware or RHEV Platforms:**](#for-vmware-or-rhev-platforms)
      - [✅ Summary](#-summary)
    - [Install `zabbix-agent2` to Destination hosts using Ansible](#install-zabbix-agent2-to-destination-hosts-using-ansible)
    - [Manual Installation](#manual-installation)
  - [Some more configuration for Zabbix](#some-more-configuration-for-zabbix)
    - [PHP Configuration](#php-configuration)
  - [Initial Configuration](#initial-configuration)
    - [Web Interface Setup](#web-interface-setup)
    - [Default Credentials](#default-credentials)
  - [Maintenance Tasks](#maintenance-tasks)
    - [Regular Maintenance Schedule](#regular-maintenance-schedule)
      - [Daily Tasks](#daily-tasks)
      - [Weekly Tasks](#weekly-tasks)
      - [Monthly Tasks](#monthly-tasks)
    - [Database Maintenance](#database-maintenance)
    - [Log Rotation](#log-rotation)
  - [Troubleshooting](#troubleshooting)
    - [Common Issues and Solutions](#common-issues-and-solutions)
      - [Zabbix Server Won't Start](#zabbix-server-wont-start)
      - [Agent Connection Issues](#agent-connection-issues)
      - [Web Interface Problems](#web-interface-problems)
    - [Performance Tuning](#performance-tuning)
      - [Database Optimization](#database-optimization)
      - [Zabbix Server Tuning](#zabbix-server-tuning)
  - [Additional Resources](#additional-resources)

## Prerequisites

### System Requirements

- **Operating System**: RHEL 9
- **CPU**: 4 cores
- **RAM**: 8GB
- **Storage**: As per plan
- **Network**:
  - ens3 | 10.2.64.8 | O&M
  - ens4 | 10.18.248.26 | Service IP
  - VM Hosted in Management Zone.

### Software Dependencies

- **Database**: MySQL 8.0+
- **Web Server**: Apache/2.4.62
- **PHP**: PHP 8.0.30.

## Installation Methods

### Method 1: Docker Installation (Recommended)

```bash
# Clone the official Zabbix Docker repository
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker

# Start Zabbix with Docker Compose
docker-compose up -d
```

### Method 2: Package Installation

```bash
# RHEL/CentOS 8+
sudo dnf install zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-sql-scripts

# Ubuntu/Debian
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts
```

## Database Setup

### MySQL Database Configuration

```sql
-- Create database and user
CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY '*******************';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;

-- Import schema and data
mysql -u zabbix -p zabbix < /usr/share/doc/zabbix-sql-scripts/mysql/create.sql
mysql -u zabbix -p zabbix < /usr/share/doc/zabbix-sql-scripts/mysql/images.sql
mysql -u zabbix -p zabbix < /usr/share/doc/zabbix-sql-scripts/mysql/data.sql
```

## Zabbix Server Configuration

### Main Configuration File: `/etc/zabbix/zabbix_server.conf`

```ini
# Logging Configuration
LogFile=/var/log/zabbix/zabbix_server.log
LogFileSize=0
PidFile=/run/zabbix/zabbix_server.pid
SocketDir=/run/zabbix

# Database Configuration
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=*******************

# Performance Tuning
ValueCacheSize=128M
Timeout=4
LogSlowQueries=3000

# SNMP Configuration
SNMPTrapperFile=/var/log/snmptrap/snmptrap.log

# Security Configuration
StatsAllowedIP=127.0.0.1
EnableGlobalScripts=0

# Network Configuration
NodeAddress=localhost:10051

# Include additional configuration files
Include=/etc/zabbix/zabbix_server.d/*.conf
```

### Create Required Directories

```bash
sudo mkdir -p /var/log/zabbix
sudo mkdir -p /var/log/snmptrap
sudo mkdir -p /run/zabbix
sudo chown -R zabbix:zabbix /var/log/zabbix /var/log/snmptrap /run/zabbix
```

## Zabbix Agent Installation

### Copy RPM file to destination server

We will use `zabbix-agent2` for zabbix agent. [Why Zabbix Agent2?](https://www.zabbix.com/documentation/current/en/manual/concepts/agent2)

Download `zabbix-agent2` for RHEL 7,8,9.

```bash
# RHEL9
wget https://repo.zabbix.com/zabbix/7.2/release/rhel/9/noarch/zabbix-release-latest-7.2.el9.noarch.rpm
# RHEL 8
wget https://repo.zabbix.com/zabbix/7.2/release/rhel/8/noarch/zabbix-release-latest-7.2.el8.noarch.rpm
# RHEL 7
wget https://repo.zabbix.com/zabbix/7.2/release/rhel/7/noarch/zabbix-release-latest-7.2.el7.noarch.rpm
```
<!-- markdownlint-disable MD036 -->
**Copy rpm to destination server**
<!-- markdownlint-enable MD036 -->
```bash
# Navigate to ansible directory
cd ansible/
ansible-playbook -i inventory/inventory.yaml ansible-copy/copy-from-bastion-msg.yaml
```

[Detail Logs](./logs/copy-logs.txt)
<!-- markdownlint-disable MD036 -->
**Confirm files copied into destination server**
<!-- markdownlint-enable MD036-->
```bash
ansible all -i inventory/inventory.yaml  -m shell -a 'ls /tmp/zabbix*'
```

[Detail Logs](./logs/copy-confirmation-logs.txt)

**Note:** We can copy any files with the ansible script

### 📘 Zabbix – Management Interface Routing Guidelines

<!-- markdownlint-disable MD036 MD033 -->
<span style="color:red">**Check ROUTE in every host WHY**</span>
<!-- markdownlint-enable MD036 MD033-->
In both **Bogura DA** and **Savar DA** environments, each host is equipped with two network interfaces:

- **Service Interface** – used for general traffic and configured with the default route.
- **Management Interface** – designated for administrative tasks such as monitoring, PAM, etc.

```bash
ip route add 10.2.64.8/32 via <gateway ip>
```
<!-- markdownlint-disable MD036 -->
**Also we can add route in file if available**

```bash
cd /etc/sysconfig/network-scripts
$ vim route-ens4
10.2.64.30/32 via 10.12.94.97
10.2.64.37/32 via 10.12.94.97
192.168.135.107/32 via 10.12.94.97
192.168.129.204/32 via 10.12.94.97
10.2.64.8/32 via <gateway ip>
```

As per **Grameenphone Network Guidline all monitoring traffic (e.g., Zabbix communication)** must flow through the **Management Interface**. Since the default route is associated with the Service Interface, monitoring traffic **will not reach the Zabbix server** unless a custom route is configured.

Our **Zabbix Server resides in the Management Zone**, so in order for **Zabbix Agent (e.g., `zabbix-agent2`)** on a host to communicate with the server, a specific route to the management gateway must be added.

---

#### 🔧 Required Configuration by Platform

##### **For OpenStack Platform:**

1. **Allow Port 10051** (Zabbix communication port) in the host's Security Group.
2. **Add a static route** in the OpenStack route table to direct traffic for the Zabbix Server IP (or Management Zone) via the **Management Interface Gateway**.

##### **For VMware or RHEV Platforms:**

Static routes must be **manually configured on each host** to ensure management traffic (e.g., to the Zabbix server) is routed via the **Management Interface Gateway**.

---

#### ✅ Summary

Without these custom routes, the monitoring traffic will follow the default route via the Service Interface, causing failure in Zabbix communication. Ensuring the correct routing configuration will enable successful monitoring via the Management Interface as intended.

---

### Install `zabbix-agent2` to Destination hosts using Ansible
<!-- markdownlint-disable MD036 -->
**Check before run the deployment playbook**
<!-- markdownlint-enable MD036 -->
```bash
# Navigate to ansible directory
cd /home/healthcheck/abdulaziz/ansible
ansible-playbook -i inventory/inventory.yaml  play-zabbix/deploy_zabbix.yml --check --diff
```

[Detail logs](./logs/ansible-script-check.txt)

```bash
# Run the deployment playbook
ansible-playbook -i ../inventory/inventory.yaml deploy_zabbix.yml
```

[Command and detail logs](./logs/install-logs.txt)

### Manual Installation

```bash
# Install Zabbix Agent
# Copy zabbix-agent2 into /tmp directory as per RHEL version
cd /tmp
cat /etc/redhat-release
# Output: Red Hat Enterprise Linux release 8.5 (Ootpa)
rpm -ivh zabbix-agent2-7.2.7-release1.el8.x86_64.rpm
# Configure agent
sudo tee /etc/zabbix/zabbix_agent2.conf > /dev/null <<EOF
PidFile=/run/zabbix/zabbix_agent2.pid
LogFile=/var/log/zabbix/zabbix_agent2.log
LogFileSize=0
Server=127.0.0.1,10.2.64.8
Plugins.System.Capacity=3
ListenPort=10050
ServerActive=10.2.64.8
Hostname=a0420ssecdapmcmscas02
HostMetadata=Linux_server
Include=/etc/zabbix/zabbix_agent2.d/*.conf
Timeout=10
EOF

# Start and enable service
sudo systemctl enable zabbix-agent2
sudo systemctl start zabbix-agent2
```

## Some more configuration for Zabbix

-[ ] **Web Interface Setup**

### PHP Configuration

```ini
# /etc/php.ini adjustments
max_execution_time = 300
memory_limit = 128M
post_max_size = 16M
upload_max_filesize = 2M
date.timezone = UTC
```

## Initial Configuration

### Web Interface Setup

1. **Access the Web Interface**: Navigate to [Zabbix Server web UI](https://a0420pmgtzabbix01.grameenphone.com/zabbix)
2. **Check Prerequisites**: Ensure all requirements are met
3. **Configure Database Connection**:
   - Database type: MySQL
   - Host: localhost
   - Port: 3306 (MySQL)
   - Database name: zabbix
   - User: zabbix
   - Password: *******************
4. **Configure Zabbix Server Details**:
   - Host: localhost
   - Port: 10051
   - Name: Zabbix server
5. **Complete Installation**: Review and finish setup

### Default Credentials

- **Username**: Admin
- **Password**: zabbix (Default password change after first login)

## Maintenance Tasks

### Regular Maintenance Schedule

#### Daily Tasks

- Check Zabbix server logs for errors
- Monitor database performance
- Verify agent connectivity

#### Weekly Tasks

- Review and clean old data
- Update Zabbix server and agents
- Backup configuration and database

#### Monthly Tasks

- Performance tuning review
- Security updates
- Capacity planning

### Database Maintenance

```sql
-- Clean old history data (older than 30 days)
DELETE FROM history WHERE clock < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 30 DAY));
DELETE FROM history_uint WHERE clock < UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 30 DAY));

-- Optimize tables
OPTIMIZE TABLE history, history_uint, trends, trends_uint;
```

### Log Rotation

```bash
# /etc/logrotate.d/zabbix
/var/log/zabbix/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 zabbix zabbix
    postrotate
        systemctl reload zabbix-server
    endscript
}
```

## Troubleshooting

### Common Issues and Solutions

#### Zabbix Server Won't Start

```bash
# Check configuration syntax
zabbix_server -t

# Check logs
tail -f /var/log/zabbix/zabbix_server.log

# Verify database connectivity
mysql -u zabbix -p -e "SELECT 1;"
```

#### Agent Connection Issues

```bash
# Test agent connectivity
zabbix_get -s hostname -k agent.ping

# Check agent logs
tail -f /var/log/zabbix/zabbix_agent2.log

# Verify firewall settings
sudo firewall-cmd --list-all
```

#### Web Interface Problems

```bash
# Check PHP configuration
php -m | grep -E "(gd|bcmath|mbstring|xml)"

# Verify web server configuration
apachectl -t
nginx -t

# Check SELinux status
getenforce
```

### Performance Tuning

#### Database Optimization

```sql
-- Increase connection pool
SET GLOBAL max_connections = 200;

-- Optimize query cache
SET GLOBAL query_cache_size = 128M;
```

#### Zabbix Server Tuning

```ini
# /etc/zabbix/zabbix_server.conf
StartPollers=10
StartPingers=5
StartTrappers=5
StartDiscoverers=1
StartPreprocessors=3
StartAlerters=3
```
<!-- markdownlint-enable MD036 -->

## Additional Resources

- [Official Zabbix Documentation](https://www.zabbix.com/documentation/current/en/manual/installation/containers)
- [Zabbix Community Forum](https://www.zabbix.com/forum)
- [Zabbix GitHub Repository](https://github.com/zabbix/zabbix)
- [Zabbix Best Practices Guide](https://www.zabbix.com/documentation/current/en/manual/appendix/install/best_practices)
- [Zabbix Alert Setup](https://youtu.be/9DT7kR8fa0o)
- [Trigger Action](https://youtu.be/aCODvkJPOVY)
