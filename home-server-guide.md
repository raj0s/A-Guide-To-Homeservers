# Building Your Own Home Server

A practical guide to turning a spare PC into a private cloud: your own photo/video backup, media streaming, and music library, accessible from anywhere, without a subscription.

> Reference video: [Kalos Likes Computers – Home Server](https://www.youtube.com/watch?v=IuRWqzfX1ik)

---

## Table of Contents

- [Why self-host?](#why-self-host)
- [Technologies at a glance](#technologies-at-a-glance)
- [What You Need to Start](#what-you-need-to-start)
- [Setting Up Your Server: Ubuntu Server + Docker](#setting-up-your-server-ubuntu-server--docker)
- [Docker Crash Course](#docker-crash-course)
- [Setting Up Nextcloud (files, photos, videos)](#setting-up-nextcloud)
- [Setting Up Jellyfin (movies & TV)](#setting-up-jellyfin)
- [Setting Up Navidrome + Amperfy (music)](#setting-up-navidrome--amperfy)
- [Remote Access: Tailscale vs. WireGuard](#remote-access-tailscale-vs-wireguard)
- [Project Ideas to Get Started](#project-ideas-to-get-started)
- [Maintenance Checklist](#maintenance-checklist)

---

## Why self-host?

- **No subscriptions.** Nextcloud, Jellyfin, and Navidrome replace paid cloud storage, Netflix-style hosting, and streaming music apps.
- **Privacy.** Your photos and files stay on hardware you control.
- **Reuse old hardware.** An old laptop, mini PC, or NAS box is usually enough to start.
- **A great excuse to learn Linux, Docker, and networking.**

**Minimum hardware to get going:** any x86_64 machine with 4 GB+ RAM, a multi-core CPU, and some storage (an SSD for the OS, plus a larger HDD/SSD for media works well). Anything from a 10-year-old laptop to a dedicated mini PC will work for a first server.

---

## Technologies at a glance

| | Technology | What it's for |
|---|---|---|
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/ubuntu.png" width="32" alt="Ubuntu logo">](https://ubuntu.com/server) | [Ubuntu Server LTS](https://ubuntu.com/server) | The base operating system for the server |
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/docker.png" width="32" alt="Docker logo">](https://www.docker.com) | [Docker](https://www.docker.com) | Runs every service in its own lightweight container |
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/nextcloud.png" width="32" alt="Nextcloud logo">](https://nextcloud.com) | [Nextcloud](https://nextcloud.com) | Self-hosted files, photo, and video backup |
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/jellyfin.png" width="32" alt="Jellyfin logo">](https://jellyfin.org) | [Jellyfin](https://jellyfin.org) | Movie and TV streaming |
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/navidrome.png" width="32" alt="Navidrome logo">](https://www.navidrome.org) | [Navidrome](https://www.navidrome.org) | Self-hosted music streaming server |
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/tailscale.png" width="32" alt="Tailscale logo">](https://tailscale.com) | [Tailscale](https://tailscale.com) | Mesh VPN for easy remote access |
| [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/wireguard.png" width="32" alt="WireGuard logo">](https://www.wireguard.com) | [WireGuard](https://www.wireguard.com) | The VPN protocol behind Tailscale, usable on its own |

---

## What You Need to Start

- **A server device.** An old laptop or desktop PC works fine for a first server. Look for 4 GB+ RAM, a 64-bit CPU, and enough storage for your media (an SSD for the OS, plus a larger HDD/SSD for files, is a comfortable setup).
- **A USB flash drive (8 GB or larger).** This is what you'll turn into a bootable installer for Ubuntu Server.
- **A monitor and keyboard, temporarily.** You only need these to get the OS installed and SSH enabled. Once that's done, you can unplug them and manage the server entirely from another computer over the network.
- **A separate computer** (Windows or macOS) to download the Ubuntu Server ISO and flash it onto the USB drive.
- **An Ethernet cable**, if possible. A wired connection is more reliable than Wi-Fi for a server that needs to stay online.
- **Access to your router's admin page**, for setting a static IP or DHCP reservation later on.

### Flashing the USB drive

First, download the Ubuntu Server LTS ISO from [ubuntu.com/download/server](https://ubuntu.com/download/server). Then follow the steps for your computer below.

**On Windows, using Rufus**

1. Download [Rufus](https://rufus.ie) (the portable version needs no installation).
2. Insert your USB drive, then open Rufus.
3. Under **Device**, make sure your USB drive is selected.
4. Click **SELECT** next to Boot selection, and choose the Ubuntu Server ISO you downloaded.
5. Leave **Partition scheme** on GPT for modern (post-2018) UEFI hardware, or switch to MBR for older BIOS-only machines.
6. Click **START**.
7. If Rufus asks about the ISO write mode, keep **Write in ISO Image mode** selected and click OK.
8. Confirm the warning that all data on the USB drive will be erased.
9. Wait for the process to finish, then safely eject the drive.

**On macOS, using Balena Etcher**

1. Download [Balena Etcher](https://etcher.balena.io).
2. Insert your USB drive, then open Etcher.
3. Click **Flash from file** and select the Ubuntu Server ISO.
4. Click **Select target** and choose your USB drive.
5. Click **Flash!** and enter your Mac password if prompted.
6. Etcher writes and automatically verifies the image; eject the drive when it's done.

Once the drive is flashed, plug it into your server, power it on, and open the boot menu (commonly `F2`, `F10`, `F12`, `Del`, or `Esc` right after power-on, depending on the manufacturer) to boot from the USB drive and start the Ubuntu Server installer.

---

## Setting Up Your Server: Ubuntu Server + Docker

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/ubuntu.png" width="60" alt="Ubuntu logo"> <img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/docker.png" width="60" alt="Docker logo">](https://ubuntu.com/server)

Every service in this guide runs as a Docker container on top of a plain Ubuntu Server LTS install. Ubuntu Server LTS is the default because it's free, well documented, has 5 years of security updates, and has the broadest hardware driver support of any mainstream Linux server distribution.

### 1. Install Ubuntu Server LTS

Boot from the USB drive you created earlier and walk through the installer. Download **Ubuntu Server 24.04 LTS** or the newer **26.04 LTS**, and install it headless (no desktop environment). Tick **"Install OpenSSH server"** when prompted, set a static IP or a DHCP reservation in your router, and update the system once it's up:

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Docker Engine + Compose

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

The second line lets you run `docker` without typing `sudo` every time (log out and back in for it to fully apply). Docker Compose v2 ships as a plugin with modern Docker installs, so `docker compose` (no hyphen) should already work. Check with:

```bash
docker compose version
```

### 3. Organize your folders

A clean layout keeps things sane as you add more services:

```
/srv/
├── docker/           # one folder per service, each with its docker-compose.yml
│   ├── jellyfin/
│   ├── navidrome/
│   └── nextcloud/
└── data/
    ├── movies/
    ├── tv/
    ├── music/
    └── photos/
```

### 4. Set up remote access

You have two options for reaching your server securely from outside your home network. Tailscale is the easier starting point; WireGuard gives you full control once you're comfortable managing it yourself.

**Option A: Tailscale (recommended for most people)**

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/tailscale.png" width="48" alt="Tailscale logo">](https://tailscale.com)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

This prints a login link. Open it once to connect the machine to your **tailnet** (your private mesh network). From then on, any other device you've added to that tailnet (phone, laptop) can reach your server at its Tailscale IP or MagicDNS name, from anywhere in the world, without opening any ports on your router.

**Option B: WireGuard (self-managed)**

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/wireguard.png" width="48" alt="WireGuard logo">](https://www.wireguard.com)

WireGuard is the protocol Tailscale is built on. Running it yourself means no third-party coordination server, at the cost of manually managing keys and port forwarding.

```bash
sudo apt install wireguard -y
wg genkey | tee privatekey | wg pubkey > publickey
```

Create `/etc/wireguard/wg0.conf` on the server:

```ini
[Interface]
PrivateKey = <server-private-key>
Address = 10.10.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client-public-key>
AllowedIPs = 10.10.0.2/32
```

Generate a matching key pair and config on your client device (phone/laptop), forward UDP port `51820` on your router to the server, then bring the interface up:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

Your client connects to `10.10.0.1` and can now reach every service running on the server, the same way it would on your home LAN.

---

## Docker Crash Course

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/docker.png" width="60" alt="Docker logo">](https://www.docker.com)

If you're new to Docker, here's the vocabulary you need for everything below:

- **Image:** a packaged application (e.g. `jellyfin/jellyfin`). You don't build these yourself; you pull them.
- **Container:** a running instance of an image. Stateless by default: anything not explicitly saved to a volume is lost on removal.
- **Volume / bind mount:** a folder on your host machine mapped into the container, so data survives updates and restarts.
- **docker-compose.yml:** a file describing one or more containers, their ports, volumes, and environment variables, so you can start the whole stack with one command.

### Everyday commands

```bash
docker compose up -d          # start services defined in docker-compose.yml, in the background
docker compose down           # stop and remove the containers (volumes are kept)
docker compose logs -f        # follow logs for troubleshooting
docker compose pull           # fetch newer images
docker compose up -d          # recreate containers with the new images
docker ps                     # list running containers
```

Rule of thumb: run `docker compose up -d` from inside each service's folder (where its `docker-compose.yml` lives).

---

## Setting Up Nextcloud

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/nextcloud.png" width="72" alt="Nextcloud logo">](https://nextcloud.com)

Nextcloud is your self-hosted Google Drive/Photos replacement: files, photo and video backup (via the Nextcloud mobile app's auto-upload), contacts, calendars, and more.

The officially recommended method is **Nextcloud All-in-One (AIO)**, a single master container that provisions and manages the rest of the stack (database, cache, reverse proxy) for you.

```bash
sudo docker run \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 80:80 \
  --publish 8080:8080 \
  --publish 8443:8443 \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  nextcloud/all-in-one:latest
```

Open `https://<server-ip>:8443` in a browser to reach the AIO setup wizard, which walks you through provisioning the full Nextcloud stack with one click.

**Photo/video backup:** install the Nextcloud mobile app, sign in, and enable **auto-upload** in the app settings. Every new photo and video on your phone backs up automatically over Wi-Fi (or Tailscale/WireGuard, when you're out).

---

## Setting Up Jellyfin

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/jellyfin.png" width="72" alt="Jellyfin logo">](https://jellyfin.org)

Jellyfin is a free, open-source alternative to Plex/Emby for movies and TV: no premium tier, no phone-home requirement.

`docker-compose.yml`:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 1000:1000
    network_mode: "host"
    volumes:
      - /srv/docker/jellyfin/config:/config
      - /srv/docker/jellyfin/cache:/cache
      - /srv/data/movies:/data/movies
      - /srv/data/tv:/data/tv
    restart: unless-stopped
```

```bash
docker compose up -d
```

Visit `http://<server-ip>:8096`, run through the setup wizard, and add `/data/movies` and `/data/tv` as your library folders. If your CPU supports hardware transcoding (Intel Quick Sync, for example), map `/dev/dri` into the container for smoother playback on weaker client devices.

---

## Setting Up Navidrome + Amperfy

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/navidrome.png" width="72" alt="Navidrome logo">](https://www.navidrome.org)

**Navidrome** is a self-hosted music server that speaks the Subsonic API, meaning a large ecosystem of existing mobile apps can connect to it out of the box, no custom app required.

`docker-compose.yml`:

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    user: 1000:1000
    ports:
      - "4533:4533"
    restart: unless-stopped
    volumes:
      - /srv/docker/navidrome/data:/data
      - /srv/data/music:/music:ro
```

```bash
docker compose up -d
```

Visit `http://<server-ip>:4533`, create an admin account, and let Navidrome scan your `/music` folder.

**Amperfy** (iOS/macOS) is a free, open-source Subsonic-compatible client, a natural pairing with Navidrome. In Amperfy, add a new server connection with:

- **Server URL:** `http://<server-ip>:4533` (or your Tailscale/WireGuard address, for access away from home)
- **Username / password:** the Navidrome account you just created

---

## Remote Access: Tailscale vs. WireGuard

[<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/tailscale.png" width="56" alt="Tailscale logo">](https://tailscale.com) [<img src="https://cdn.jsdelivr.net/gh/selfhst/icons/png/wireguard.png" width="56" alt="WireGuard logo">](https://www.wireguard.com)

A quick recap of the two options from the setup steps above, side by side:

| | Tailscale | WireGuard (self-managed) |
|---|---|---|
| Setup effort | Minutes: one install script, one login | More manual: key generation, config files, port forwarding |
| Router config | None needed (NAT traversal handled for you) | Usually need to forward a UDP port |
| Central dependency | Uses Tailscale's coordination servers to help peers find each other | None, fully self-contained |
| Under the hood | Built on WireGuard | N/A |
| Best for | Getting remote access working fast | Learning the fundamentals, full control, no third party involved |

Both give you an encrypted, direct connection to your server from anywhere. The difference is how much of the plumbing you want to manage yourself.

---

## Project Ideas to Get Started

A rough progression if you want a roadmap rather than doing everything at once:

1. **Week 1, get something running.** Install Ubuntu Server and Docker, get Jellyfin streaming one movie to your laptop over your home Wi-Fi.
2. **Week 2, go remote.** Install Tailscale, confirm you can reach Jellyfin from your phone on cellular data.
3. **Week 3, backup your photos.** Set up Nextcloud, install the mobile app, and turn on auto-upload.
4. **Week 4, add music.** Set up Navidrome, import your library, connect Amperfy.
5. **Month 2, harden it.** Swap Tailscale for a self-managed WireGuard tunnel; set up scheduled backups of your Docker volumes to an external drive or a second machine.
6. **Ongoing, expand the stack.** Once the core three services are stable, look into: Immich (photo backup with AI search), Home Assistant (smart home hub), or a `*arr` stack (Sonarr/Radarr) for automated media library management.

---

## Maintenance Checklist

- [ ] Set a static IP or DHCP reservation for the server
- [ ] Enable automatic security updates on the host OS (`unattended-upgrades` on Ubuntu)
- [ ] Back up Docker volumes / bind-mount folders regularly, off the server itself
- [ ] Periodically run `docker compose pull && docker compose up -d` in each service folder to update images
- [ ] Disable key expiry for trusted always-on devices in Tailscale, or renew WireGuard keys as needed
- [ ] Monitor disk space on your media/data drives

---

*Guide compiled for an internal presentation on home server basics. Core setup approach based on the workflow shown in [Kalos Likes Computers' home server video](https://www.youtube.com/watch?v=IuRWqzfX1ik). Technology icons courtesy of [selfh.st/icons](https://selfh.st/icons).*
