# Home-NAS

A lightweight, container-based NAS system built on an unused Windows laptop repurposed as a Linux server.  
This project demonstrates practical system design and service orchestration using Ubuntu 24.04, Docker, Plex, Immich, Samba, and an external USB storage device.

## 1. Overview

This project converts a spare laptop into a functional home server capable of:

- Hosting a Plex media library  
- Running Immich as a self-hosted photo and video backup service  
- Sharing files over the local network using SMB  
- Mounting and serving data from a 5TB USB 3.0 external drive  
- Operating headlessly through SSH  

The goal was to build a reliable, low-cost NAS solution while applying modern infrastructure practices such as containerization, persistent storage mapping, and network service configuration.

## 2. System Architecture

```
Laptop (Ubuntu 24.04)
│
├── Docker Engine
│   ├── Plex Media Server
│   ├── Immich Server + Worker Components
│   ├── PostgreSQL (pgvector)
│   └── Redis
│
└── External 5TB NTFS Drive (mounted at /mnt/storage)
     ├── Movies/
     ├── immich/
     └── SMB Share: smb://<server-ip>/storage
```

Key design principles:

- Containers run statelessly; persistent data lives on the mounted storage  
- All services isolate configuration from stored media  
- Laptop lid-close events disabled for uninterrupted operation  
- Drive mounted via UUID for consistent availability  

## 3. External Storage Configuration

The 5TB NTFS volume required repair:

```
sudo ntfsfix /dev/sda1
```

Permanent mount via `/etc/fstab`:

```
UUID=<YOUR-UUID> /mnt/storage ntfs defaults,uid=1000,gid=1000,umask=022 0 0
```

This ensures stable media access for Plex and Immich after reboots.

## 4. Docker Compose Deployment

All services are orchestrated using Docker Compose.  
Below is the complete production-ready configuration:

```yaml
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

## 5. SMB File Sharing

Install Samba:

```
sudo apt install samba
```

Add share:

```
[storage]
    path = /mnt/storage
    browseable = yes
    writeable = yes
    valid users = <your-username>
```

Restart:

```
sudo systemctl restart smbd
```

Access from macOS:

```
smb://<server-ip>/storage
```

## 6. Operational Notes

Common commands:

```
docker ps
docker logs <container>
docker restart <container>
docker compose up -d
docker compose down
```

Laptop configured not to sleep on lid close:

```
HandleLidSwitch=ignore
```

## 7. Lessons Learned

- NTFS drives can require repair before Linux can mount them reliably  
- Immich requires the proper Postgres image with pgvector support  
- Container dependency ordering matters during initial database migration  
- SMB is the cleanest way to manage large files from macOS  
- A repurposed laptop is capable of running multiple media workloads efficiently  

## 8. Future Enhancements

- HTTPS and reverse proxy (Caddy / Traefik)  
- Automated PostgreSQL backups  
- Trigger-based media indexing for Plex  
- Cloud sync of critical data  
- Monitoring stack (Prometheus, Grafana)  
- Optional RAID mirroring with a second external drive  
