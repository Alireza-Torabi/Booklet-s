# NetBox Professional Deployment Guide

**Infrastructure:** VMware ESXi – Three-Tier Architecture (App / DB / Cache)

This guide is written for the following real-world scenario:

- 2000 devices  
- 1500 virtual machines  
- 1000 clients (optional)  
- Virtual infrastructure based on ESXi  
- No dedicated load balancer  
- Requirement for a Production-Stable role with full features (App + RQ + Redis + DB)  
- Package management via **Nexus Repository**  
- Enterprise security (separate DB, secured Redis, restricted ports)

This guide is step-by-step and **every step must be executed exactly as written**. No step is redundant or unnecessary.

---

## Part 1: Target Architecture and Deployment Topology

In this scenario, NetBox consists of three main components:

### 1. Application Layer

- One VM named: `Netbox-APP`  
- Runs Django + Gunicorn  
- Hosts API and UI  
- Includes RQ Worker  

### 2. PostgreSQL Database

- One VM named: `Netbox-DB`  
- PostgreSQL 15  
- Connections: Only from App  
- Authentication: Password-based  
- Listening only on internal IP  

### 3. Redis

- One small VM or on the App server (for small profiles, running on App is preferred)  
- Redis 7  
- Protected with `requirepass`  
- Two databases:
  - `tasks` → database 0  
  - `caching` → database 1  

This architecture relies on **VMware-level HA**, therefore duplicating VMs for High Availability is not required.

---

## Part 2: Repository Environment Preparation

**Objective:** Replace all external repositories with an internal **Nexus** repository.

### Nexus Configuration for Ubuntu

If the organization has no internet access or limited traffic:

- Create an APT Proxy Repository  
  - `Type: APT (proxy)`  
  - Upstream:  
    ```
    http://archive.ubuntu.com/ubuntu/
    ```

- PostgreSQL APT Repository → Proxy Repository  
  Upstream:
````

