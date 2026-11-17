# Proxmox Media & Automation Stack (Jellyfin, qBittorrent, Radarr, Sonarr, Bazarr)

## How It Works (The Architecture)

This stack splits your media management into independent, lightweight unprivileged Proxmox LXC containers connected via shared host storage and Proxmox bind-mounts:

1. **qBittorrent (Download Client):** Handles searching, downloading, and seeding torrent files. It saves all raw downloads to a dedicated staging directory (`/mnt/data/downloads`).
2. **Radarr & Sonarr (Media Automation):** Monitor your media lists, talk to indexers, and command qBittorrent to download movies and TV shows. Once a download completes, Radarr/Sonarr automatically creates **hardlinks** from `/mnt/data/downloads` into your final media libraries (`/mnt/data/movies` and `/mnt/data/shows`) without duplicating disk space, allowing qBittorrent to continue seeding.
3. **Bazarr (Subtitle Automation):** Connects to Radarr and Sonarr, monitors your newly added movies and episodes, and automatically downloads matching subtitles (`.srt`) directly into `/mnt/data/movies` and `/mnt/data/shows` alongside your video files.
4. **Jellyfin (Media Server):** Continuously scans your organized `/mnt/data/movies` and `/mnt/data/shows` directories, fetches metadata, posters, and local subtitles, and streams your media to your client devices.

---

## Deployment & Setup Guide

### Step 1: Prepare Shared Storage on the Proxmox Host

Log into your **Proxmox host shell** and create the shared directories with proper permissions for unprivileged containers:

```bash
mkdir -p /mnt/data/movies /mnt/data/shows /mnt/data/downloads
chown -R 100000:100000 /mnt/data/movies /mnt/data/shows /mnt/data/downloads
chmod -R 775 /mnt/data/movies /mnt/data/shows /mnt/data/downloads
```

### Step 2: Create the LXC Containers

Run the following commands on your Proxmox host to create the four containers (adjusting IDs or network settings if needed):

```bash
# 1. qBittorrent Container (ID 202)
pct create 202 local:debian-12-standard_12.12-1_amd64.tar.zst --hostname qbittorrent --cores 2 --memory 1024 --swap 512 --rootfs local-lvm:8 --net0 name=eth0,bridge=vmbr0,ip=DHCP --unprivileged 1 --features nesting=1

# 2. Radarr Container (ID 203)
pct create 203 local:debian-12-standard_12.12-1_amd64.tar.zst --hostname radarr --cores 2 --memory 1024 --swap 512 --rootfs local-lvm:8 --net0 name=eth0,bridge=vmbr0,ip=DHCP --unprivileged 1 --features nesting=1

# 3. Sonarr Container (ID 204)
pct create 204 local:debian-12-standard_12.12-1_amd64.tar.zst --hostname sonarr --cores 2 --memory 1024 --swap 512 --rootfs local-lvm:8 --net0 name=eth0,bridge=vmbr0,ip=DHCP --unprivileged 1 --features nesting=1

# 4. Bazarr Container (ID 205)
pct create 205 local:debian-12-standard_12.12-1_amd64.tar.zst --hostname bazarr --cores 2 --memory 1024 --swap 512 --rootfs local-lvm:8 --net0 name=eth0,bridge=vmbr0,ip=DHCP --unprivileged 1 --features nesting=1
```

### Step 3: Configure Bind-Mounts

Edit the configuration files for each container on the Proxmox host to map your storage:

- **qBittorrent (`/etc/pve/lxc/202.conf`):** Add `mp0: /mnt/data/downloads,mp=/mnt/downloads`
- **Radarr (`/etc/pve/lxc/203.conf`):** Add `mp0: /mnt/data,mp=/mnt/data`
- **Sonarr (`/etc/pve/lxc/204.conf`):** Add `mp0: /mnt/data,mp=/mnt/data`
- **Bazarr (`/etc/pve/lxc/205.conf`):** Add `mp0: /mnt/data,mp=/mnt/data`

Start all four containers:

```bash
pct start 202 && pct start 203 && pct start 204 && pct start 205
```

### Step 4: Install Services Inside Containers

#### A. qBittorrent (`pct enter 202`)

```bash
apt update && apt install -y qbittorrent-nox
```

Create a systemd service (`/etc/systemd/system/qbittorrent.service`):

