# Home-NAS

A fully containerized, low-cost home NAS built from an unused Windows laptop running Ubuntu 24.04.  
This project demonstrates practical system design, Linux configuration, Docker orchestration, persistent storage design, and multi‑service deployment (Plex, Immich, Samba, SMART monitoring).

The goal was to build a reliable, headless NAS and media platform using commodity hardware — with architectural thinking similar to production systems.

---

# 1. System Architecture

```
Laptop Hardware (Ubuntu 24.04)
│
├── Docker Engine
│   ├── Plex Media Server
│   ├── Immich (Server / Web / Microservices / Machine Learning)
│   ├── PostgreSQL (pgvector)
│   ├── Redis
│   ├── Prometheus + Node Exporter + cAdvisor
│   └── Grafana
│
└── External 5TB USB 3.0 HDD  →  mounted at /mnt/storage
      ├── Movies/
      ├── immich/
      ├── monitoring/
      └── SMB share for macOS access
```

### Design Principles

- **All app data lives on `/mnt/storage`**, not inside containers
- **Laptop runs headless** via SSH (no keyboard/monitor needed)
- **Docker Compose defines all services** and ensures reproducible builds
- **External drive auto‑mounts on boot via UUID**
- **Prometheus + Grafana monitor CPU, memory, containers, and disk SMART health**
- **Samba enables macOS Finder access to the NAS**

---

# 2. Preparing the Laptop (Ubuntu Setup)

### Install Ubuntu 24.04  
Repurpose the old Windows laptop → format → install Ubuntu.

### Prevent laptop from sleeping
To ensure Plex, Immich and downloads run all night:

```
sudo nano /etc/systemd/logind.conf
HandleLidSwitch=ignore
```

```
sudo systemctl restart systemd-logind
```

---

# 3. Mounting the 5TB External HDD

### 1. Identify drive
```
lsblk
```

Example:

```
sda1  →  5TB NTFS external HDD
```

### 2. Fix NTFS issues (required)
```
sudo ntfsfix /dev/sda1
```

### 3. Get the UUID
```
sudo blkid
```

### 4. Add to `/etc/fstab`
```
UUID=<YOUR-UUID> /mnt/storage ntfs defaults,uid=1000,gid=1000,umask=022 0 0
```

### 5. Mount
```
sudo mount -a
```

Folder structure:

```
/mnt/storage
├── Movies
├── immich
└── monitoring
```

---

# 4. SMB File Sharing (macOS Access)

Install Samba:

```
sudo apt install samba
```

Edit config:

```
sudo nano /etc/samba/smb.conf
```

Add:

```
[storage]
    path = /mnt/storage
    browseable = yes
    writeable = yes
    valid users = randy_ubuntu
```

Restart:

```
sudo systemctl restart smbd
```

On Mac Finder:

```
smb://<server-ip>/storage
```

---

# 5. Docker Compose Deployment (Plex + Immich)

Below is the full working stack:

```
services:
  immich-server:
    image: ghcr.io/immich-app/immich-server:release
    container_name: immich-server
    environment:
      DB_HOST: immich-db
      DB_PORT: 5432
      DB_USERNAME: postgres
      DB_PASSWORD: immich
      DB_DATABASE_NAME: immich
      REDIS_HOST: immich-redis
    depends_on:
      - immich-db
      - immich-redis
    volumes:
      - /mnt/storage/immich:/usr/src/app/upload
    ports:
      - "2283:3001"
    restart: always

  immich-machine-learning:
    image: ghcr.io/immich-app/immich-machine-learning:release
    container_name: immich-machine
    restart: always

  immich-web:
    image: ghcr.io/immich-app/immich-web:release
    container_name: immich-web
    ports:
      - "2284:3000"
    restart: always

  immich-microservices:
    image: ghcr.io/immich-app/immich-server:release
    container_name: immich-micro
    command: ["start-microservices.sh"]
    environment:
      DB_HOST: immich-db
      DB_PORT: 5432
      DB_USERNAME: postgres
      DB_PASSWORD: immich
      DB_DATABASE_NAME: immich
      REDIS_HOST: immich-redis
    depends_on:
      - immich-db
      - immich-redis
    volumes:
      - /mnt/storage/immich:/usr/src/app/upload
    restart: always

  immich-db:
    image: ghcr.io/immich-app/pgvector:16
    container_name: immich-db
    environment:
      POSTGRES_DB: immich
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: immich
    volumes:
      - ./postgres:/var/lib/postgresql/data
    restart: always

  immich-redis:
    image: redis:7
    container_name: immich-redis
    restart: always

  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
    volumes:
      - /mnt/storage/Movies:/data/movies
      - ./plex/config:/config
    restart: always
```

Deploy:

```
docker compose up -d
```

---

# 6. Monitoring Stack (Prometheus + Grafana + Node Exporter + cAdvisor + SMART)

Monitoring folder: `/mnt/storage/monitoring`

Features:

- Container CPU / memory / restarts (cAdvisor)
- Host CPU / RAM / Disk IO (Node Exporter)
- HDD temperature, bad sectors, lifespan (SMART)
- Grafana dashboards for everything

## SMART Exporter (HDD/NVMe S.M.A.R.T. health)

```
smartctl-exporter:
  image: prometheuscommunity/smartctl-exporter:v0.14.0
  container_name: smartctl-exporter
  restart: always
  privileged: true
  ports:
    - "9902:9902"
  devices:
    - "/dev/sda:/dev/sda"
    - "/dev/nvme0:/dev/nvme0"
  volumes:
    - "/run/udev:/run/udev:ro"
```

Exposes:

- Temperature  
- Reallocated Sector Count  
- Pending Sector Count  
- Offline Uncorrectable  
- NVMe wear %  
- CRC Cable Errors  

Prometheus scrapes all metrics.

Grafana visualizes them in a dashboard.

---

# 7. Lessons Learned

- NTFS drives usually require repair before Linux mounting  
- Immich needs the pgvector-enabled Postgres image  
- Docker dependency order matters during first migration  
- SMB is the easiest way for macOS to interact with the NAS  
- A laptop + external drive is surprisingly capable as a homelab  
- Adding monitoring turns a DIY NAS into a semi‑professional system  
- SMART metrics provide early warnings of HDD failure  

---

# 8. Future Enhancements

- Reverse proxy (Caddy / Traefik) with HTTPS  
- Nightly Postgres backup to another disk or cloud  
- Automatic Plex library refresh hooks  
- A second external drive for mirroring (manual RAID1)  
- Tailscale remote access  
- Automated Ansible deployment  