[https://apt.postgresql.org/pub/repos/apt/](https://apt.postgresql.org/pub/repos/apt/)

```

- PyPI Mirror → Proxy Repository  
Upstream:
```

[https://pypi.org/simple/](https://pypi.org/simple/)

```

**Result:** All installed packages are pulled from Nexus.

---

## Part 3: NetBox Database Server Deployment (Netbox-DB)

**Operating System:** Ubuntu 22.04 LTS

### Install PostgreSQL from Nexus Proxy

File: `/etc/apt/sources.list.d/pg.list`

```

deb [trusted=yes] [https://repo.mvmco.net/repository/apt-postgres/](https://repo.mvmco.net/repository/apt-postgres/) jammy-pgdg main

````

Then:

```bash
apt update
apt install postgresql-15 -y
````

---

### PostgreSQL Configuration

#### File: `/etc/postgresql/15/main/postgresql.conf`

Find:

```
#listen_addresses = 'localhost'
```

Change to:

```
listen_addresses = '192.168.169.171'
```

(This is the DB server IP.)

---

#### File: `/etc/postgresql/15/main/pg_hba.conf`

Add:

```
host    netbox     netbox     192.168.169.170/32      scram-sha-256
```

*Both occurrences of `netbox` must be lowercase.*

---

### Create Database and User

```bash
sudo -u postgres psql
CREATE USER netbox WITH PASSWORD 'VeryStrongDBPassword!';
CREATE DATABASE netbox OWNER netbox;
\q
```

---

### Restart Service

```bash
systemctl restart postgresql
```

---

### Test from APP Server

```bash
psql -h 192.168.169.171 -U netbox -d netbox
```

Connection must succeed after entering the password.

---

## Part 4: Redis Deployment (on APP)

```bash
apt install redis-server -y
```

### In `/etc/redis/redis.conf`

Enable:

```
requirepass VeryStrongRedisPassword!
```

Restart:

```bash
systemctl restart redis-server
```

Test:

```bash
redis-cli
AUTH VeryStrongRedisPassword!
PING
```

---

## Part 5: NetBox Installation on APP Server

**Operating System:** Ubuntu 22.04

### Prerequisites

```bash
apt update
apt install -y python3 python3-pip python3-venv python3-dev \
               build-essential libpq-dev redis-tools nginx \
               git unzip curl
```

---

### Download Official NetBox Release

```bash
cd /opt
wget https://github.com/netbox-community/netbox/archive/refs/tags/v4.4.0.tar.gz
tar -xzf v4.4.0.tar.gz
ln -sfn /opt/netbox-4.4.0 /opt/netbox
```

---

### Create NetBox User

```bash
adduser --system --group netbox
chown -R netbox:netbox /opt/netbox-4.4.0
chown -h netbox:netbox /opt/netbox
```

---

### Create Virtual Environment

```bash
cd /opt/netbox
sudo -u netbox python3 -m venv venv
sudo -u netbox venv/bin/pip install --upgrade pip
sudo -u netbox venv/bin/pip install -r requirements.txt
```

---

## Part 6: NetBox Configuration

### Main Configuration File

Copy the sample file:

```bash
cp /opt/netbox/netbox/netbox/configuration_example.py \
   /opt/netbox/netbox/netbox/configuration.py
```

Edit:

```bash
nano /opt/netbox/netbox/netbox/configuration.py
```

---

### Set `SECRET_KEY`

```python
SECRET_KEY = 'A_very_strong_secret_key_here'
```

---

### Database Settings

```python
DATABASE = {
    'NAME': 'netbox',
    'USER': 'netbox',
    'PASSWORD': 'VeryStrongDBPassword!',
    'HOST': '192.168.169.171',
    'PORT': '5432',
}
```

---

### Redis Settings

```python
REDIS = {
    'tasks': {
        'HOST': 'localhost',
        'PORT': 6379,
        'PASSWORD': 'VeryStrongRedisPassword!',
        'SSL': False,
        'DATABASE': 0,
    },
    'caching': {
        'HOST': 'localhost',
        'PORT': 6379,
        'PASSWORD': 'VeryStrongRedisPassword!',
        'SSL': False,
        'DATABASE': 1,
    }
}
```

---

## Part 7: Gunicorn Configuration

File: `/opt/netbox/gunicorn.py`

```python
import multiprocessing

bind = "127.0.0.1:8001"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "sync"
timeout = 120
keepalive = 5

pidfile = "/var/tmp/netbox.pid"
errorlog = "-"
accesslog = "-"
loglevel = "info"
```

---

## Part 8: Migrate and Collect Static Files

```bash
cd /opt/netbox
source venv/bin/activate

python3 netbox/manage.py migrate
python3 netbox/manage.py collectstatic
python3 netbox/manage.py createsuperuser
```

---

## Part 9: Systemd Services

### netbox.service

File: `/etc/systemd/system/netbox.service`

```
[Unit]
Description=NetBox WSGI Service
After=network-online.target

[Service]
Type=simple
User=netbox
Group=netbox
WorkingDirectory=/opt/netbox
PIDFile=/var/tmp/netbox.pid
ExecStart=/opt/netbox/venv/bin/gunicorn --pid /var/tmp/netbox.pid --pythonpath /opt/netbox/netbox --config /opt/netbox/gunicorn.py netbox.wsgi
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

---

### netbox-rq.service

```
[Unit]
Description=NetBox Request Queue Worker

[Service]
Type=simple
User=netbox
Group=netbox
WorkingDirectory=/opt/netbox
ExecStart=/opt/netbox/venv/bin/python3 /opt/netbox/netbox/manage.py rqworker high default low
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

### netbox-housekeeping

Copy from `/opt/netbox/contrib/`:

```bash
cp contrib/netbox-housekeeping.* /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now netbox-housekeeping.timer
```

---

## Part 10: Nginx Configuration

File: `/etc/nginx/sites-available/netbox`

```
server {
    listen 80;
    server_name 192.168.169.170;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable:

```bash
ln -s /etc/nginx/sites-available/netbox /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

---

## Part 11: Final Tests

### Service Status

```bash
systemctl status netbox
systemctl status netbox-rq
systemctl status netbox-housekeeping.timer
```

---

### Port Tests

```bash
curl http://127.0.0.1:8001
curl http://192.168.169.170
```

---

### UI Test

Open in browser:

```
http://192.168.169.170
```

The NetBox login page must be displayed.

---

## Part 12: Security

* PostgreSQL port must be open **only** to Netbox-APP
* Redis must listen only on `localhost`
* Gunicorn must bind only to `127.0.0.1`
* Nginx is the only entry point
* SELinux is optional (not enabled on Ubuntu)
* SSL can be enabled on Nginx using Let’s Encrypt or an internal CA

```