```ini
[Unit]
Description=qBittorrent Daemon
After=network.target

[Service]
Type=forking
User=root
ExecStart=/usr/bin/qbittorrent-nox -d --webui-port=8080
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
systemctl daemon-reload && systemctl enable --now qbittorrent
```

#### B. Radarr & Sonarr (`pct enter 203` and `pct enter 204`)

Install dependencies and use the official community installation scripts:

```bash
apt update && apt install -y curl sqlite3 libicu-dev

# Radarr installation (run inside container 203):
wget --no-check-certificate -O - https://raw.githubusercontent.com/Radarr/Radarr/develop/Distribution/install.sh | bash

# Sonarr installation (run inside container 204):
wget --no-check-certificate -O - https://raw.githubusercontent.com/Sonarr/Sonarr/develop/Distribution/install.sh | bash
```

#### C. Bazarr (`pct enter 205`)

Install Python dependencies, download the latest release, and configure it as a background service:

```bash
apt update && apt install -y 7zip python3-dev python3-pip python3-venv unrar unzip wget ffmpeg

mkdir -p /opt/bazarr
wget https://github.com/morpheus65535/bazarr/releases/latest/download/bazarr.zip -O /tmp/bazarr.zip
unzip /tmp/bazarr.zip -d /opt/bazarr
rm /tmp/bazarr.zip

python3 -m venv /opt/bazarr/venv
/opt/bazarr/venv/bin/pip install -r /opt/bazarr/requirements.txt
```

Create a systemd service (`/etc/systemd/system/bazarr.service`):

```ini
[Unit]
Description=Bazarr Daemon
After=syslog.target network.target

[Service]
WorkingDirectory=/opt/bazarr
User=root
Group=root
UMask=0002
Restart=on-failure
RestartSec=5
Type=simple
ExecStart=/opt/bazarr/venv/bin/python3 /opt/bazarr/bazarr.py
KillSignal=SIGINT
TimeoutStopSec=20
SyslogIdentifier=bazarr

[Install]
WantedBy=multi-user.target
```

Enable and start Bazarr:

```bash
systemctl daemon-reload && systemctl enable --now bazarr
```

---

## Connecting Everything Together

1. **Configure qBittorrent Download Path:**
   - Open qBittorrent (`http://<qbittorrent-ip>:8080`), go to **Options > Downloads**, and set the **Default Save Path** to `/mnt/downloads`.

2. **Link qBittorrent to Radarr/Sonarr:**
   - Open Radarr (`http://<radarr-ip>:7878`) and Sonarr (`http://<sonarr-ip>:8989`).
   - Go to **Settings > Download Clients > Add Client** and choose **qBittorrent**. Enter your qBittorrent container IP and port (`8080`).
   - _(Important)_ Because qBittorrent sees `/mnt/downloads` while Radarr/Sonarr see `/mnt/data/downloads`, add a **Remote Path Mapping** at the bottom of the Download Clients page in Radarr and Sonarr:
     - **Host:** `<qbittorrent-ip>`
     - **Remote Path:** `/mnt/downloads/`
     - **Local Path:** `/mnt/data/downloads/`

3. **Configure Root Folders:**
   - In Radarr (**Settings > Media Management**), add your root folder pointing to `/mnt/data/movies`.
   - In Sonarr (**Settings > Media Management**), add your root folder pointing to `/mnt/data/shows`.

4. **Link Bazarr to Radarr & Sonarr:**
   - Open Bazarr (`http://<bazarr-ip>:6767`).
   - Go to **Settings > Radarr** and **Settings > Sonarr**, enable each connection, and enter their respective container IPs, ports (`7878` and `8989`), and **API Keys** (found under **Settings > General** in Radarr/Sonarr).
   - Go to **Settings > Languages** to set up your desired subtitle languages and create a language profile, then add your subtitle providers (e.g., OpenSubtitles, Podnapisi) under **Settings > Providers**.

5. **Link to Jellyfin:**
   - Ensure Jellyfin has access to `/mnt/data/movies` and `/mnt/data/shows` via its own bind-mount from the host (`mp0: /mnt/data,mp=/mnt/data`).
   - Add the libraries inside the Jellyfin Web UI (`http://<jellyfin-ip>`) pointing to `/mnt/data/movies` and `/mnt/data/shows`. Jellyfin will automatically pick up the `.srt` files Bazarr saves next to your media.

