# <center>Cerebro Home Server</center>

# <center>Architecture and Installation Documentation</center>

## Table of Contents

* [Introduction](#introduction)
* |__[Objective](#objective)
* |__[Architecture - General](#architecture---general)
* |__[Architecture - Layers](#architecture---layers)
* |__[Storage](#storage)
* |__[IP Addressing](#ip-addressing)
* |__[DNS (Pi-hole)](#dns-pi-hole)
* |__[Reverse Proxy (Nginx Proxy Manager)](#reverse-proxy-nginx-proxy-manager)
* |__[External Access](#external-access)
* |__[Target State (goal)](#target-state-goal)
* [Hypervisor Proxmox VE](#hypervisor-proxmox-ve)
* |__[Prerequisites](#prerequisites) / [Dependencies](#dependencies)
* |__[Installation](#installation)
* |__[Configuration](#configuration)
* |__[Adjust repositories](#adjust-repositories)
* |__[Register storage](#register-storage)
* |__[Folder structure](#folder-structure)
* |__[Virtualization (standard LXC procedure)](#virtualization)
* [Pi-hole](#pi-hole)
* [NPM - Nginx Proxy Manager](#npm---nginx-proxy-manager)
* [Certificates (HTTPS)](#certificates-https)
* [Vaultwarden](#vaultwarden)
* [Seafile](#seafile)
* [Maintenance (updates)](#maintenance-updates)
* [TODOs](#todos)


## Introduction

### Objective

This document describes, in a consolidated manner, all the architecture decisions and the correct steps executed for creating a Home Server based on Proxmox VE, as discussed throughout the project. The goal is to allow manual reconstruction of an identical environment, avoiding already identified errors.

### Architecture - General

* Hypervisor: Proxmox VE 9 (host called 'pve'), Debian 13 "trixie"
* Model: 1 physical host + multiple LXC containers
* Network: LAN 192.168.1.0/24
* Local DNS: Pi-hole
* Reverse Proxy: Nginx Proxy Manager (NPM)
* Main services: Vaultwarden, Pi-hole, Seafile, Immich

### Architecture - Layers

| 1 - Hardware | 2 - Hypervisor | 3 - Base Infra | 4 - Services |
| :---- | :---- | :---- | :---- |
| Physical server | Proxmox VE | DNS | Vaultwarden |
| sda SSD (system) | ext4 | Local reverse proxy | Seafile |
| sdb HDD (data) |  | External access | Immich |
|  |  | Backup | etc |

### Storage

* SSD Disk (~240GB) - sda: used for Proxmox system and LXC root disks (`local-lvm`)
* HDD Disk (~1TB) - sdb: used for persistent data and logs (`data-hdd`)
* ext4 instead of ZFS
* HDD mounted on host at /srv/data
* Standard structure, one folder per service, bind-mounted into its LXC:
  * /srv/data/containers/<service>/data → the service's persistent data
  * /srv/data/containers/<service>/logs → the container's whole /var/log

#### Logs

- **Premise:** logs should go to sdb (1TB HDD), avoiding overloading sda (240GB SSD).
- Each container's whole /var/log is a bind mount of /srv/data/containers/<service>/logs. This covers system logs (journal), apt logs and the service's own logs, with no per-app setup. See [Logs on the HDD](#virtualization).
- Apps running in Docker write their logs to a folder under /var/log (e.g. /var/log/npm, /var/log/vaultwarden).
- Docker's own container logs can't leave Docker's data folder (SSD), so they are capped at 3 × 10MB per container in `/etc/docker/daemon.json`.
- Every new service (Seafile, Immich, ...) must follow this from the start.

### IP Addressing

- Proxmox Host (pve): 192.168.1.10
- Pi-hole: 192.168.1.11
- Vaultwarden: 192.168.1.12
- Nginx Proxy Manager: 192.168.1.13
- Seafile: 192.168.1.14
- IPs defined outside the router's DHCP range (pool 192.168.1.64 to 192.168.1.253, 1 hour lease) to avoid conflicts

### DNS (Pi-hole)

- Pi-hole configured as primary DNS on the network
- Router DHCP settings (the router keeps doing DHCP): primary DNS `192.168.1.11`, secondary DNS `192.168.1.11` (both Pi-hole, so no device can skip the filtering), domain name `lan`. Devices pick this up when they renew their lease.
- Trade-off: with both entries pointing to Pi-hole, if Pi-hole is down no device has DNS.
- Only DHCP clients get this. The host and the containers have fixed settings and **don't**: by default they use the router's own resolver (`192.168.1.1`), which doesn't forward to Pi-hole, doesn't know Pi-hole's local records and sometimes times out. Their DNS is set by hand (containers: see the DNS tab in [Virtualization](#virtualization); host: *pve > System > DNS*).
- Local records:
  vault.countess-lovelace.duckdns.org -> 192.168.1.13
  seafile.countess-lovelace.duckdns.org -> 192.168.1.13
  vault.lan -> 192.168.1.13 (fallback, local CA, main computer's browser)
  seafile.lan -> 192.168.1.13 (fallback, local CA, main computer's browser)
- Reason: all access goes through the reverse proxy
- The `.lan` records were removed and added back on 2026-11-22: the services moved to the public certificate, and `.lan` stayed as a fallback (see [Certificates](#certificates-https)).

### Reverse Proxy (Nginx Proxy Manager)

- Running in a dedicated LXC
- Ports: 80/443 (proxy), 81 (admin panel)
- Responsible for all internal HTTPS
- Let's Encrypt wildcard certificate (DNS challenge with DuckDNS) for all services, see [Certificates](#certificates-https)
- The local CA (OpenSSL) used before is kept as a fallback (`vault.lan` and `seafile.lan`, browser of the main computer) and as an option for internal-only services

### External Access

- Not yet configured. First decide whether external access is needed at all, and how.
- Options:

| Option | Needs a domain | Notes |
| :---- | :---- | :---- |
| **VPN** (WireGuard, or Tailscale) | No | Nothing is exposed publicly. The phone joins the home network and uses the services as if at home. The safest option, and a common choice for Vaultwarden and Seafile |
| **Port forwarding + DuckDNS** | No (DuckDNS is enough) | Services are public. Only works if the ISP gives a public IP (not CGNAT), and the exposure is managed by hand |
| **Cloudflare Tunnel** | Yes (~$10/year) | Services are public with no open router ports, and it works behind CGNAT. Traffic passes through Cloudflare |

- **Preferred direction: VPN.** Not decided or set up yet. A VPN covers reaching Vaultwarden and Seafile from outside with only the owner's devices. A public option (tunnel or port forwarding) is only needed to share Seafile links with other people or to reach services without a VPN app.
- Public CA certificates are independent of this choice, see [DuckDNS or your own domain](#duckdns-or-your-own-domain).

### Target State (goal)

This is the state the project aims for. It is **not** the current state.

- Pi-hole working as DNS
- Vaultwarden accessible via HTTPS
- Seafile accessible via HTTPS with upload and sync working
- NPM as single entry point
- Persistent data and logs on HDD
- Scheduled backups

## Hypervisor Proxmox VE

### Prerequisites:

* Complete and tested backup of sda
* Accept that the installation disk (sda) will be completely erased

### Dependencies:

* Proxmox bootable image
* USB pen drive
* Hardware virtualization

1. Download correct Proxmox VE (Virtual Environment) image.
2. Identify the pen drive
    ```bash
    lsblk -o NAME,SIZE,MODEL
    ```

3. Unmount the pen drive (pen drive identified as sdc)
   ```bash
   sudo umount /dev/sdc*
   ```

4. Burn the ISO
   ```bash
   sudo dd if=proxmox-ve.iso of=/dev/sdc bs=4M status=progress oflag=sync
   ```

5. Enable hardware virtualization in BIOS/UEFI.
   Depending on the manufacturer, look for:

   For Intel
    * Intel Virtualization Technology
    * VT-x
    * VT-d (also enable if present)

    For AMD
    * SVM Mode
    * AMD-V
      Usually located in:
        * Advanced
        * Advanced BIOS Features
        * CPU Configuration
        * Northbridge / Chipset

### Installation

When you boot from the Proxmox pen drive, you will see one of these initial screens.

Usually something like:

* Install Proxmox VE (Graphical)
* Install Proxmox VE (Terminal UI)
* Advanced Options
* Rescue Boot

Choose: Install Proxmox VE (Graphical)

Accept the license terms.

On the disk selection screen, ensure that sda (SSD ~240GB) is selected and the filesystem is ext4.

Accept the location and keyboard configuration, and root password definition.

In network configuration, the choice of network interface should match your physical network card driver. In my case:

* nic0 - something that looks like a hexadecimal (r8169)
* nic1 - hexadecimal (cdc_ncm)

In this case, nic1 was the USB interface and not the network card.

Still on the same screen, the hostname should already be filled in. "pve.lan" is an acceptable name, as well as "pve.local" or "proxmox.mydomain".

Also define a fixed IP outside your router's DHCP range. Ex:

* IP: 192.168.1.10
* Gateway: 192.168.1.1
* Netmask: 255.255.255.0
* DNS: 192.168.1.1

Proceed to complete the installation. It should take a few minutes. When done, it will ask you to reboot or will reboot on its own. Remove the pen drive. After boot, the screen will show something like:

>You can now connect to the Proxmox VE web interface:
>https://192.168.1.10:8006

No graphical interface, only maintenance terminal. You can log in through the server terminal with the root user and the password defined during installation, but it is recommended to access this address from another device on the network and start configuration through the GUI.

### Configuration

1. Go to another device
2. Open the browser
3. Access: https://192.168.1.10:8006
4. Ignore the certificate warning (normal)
5. Log in with:
   user: root
   password: the one you created

Basic system state:

Datacenter
└── pve

Click on *pve* and then the *Summary* tab. Confirm resources like CPU and memory and uptime.

#### Adjust repositories

By default, Proxmox uses "enterprise" (paid) repositories. Let's change to the free "no-subscription" repository.

> Proxmox VE 9 is based on Debian 13 **trixie**. Proxmox VE 8 was based on Debian 12 **bookworm**. The repository must match the Proxmox version, otherwise Proxmox packages are never updated.

**Option A - GUI (recommended)**

1. Click on *pve* > *Updates* > *Repositories*
2. Select the `pve-enterprise` repository and click *Disable*
3. Select the `ceph` enterprise repository (if present) and click *Disable*
4. Click *Add*, choose *No-Subscription* and click *Add*

**Option B - Shell**

Click on *pve* and then *shell*. Run the following commands.

Disable the enterprise repositories by renaming the files:

```bash
mv /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/pve-enterprise.sources.disabled
mv /etc/apt/sources.list.d/ceph.sources /etc/apt/sources.list.d/ceph.sources.disabled
```

Add the no-subscription repository:

```bash
nano /etc/apt/sources.list.d/proxmox.sources
```

Add to the file:

```ini
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```

Save and exit.

**Both options:** update and upgrade:

```bash
apt update && apt full-upgrade -y
```

Confirm that Proxmox packages come from the trixie repository:

```bash
apt-cache policy pve-manager
```

The `Candidate` version should be 9.x, coming from `trixie/pve-no-subscription`.

If you get an "unauthorized" (401) error, Proxmox still tries to access an enterprise repository. Search with:

`grep -r "enterprise.proxmox.com" /etc/apt/`

If anything appears in an active file, disable it by adding ".disabled" to the end of the filename. Run the update and upgrade commands again.

#### Register storage

1) Open *shell* and run:

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

Confirm that the system disk (sda) and data disk (sdb) appear.

2) Create GPT partition table on HDD:

```bash
parted /dev/sdb --script mklabel gpt
```

3) Create single partition that occupies entire disk:

```bash
parted /dev/sdb --script mkpart primary ext4 0% 100%
```

4) Confirm:

```bash
lsblk
```

You should see something like:

> sdb
└─sdb1 1T

5) Create ext4 filesystem:

```bash
mkfs.ext4 -L data /dev/sdb1
```

* -L data creates a useful label
* ext4 is simple, stable and ideal for this case

6) Create mount point:

```bash
mkdir -p /srv/data
```

This doesn't create anything on sdb, just an empty directory on the system disk that will be used as mount point.

7) Temporarily mount HDD (test):

```bash
mount /dev/sdb1 /srv/data
```

If something like below appears when running `df -h | grep data` it means it worked.

> /dev/sdb1   916G   ...  /srv/data

8) Permanently mount HDD:

Get UUID with `blkid /dev/sdb1`

Edit fstab with `nano /etc/fstab` and add at the end

> UUID=xxxx-xxxx-xxxx  /srv/data  ext4  defaults,nofail  0  2

* `nofail` → if the HDD is missing or broken, the server still boots (without it, boot stops in emergency mode and you lose remote access to Proxmox).

Save and exit.

9) Confirm:

```bash
umount /srv/data

mount -a
```

If this message appears "*mount: (hint) your fstab has been modified, but systemd still uses the old version; use 'systemctl daemon-reload' to reload.*" it just means systemd still has the old version in memory. You can run the command from the message to update it and try the mount again.

```bash
df -h | grep data
```

10) Integrate and add to Proxmox:

In the web panel go to Datacenter > Storage > Add > Directory

Fill in:

* Id: data-hdd
* Directory: /srv/data
* Content: check
  * Disk image
  * Container
  * Backup
* Nodes: pve

Save.

11) Tell Proxmox that /srv/data must be a mounted disk (shell only, there is no GUI option):

```bash
pvesm set data-hdd --is_mountpoint yes
```

* Because of `nofail`, the server can boot without the HDD. In that case /srv/data is just an empty folder on the SSD.
* With `is_mountpoint`, Proxmox marks data-hdd as unavailable instead of silently writing backups and disks into that folder (which would fill the SSD).

Verify that data-hdd storage appears in the list, is enabled:yes and has no alerts or errors in red.

Up to here we integrated a data disk (sdb) and have a disk (sda) isolated for system.

#### Folder structure

1) Run `mkdir -p /srv/data/{backups,containers,media,services}`

This creates:

* backups → services backups (Proxmox backups go to `dump/`, see below)
* containers → persistent LXC data
* media → Plex / Immich
* services → configs (Pi-hole, Vaultwarden, etc.)

Proxmox creates these folders itself, because data-hdd is registered with content Disk image, Container and Backup:

* dump → Proxmox backups (vzdump)
* images → VM disks
* private → LXC root disks stored on data-hdd

2) Verify they were created:

```bash
ls -l /srv/data
```

#### Virtualization

The chosen option, as mentioned in the architecture and the introduction proposal, is LXC (Linux containers) because it is lighter, uses the host kernel and is simpler to configure and maintain. It is ideal for services like pi-hole, vaultwarden, npm, seafile, etc.

`Standard procedure for each service`

1) Download Debian template if not already present. Go to:

>pve > local > CT Templates

Click on Templates

Download `debian-12-standard_*.tar.zst` (currently `debian-12-standard_12.12-1_amd64.tar.zst`). Wait for it to finish.

2) Create the container.

Click on Create CT

Fill in:

General Tab

* CT ID: <id, LXCs use 1xx (100, 101, ...)>
* Hostname: <service name>
* Unprivileged container: <checked, unless the service section says otherwise>
* Nesting: <checked, required for Docker and for Debian 12 systemd>
* Password
* SSH public key: <leave empty>

  Click Next.

Template Tab

* Storage: local
* Template: debian-12-standard_12.12-1_amd64.tar.zst

  Click Next.

Disks Tab

* Storage: "local-lvm" (system SSD). All current containers use local-lvm; persistent data goes to the HDD through bind mounts.
* Disk Size (GiB): <size>
  Click Next.

CPU Tab

* Cores: <quantity>

  Click Next.

Memory Tab

* Memory: <quantity> MB
* Swap: <quantity> MB

  Click Next.

Network Tab

* Name: eth0
* Bridge: vmbr0
* IPv4: Static
  * IPv4/CIDR: <service IP>/24 (see [IP Addressing](#ip-addressing))
  * Gateway: 192.168.1.1
* IPv6: None

  Click Next.

DNS Tab

* DNS domain: `lan`
* DNS servers: `192.168.1.11 1.1.1.1` (Pi-hole first, Cloudflare as fallback). Don't leave "use host": containers have fixed IPs, so they don't get the DNS the router's DHCP hands out. For Pi-hole itself (100) use only `1.1.1.1`, so it never depends on itself.
* Already created containers: `pct set <id> --nameserver "192.168.1.11 1.1.1.1" --searchdomain lan`, then `pct reboot <id>`.

  Click Next.

Confirm Tab

* Check everything
* Uncheck "start after created"

  Click Finish.

3) Verification

The container assumes a CT ID and appears as Stopped.

4) Logs on the HDD

Do this **before the first start**, on the Proxmox host. It copies the template's /var/log to the HDD and mounts it back as /var/log, so all the container's logs are written to the HDD from day one.

```bash
ID=<ct id>; NAME=<service name>; MP=<next free mount point, e.g. mp1>

mkdir -p /srv/data/containers/$NAME/logs
pct mount $ID
cp -a /var/lib/lxc/$ID/rootfs/var/log/. /srv/data/containers/$NAME/logs/
chown 100000:100000 /srv/data/containers/$NAME/logs
rm -rf /var/lib/lxc/$ID/rootfs/var/log/*
pct unmount $ID
pct set $ID -$MP /srv/data/containers/$NAME/logs,mp=/var/log
```

* Copying the template's /var/log (instead of mounting an empty folder) keeps `/var/log/journal`, which systemd needs to keep system logs on disk.
* `rm -rf .../var/log/*` frees the old copy on the SSD; it would be hidden under the mount anyway.
* 100000 → the container's root user as seen by the host (unprivileged containers shift UIDs by 100000).
* Each service section below says which ID, name and mount point to use.

5) HTTPS certificate

For every service that is reached through NPM (all of them), once the service is installed and answers on its own IP and port: add the Pi-hole local record `<service>.countess-lovelace.duckdns.org` and an NPM proxy host that uses the wildcard certificate, following [Certificates](#certificates-https) (*For every service*). No new certificate is needed.

## Pi-hole

### Prepare LXC

1. Create LXC container (step by step from [virtualization configuration](#virtualization)).

    General:
    - CT ID: 100
    - Hostname: pihole
    - Unprivileged: true
    - Nesting: true
    - Password: <define>
    - SSH public key: empty

    Template:
    - Debian 12

    Disks:
    - Storage: local-lvm
    - Disk size: 8GB

    CPU:
    - Cores: 1

    Memory:
    - 512MB
    - Swap: 512MB

    Network:
    - Name: eth0
    - Bridge: vmbr0
    - IPv4: Static 192.168.1.11/24, Gateway 192.168.1.1
    - IPv6: None

    DNS:
    - DNS domain: lan
    - DNS servers: 1.1.1.1 only (Pi-hole is the network DNS, so it must not depend on itself)

    Or simply run:
    ```bash
    pct create 100 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
    --hostname pihole \
    --cores 1 \
    --memory 512 \
    --swap 512 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.11/24,gw=192.168.1.1,firewall=1,type=veth \
    --nameserver 1.1.1.1 \
    --searchdomain lan \
    --rootfs local-lvm:8 \
    --unprivileged 1 \
    --features nesting=1 \
    --onboot 1 \
    --password
    ```

    - `--nameserver 1.1.1.1` → Pi-hole doesn't use itself as DNS.
    - `--onboot 1` → starts on boot (step 2 below does the same, for a container created in the GUI).

2. Start on boot

    Pi-hole is the network DNS: if it's down, name resolution on the LAN is down.

    ```bash
    pct set 100 --onboot 1
    ```

    Or in the GUI: *100 (pihole)* > *Options* > *Start at boot* > Yes.

3. Create persistent data directory (before installing Pi-hole)

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/pihole/data
    chown -R 100000:100000 /srv/data/containers/pihole
    ```

    - 100000 → the container's root user as seen by the host (unprivileged containers shift UIDs by 100000)
    - The installer needs to write into this folder

4. Configure bind mount for persistent data

    ```bash
    pct set 100 -mp0 /srv/data/containers/pihole/data,mp=/etc/pihole
    ```

    - /srv/data/containers/pihole/data → host (HDD)
    - /etc/pihole → inside the container: Pi-hole config (`pihole.toml`), blocklists (`gravity.db`) and the long-term query database (`pihole-FTL.db`, which grows over time, so it's good that it's on the HDD)

5. Move logs to the HDD (before the first start)

    Follow [Logs on the HDD](#virtualization) with `ID=100; NAME=pihole; MP=mp1`.

    - Pi-hole's logs go to /var/log/pihole. `pihole.log` records every DNS query (~16MB a day), so it's the biggest log on the server. Pi-hole rotates its own logs (`/etc/pihole/logrotate`).

### Install service

1. Enter the container

    ```bash
    pct start 100
    pct enter 100
    ```

2. Update container and install curl

    ```bash
    apt update
    apt upgrade -y
    apt install -y curl
    ```

3. Run the official installer

    ```bash
    curl -sSL https://install.pi-hole.net | bash
    ```

    Installer choices:
    - Interface: eth0
    - Upstream DNS provider: Cloudflare (1.1.1.1 / 1.0.0.1)
    - Blocklists: keep the default list
    - Query logging: on
    - Privacy mode: 0 (show everything)

    At the end the installer shows the admin password. Change it with:

    ```bash
    pihole setpassword
    ```

    After the install, the host folder owner changes to `100999:101000`. That is the `pihole` user (UID 999) inside the container. This is expected, keep it.

### Configure

Web interface: `http://192.168.1.11/admin` (also `https://192.168.1.11/admin` with Pi-hole's own self-signed certificate).

1. DNS settings (*Settings* > *DNS*)
    - Upstream DNS servers: 1.1.1.1 and 1.0.0.1 (Cloudflare)
    - Interface settings: "Allow only local requests" (answers only devices on the local network)

2. DHCP (*Settings* > *DHCP*)
    - Disabled. The router keeps doing DHCP.

3. Local DNS records (*Settings* > *Local DNS Records*)

    | Domain | IP | Reason |
    | :---- | :---- | :---- |
    | vault.countess-lovelace.duckdns.org | 192.168.1.13 | Vaultwarden, through NPM |
    | seafile.countess-lovelace.duckdns.org | 192.168.1.13 | Seafile, through NPM |
    | vault.lan | 192.168.1.13 | Fallback with the local CA (main computer's browser) |
    | seafile.lan | 192.168.1.13 | Fallback with the local CA (main computer's browser) |

    All records point to NPM, never directly to the service.

    Or from the container shell:

    ```bash
    pihole-FTL --config dns.hosts '["192.168.1.13 vault.countess-lovelace.duckdns.org","192.168.1.13 seafile.countess-lovelace.duckdns.org","192.168.1.13 vault.lan","192.168.1.13 seafile.lan"]'
    ```

4. Router: make devices use Pi-hole

    In the router's DHCP/LAN settings, set the DNS server handed out to devices to `192.168.1.11`.
    Without this step, devices keep using the router as DNS and nothing is filtered, and the local records don't resolve.
    Devices pick up the new DNS when they renew their DHCP lease (reconnect to the network).

### Verify

From the Proxmox host or any device on the network:

```bash
dig +short @192.168.1.11 vault.countess-lovelace.duckdns.org
```

Should return `192.168.1.13`.

In the Pi-hole dashboard, queries from your devices should start appearing.

## NPM - Nginx Proxy Manager

### Prepare LXC

1. Create LXC container (step by step from [virtualization configuration](#virtualization)).

    General:
    - CT ID: 102
    - Hostname: npm
    - Unprivileged: true
    - Nesting: true
    - Password: <define>
    - SSH public key: empty

    Template:
    - Debian 12

    Disks:
    - Storage: local-lvm
    - Disk size: 8GB

    CPU:
    - Cores: 1

    Memory:
    - 1024MB
    - Swap: 512MB

    Network:
    - Name: eth0
    - Bridge: vmbr0
    - IPv4: Static 192.168.1.13/24, Gateway 192.168.1.1
    - IPv6: None

    DNS:
    - DNS domain: lan
    - DNS servers: 192.168.1.11 1.1.1.1 (Pi-hole first, Cloudflare as fallback)

    Or simply run:
    ```bash
    pct create 102 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
    --hostname npm \
    --cores 1 \
    --memory 1024 \
    --swap 512 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.13/24,gw=192.168.1.1,firewall=1,type=veth \
    --nameserver "192.168.1.11 1.1.1.1" \
    --searchdomain lan \
    --rootfs local-lvm:8 \
    --unprivileged 1 \
    --features keyctl=1,nesting=1 \
    --onboot 1 \
    --password
    ```

    - `keyctl=1,nesting=1` → required for Docker inside an unprivileged LXC.
    - `--onboot 1` → starts on boot (the step "Start on boot" below does the same, for a container created in the GUI).

2. Ensure container configuration is (only needed if the container was created in the GUI)

    ```bash
    nano /etc/pve/lxc/102.conf
    ```

    Content (keep the lines Proxmox generated, make sure these are present):
    ```ini
    features: keyctl=1,nesting=1
    unprivileged: 1
    ```

    - `keyctl=1,nesting=1` → required for Docker inside an unprivileged LXC
    - No extra `lxc.*` lines are needed. Earlier versions of this guide added `lxc.apparmor.profile: unconfined`, `lxc.cgroup2.devices.allow: a` and `lxc.mount.auto: proc:rw sys:rw` during troubleshooting. They remove most of the container's isolation, and NPM was tested without them (2026-11-20), so don't add them.

3. Create persistent data directories

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/npm/{data,letsencrypt}
    chown -R 100000:100000 /srv/data/containers/npm
    ```

    - 100000 → the container's root user as seen by the host (unprivileged containers shift UIDs by 100000). NPM runs as root inside its Docker container.

4. Configure bind mounts for persistent data

    ```bash
    pct set 102 -mp0 /srv/data/containers/npm/data,mp=/data/npm -mp1 /srv/data/containers/npm/letsencrypt,mp=/data/letsencrypt
    ```

    - /srv/data/containers/npm/data → /data/npm inside the container: NPM config, database, proxy hosts and uploaded certificates
    - /srv/data/containers/npm/letsencrypt → /data/letsencrypt inside the container: Let's Encrypt certificates (the wildcard certificate)
    - Docker itself (/data/docker: images, layers, cache) stays on the container disk (SSD). It can always be downloaded again.

5. Move logs to the HDD (before the first start)

    Follow [Logs on the HDD](#virtualization) with `ID=102; NAME=npm; MP=mp2`.

6. Start on boot

    NPM is the single entry point for every service (`*.countess-lovelace.duckdns.org` → 192.168.1.13): if it's down, no service is reachable by name.

    ```bash
    pct set 102 --onboot 1
    ```

    Or in the GUI: *102 (npm)* > *Options* > *Start at boot* > Yes.

### Install service

1. Enter the container

    ```bash
    pct start 102
    pct enter 102
    ```

2. Update container

    ```bash
    apt update
    apt upgrade -y
    ```

3. Install Docker from Docker's official repository (gives the `docker compose` command)

    ```bash
    apt install -y ca-certificates curl
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    chmod a+r /etc/apt/keyrings/docker.asc

    echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" \
      > /etc/apt/sources.list.d/docker.list

    apt update
    apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

    - `bookworm` here is correct: it's the container's Debian version (Debian 12), not the host's.

4. Set Docker's data folder to /data/docker

    ```bash
    systemctl stop docker
    mkdir -p /data/docker
    nano /etc/docker/daemon.json
    ```

    Content:

    ```json
    {
      "data-root": "/data/docker",
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```

    ```bash
    systemctl start docker
    docker info | grep "Docker Root Dir"
    ```

    Should show `Docker Root Dir: /data/docker`.

    - /data/docker is a normal folder inside the container disk (SSD), not a bind mount. Docker images, layers and cache are stored there (~1GB).
    - `log-opts` → Docker's own container logs (`docker logs`) stay in /data/docker on the SSD, so they're capped at 3 files of 10MB. NPM's real logs go to the HDD (next step).

5. Configure NPM

    ```bash
    mkdir -p /var/log/npm
    nano /data/docker-compose.yml
    ```

    Content:

    ```yaml
    services:
      npm:
        image: jc21/nginx-proxy-manager:latest
        container_name: nginx-proxy-manager
        restart: unless-stopped
        ports:
          - "80:80"
          - "81:81"
          - "443:443"
        volumes:
          - /data/npm:/data
          - /data/letsencrypt:/etc/letsencrypt
          - /var/log/npm:/data/logs
    ```

    - /data/npm and /data/letsencrypt are the bind mounts from Prepare LXC step 4 (HDD)
    - /var/log/npm → NPM's logs (one access and error log per proxy host), on the HDD through the /var/log bind mount. NPM rotates them itself.
    - Docker creates an empty `logs` folder inside /data/npm as the mount point for /var/log/npm. That's expected.

6. Bring up:

    ```bash
    cd /data
    docker compose up -d
    ```

### Verify

1. Open `http://192.168.1.13:81` in the browser
2. Create the admin account
3. Access the dashboard

```bash
docker ps
```

Should show the `nginx-proxy-manager` container as `Up`.

## Certificates (HTTPS)

Two ways to get HTTPS for the services behind NPM:

- **Public certificate with DuckDNS** (since 2026-11-22): the main way. Works on every device, including the Android apps.
- **Local CA**: kept as a **fallback**, `vault.lan` and `seafile.lan` in the browser of the main computer only. Also the option for services that are only accessed internally.

### Public certificate with DuckDNS (in use)

One Let's Encrypt **wildcard** certificate (`*.countess-lovelace.duckdns.org`) that NPM gets with the *DNS challenge*. It replaced the local CA for Vaultwarden and Seafile because the Android apps (Bitwarden, Seafile) rejected the local CA. Devices need nothing installed.

| Service | Name |
| :---- | :---- |
| Vaultwarden | `vault.countess-lovelace.duckdns.org` |
| Seafile | `seafile.countess-lovelace.duckdns.org` |

Nothing is exposed to the internet: DuckDNS only holds the temporary record used to validate the challenge. The names resolve to NPM (`192.168.1.13`) through Pi-hole's local records, see [Pi-hole](#pi-hole).

#### Set up (once)

1. DuckDNS: sign in at duckdns.org, create a subdomain (`countess-lovelace`) and copy the token.
    - The subdomain name becomes public in the Certificate Transparency logs, so don't use anything that identifies you.
    - The token is a secret (it can change your DuckDNS records). Enter it only in NPM: never in this guide, in the repository or in a chat.
2. NPM: *SSL Certificates* > *Add SSL Certificate* > *Let's Encrypt via DNS*
    - Domain names: `*.countess-lovelace.duckdns.org` **and** `countess-lovelace.duckdns.org`
    - DNS provider: DuckDNS, with the token as credentials
    - A certificate only for `countess-lovelace.duckdns.org` does **not** cover `vault.countess-lovelace.duckdns.org` (the first attempt was like that and had to be redone). The wildcard name is what covers the services.
3. Check what was issued (on the host; the folder name `npm-<n>` changes with each certificate):

    ```bash
    openssl x509 -in /srv/data/containers/npm/letsencrypt/live/npm-<n>/fullchain.pem -noout -issuer -enddate -ext subjectAltName
    ```

    It must list `DNS:*.countess-lovelace.duckdns.org` and `DNS:countess-lovelace.duckdns.org`, with *Let's Encrypt* as the issuer.

#### For every service (new ones too)

No new certificate is needed, the wildcard covers any `<service>.countess-lovelace.duckdns.org`.

1. Pi-hole local record `<service>.countess-lovelace.duckdns.org` → `192.168.1.13` (NPM, never the service itself).
2. NPM proxy host for that name, with the wildcard certificate, *Force SSL* and *HTTP/2 Support*.
3. Set the service's own public URL: `DOMAIN` in Vaultwarden, `SEAFILE_SERVER_HOSTNAME` in Seafile (recreate the container afterwards).
4. Test: open it in the browser, run `curl https://<service>.countess-lovelace.duckdns.org` **without** `--cacert` (it must verify), and test the Android apps.

#### Renewal

Let's Encrypt certificates last 90 days. NPM renews them by itself. Check the expiry date in NPM's *SSL Certificates* list from time to time. If the DuckDNS token ever changes, update it in NPM.

#### Limits to know

- The names only resolve to NPM through Pi-hole. If Pi-hole is down, no name resolves. Away from home they don't resolve to the private address, so reaching them from outside needs a VPN that uses Pi-hole as DNS, see [External Access](#external-access).
- If DuckDNS is down when a renewal is due, the renewal fails, but the current certificate keeps working until it expires.
- For a service that must not appear in public records, or that doesn't need a public certificate, use the local CA below.

#### DuckDNS or your own domain

With a real domain (e.g. `vault.mydomain.com`) the procedure is the same, using the DNS provider's token (Cloudflare) instead of DuckDNS's. DuckDNS was chosen on 2026-11-22 because it's free.

**What is free and what is not:** the certificate (Let's Encrypt) and a Cloudflare account are free, but **the domain is not**: it costs about $10 a year (Cloudflare Registrar sells them at cost; one bought elsewhere can use Cloudflare's DNS by pointing its nameservers there). The free way is DuckDNS.

| Option | Cost | Names | Notes |
| :---- | :---- | :---- | :---- |
| Own domain + Cloudflare | ~$10/year (domain) | `vault.mydomain.com` | Cleaner names and full control. Only required for the certificates if you want your own domain, and required if Cloudflare Tunnel is chosen for external access. API token limited to *Zone → DNS → Edit* |
| DuckDNS | Free | `vault.yourname.duckdns.org` | NPM supports its DNS challenge. Enough for the certificates and for a VPN or port forwarding setup. Depends on a free third-party service, and can't be used with Cloudflare Tunnel |

- Let's Encrypt publishes every certificate in the public Certificate Transparency logs, so the hostnames become visible. A wildcard certificate (`*.mydomain.com`) avoids listing each service.
- Switching later from DuckDNS to an own domain means repeating the steps above for the new names.

### Local CA (fallback, browser only)

> Used from 2026-11-17 to 2026-11-22 as the main way for `vault.lan` and `seafile.lan`. Since 2026-11-22 the public certificate is the main way, and the local CA is kept as a **fallback** that is used only from the browser of the main computer, which needs the CA installed in its browser or system (step 6 below). It was removed from the phone.
>
> What is kept for it:
> - The CA and the service certificates: `/root/ca` (and the CA in `/srv/data/backups/ca`), and the two *Custom* certificates in NPM (`vault.lan (local CA)`, `seafile.lan (local CA)`).
> - Pi-hole local records `vault.lan` and `seafile.lan` → `192.168.1.13`, next to the DuckDNS ones.
> - NPM proxy hosts `vault.lan` (→ `http://192.168.1.12:8080`) and `seafile.lan` (→ `http://192.168.1.14:80`), each with its local CA certificate, WebSockets, *Force SSL* and *HTTP/2 Support*.
>
> Limits:
> - Android apps ignore CAs installed by the user, so Bitwarden and Seafile on Android don't work with it. Browsers do.
> - `DOMAIN` (Vaultwarden) and `SEAFILE_SERVER_HOSTNAME` (Seafile) are set to the DuckDNS names. `.lan` is a second door into the same service: from the host it answers correctly with the local CA (`curl --cacert /root/ca/cerebro-ca.crt https://vault.lan` and `https://seafile.lan/api2/ping/`), but links and emails the services generate use the DuckDNS name. Logging in through `.lan` in the browser works (tested 2026-11-22 on Vaultwarden and Seafile).
> - Certificates here last 825 days. See [Renewing a certificate](#renewing-a-certificate).

`.lan` is not a public domain, so Let's Encrypt can't issue certificates for it. The local CA is a small Certificate Authority we create ourselves with OpenSSL: one CA, one certificate per service, uploaded to NPM as *Custom* certificates. For every service, follow steps 2 to 5 below (create the certificate, copy it, upload it to NPM, use it in the proxy host) and test it with step 7. Devices that already have the CA installed (step 6) need nothing.

#### Why not self-signed certificates

The first attempt was one self-signed certificate per service. It was replaced by the local CA because:

- A self-signed certificate with only a Common Name (CN) and no *Subject Alternative Name* (SAN) is rejected by modern browsers (Chrome, Edge, Safari) and Android, even if you trust it manually.
- Every device had to trust every certificate one by one.
- Apps (Bitwarden, Seafile sync clients) refuse self-signed certificates.

#### The local CA

Create your own small CA once, install it on each device once, and sign one certificate per service with it. Every device then trusts every service, now and in the future.

What goes where:

| File | Where it lives | Who needs it |
| :---- | :---- | :---- |
| `cerebro-ca.key` (CA private key) | Only on the host, `/root/ca` + backups | Nobody else. Never copy it to a device or to NPM |
| `cerebro-ca.crt` (CA certificate) | Host, and **every device** that opens the services | Devices, to trust the CA. It's public, safe to share |
| `<name>.crt` + `<name>.key` (service certificate) | Host, and uploaded to **NPM** | NPM only. Devices never need them |

The CA lives in `/root/ca` on the host: only root can read it, and no container or Proxmox storage uses that folder. (`/srv/data/private` is not an option: Proxmox created it for LXC disks.) The downside is that `/root` is on the SSD and is lost if Proxmox is reinstalled, so the CA files must be backed up (step 1).

1. Create the CA (once, on the Proxmox host)

    ```bash
    mkdir -p /root/ca && chmod 700 /root/ca
    cd /root/ca

    openssl req -x509 -nodes -newkey rsa:4096 -days 3650 \
      -keyout cerebro-ca.key \
      -out cerebro-ca.crt \
      -subj "/CN=Cerebro Home Server CA"

    chmod 600 cerebro-ca.key
    ```

    - `cerebro-ca.key` → the CA's private key. Anyone who has it can create certificates your devices trust.
    - `cerebro-ca.crt` → the CA certificate, the file you install on devices.
    - Check it: `openssl x509 -in cerebro-ca.crt -noout -ext basicConstraints` must show `CA:TRUE`.

    **Back up the CA now.** If you lose the key, you must create a new CA and install it again on every device. Both files are tiny. Copy them to the HDD and also keep a copy outside the server (for example as an attachment in Vaultwarden, once it works):

    ```bash
    mkdir -p /srv/data/backups/ca && chmod 700 /srv/data/backups/ca
    cp -a /root/ca/cerebro-ca.key /root/ca/cerebro-ca.crt /srv/data/backups/ca/
    ```

2. Create a certificate for a service (repeat for each service, e.g. vault.lan, seafile.lan)

    ```bash
    cd /root/ca
    NAME=vault.lan

    openssl req -nodes -newkey rsa:2048 \
      -keyout $NAME.key \
      -out $NAME.csr \
      -subj "/CN=$NAME"

    openssl x509 -req -in $NAME.csr \
      -CA cerebro-ca.crt -CAkey cerebro-ca.key -CAcreateserial \
      -days 825 -sha256 \
      -extfile <(printf "subjectAltName=DNS:$NAME\nbasicConstraints=CA:FALSE\nkeyUsage=digitalSignature,keyEncipherment\nextendedKeyUsage=serverAuth") \
      -out $NAME.crt

    chmod 600 $NAME.key
    ```

    - `-days 825` → Apple devices reject server certificates valid for longer than 825 days. Renew before it expires (see [Renewing](#renewing-a-certificate)).
    - The command uses `<( ... )`, which only works in bash (the host's default shell).
    - Check the SAN: `openssl x509 -in $NAME.crt -noout -ext subjectAltName` should show `DNS:vault.lan`.
    - Check it was signed by your CA: `openssl verify -CAfile cerebro-ca.crt $NAME.crt` should answer `OK`.

3. Copy the files to the computer you use to open NPM's web page

    Run this **on your computer** (not on the host), in a folder you'll delete afterwards:

    ```bash
    scp root@192.168.1.10:/root/ca/vault.lan.crt root@192.168.1.10:/root/ca/vault.lan.key .
    scp root@192.168.1.10:/root/ca/cerebro-ca.crt .
    ```

    - Windows 10/11 has `scp` in PowerShell. Otherwise use WinSCP.
    - `vault.lan.key` is a private key: delete it from your computer once it's uploaded to NPM.

4. Upload to NPM: *SSL Certificates* > *Add SSL Certificate* > *Custom*
    - Name → `vault.lan (local CA)`, so it's not confused with an older one
    - Certificate Key → `<name>.key`
    - Certificate → `<name>.crt`
    - Intermediate Certificate → `cerebro-ca.crt`

5. Use it in the proxy host

    NPM keeps serving the old certificate until you change this: *Proxy Hosts* > *⋮* > *Edit* on the service > *SSL* tab > *SSL Certificate* > pick the new one > tick *Force SSL* > *Save*.

    When everything works, delete the old certificate from *SSL Certificates* (if this one replaces another, for example when renewing).

6. Install `cerebro-ca.crt` on each device (once)

    Get the file to the device first (USB cable, `scp`, or a file transfer you trust). Only the `.crt`; never the `.key`.

    - **Windows:** double-click the file > *Install Certificate* > *Local Machine* > *Place all certificates in the following store* > *Trusted Root Certification Authorities*
    - **Android:** *Settings* > *Security* > *Encryption & credentials* > *Install a certificate* > *CA certificate*
    - **iPhone/iPad:** open the file to install the profile, then *Settings* > *General* > *About* > *Certificate Trust Settings* > enable it
    - **Linux:** `cp cerebro-ca.crt /usr/local/share/ca-certificates/ && update-ca-certificates`. This only covers the system store (`curl`, `wget`). **Chrome on Linux keeps its own database** and ignores it, so also run, as your normal user (not `sudo`): `certutil -d sql:$HOME/.pki/nssdb -A -t "C,," -n "Cerebro Home Server CA" -i cerebro-ca.crt` (package `libnss3-tools`), check with `certutil -d sql:$HOME/.pki/nssdb -L`, then quit Chrome completely (`pkill -f chrome`). The *Import* button in `chrome://certificate-manager` is for client certificates and asks for a password, it did not work for the CA (2026-11-22)
    - **Firefox** (any OS) uses its own store: *Settings* > *Privacy & Security* > *Certificates* > *View Certificates* > *Authorities* > *Import*

    Restart the browser afterwards. If Chrome kept a saved bypass from an earlier warning ("you have chosen to turn off security warnings for this site"), click *Turn on warnings* in the address bar's site information and reload.

7. Test

    - From a device with the CA installed, open https://vault.lan: no warning, and the padlock's certificate details show *Cerebro Home Server CA* as the issuer.
    - From the command line: `curl --cacert cerebro-ca.crt https://vault.lan` must connect without errors.
    - Apps (Bitwarden, Seafile client): Android apps often ignore CAs installed by the user, so test them separately from the browser.

    > **Observation (2026-11-20): the Bitwarden Android app doesn't trust this CA.** With `cerebro-ca.crt` installed on the phone, https://vault.lan opens without warnings in the phone's browser, but the Bitwarden app, with *Self-hosted* server URL `https://vault.lan`, fails to log in saying it couldn't verify the server certificate. Likely cause: since Android 7, apps ignore CAs installed by the user unless they opt in, while browsers accept them. The app's *Import certificate* option is not a fix, because it is for client certificates. For now the phone uses the web vault in the browser. Real fix: a certificate from a public CA. Done on 2026-11-22, see [Public certificate with DuckDNS](#public-certificate-with-duckdns-in-use): the Bitwarden and Seafile Android apps work with it.

#### Renewing a certificate

A service certificate lasts 825 days (the CA, 10 years). Before it expires, repeat step 2 for the same `NAME` (it overwrites the old files), upload the new one in NPM (step 4), switch the proxy host to it (step 5) and delete the old one. Devices need nothing, since they trust the CA, not the certificate. Check the expiry date with `openssl x509 -in <name>.crt -noout -enddate`.

## Vaultwarden

### Prepare LXC

1. Create LXC container (step by step from [virtualization configuration](#virtualization)).

    General:
    - CT ID: 101
    - Hostname: vaultwarden
    - Unprivileged: true
    - Nesting: true
    - Password: <define>
    - SSH public key: empty

    Template:
    - Debian 12

    Disks:
    - Storage: local-lvm
    - Disk size: 8GB

    CPU:
    - Cores: 1

    Memory:
    - 512MB
    - Swap: 512MB

    Network:
    - Name: eth0
    - Bridge: vmbr0
    - IPv4: Static 192.168.1.12/24, Gateway 192.168.1.1
    - IPv6: None

    DNS:
    - DNS domain: lan
    - DNS servers: 192.168.1.11 1.1.1.1 (Pi-hole first, Cloudflare as fallback)

    Or simply run:
    ```bash
    pct create 101 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
    --hostname vaultwarden \
    --cores 1 \
    --memory 512 \
    --swap 512 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.12/24,gw=192.168.1.1,firewall=1,type=veth \
    --nameserver "192.168.1.11 1.1.1.1" \
    --searchdomain lan \
    --rootfs local-lvm:8 \
    --unprivileged 1 \
    --features keyctl=1,nesting=1 \
    --onboot 1 \
    --password
    ```

    - `--onboot 1` → starts on boot (the step "Start on boot" below does the same, for a container created in the GUI).

2. Enable keyctl (if the container was created in the GUI)

    ```bash
    pct set 101 --features keyctl=1,nesting=1
    ```

    - `keyctl=1,nesting=1` → required for Docker inside an unprivileged LXC

3. Create persistent data directory

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/vaultwarden/data
    chown -R 100000:100000 /srv/data/containers/vaultwarden
    ```

    - 100000 → the container's root user as seen by the host (unprivileged containers shift UIDs by 100000). Vaultwarden runs as root inside its Docker container.

4. Configure bind mount for persistent data

    ```bash
    pct set 101 -mp0 /srv/data/containers/vaultwarden/data,mp=/data/vaultwarden
    ```

    - /srv/data/containers/vaultwarden/data → /data/vaultwarden inside the container: database (`db.sqlite3`), keys (`rsa_key.pem`) and attachments
    - Docker itself (/data/docker: images, layers, cache) stays on the container disk (SSD). It can always be downloaded again.

5. Move logs to the HDD (before the first start)

    Follow [Logs on the HDD](#virtualization) with `ID=101; NAME=vaultwarden; MP=mp1`.

6. Start on boot

    ```bash
    pct set 101 --onboot 1
    ```

    Or in the GUI: *101 (vaultwarden)* > *Options* > *Start at boot* > Yes.

### Install service

1. Enter the container

    ```bash
    pct start 101
    pct enter 101
    ```

2. Update container

    ```bash
    apt update
    apt upgrade -y
    ```

    If you see `perl: warning: Setting locale failed` messages during installs, they are harmless. Optional fix: `apt install -y locales && dpkg-reconfigure locales` and choose `en_US.UTF-8`.

3. Install Docker from Docker's official repository (same as [NPM](#npm---nginx-proxy-manager))

    ```bash
    apt install -y ca-certificates curl
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    chmod a+r /etc/apt/keyrings/docker.asc

    echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" \
      > /etc/apt/sources.list.d/docker.list

    apt update
    apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
    ```

4. Set Docker's data folder to /data/docker

    ```bash
    systemctl stop docker docker.socket
    mkdir -p /data/docker
    nano /etc/docker/daemon.json
    ```

    Content:

    ```json
    {
      "data-root": "/data/docker",
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```

    ```bash
    systemctl start docker
    docker info | grep "Docker Root Dir"
    ```

    Should show `Docker Root Dir: /data/docker`.

5. Configure Vaultwarden

    ```bash
    mkdir -p /var/log/vaultwarden
    nano /data/docker-compose.yml
    ```

    Content:

    ```yaml
    services:
      vaultwarden:
        image: vaultwarden/server:latest
        container_name: vaultwarden
        restart: unless-stopped
        ports:
          - "8080:80"
        environment:
          DOMAIN: "https://vault.countess-lovelace.duckdns.org"
          SIGNUPS_ALLOWED: "true"
          LOG_FILE: "/var/log/vaultwarden/vaultwarden.log"
        volumes:
          - /data/vaultwarden:/data
          - /var/log/vaultwarden:/var/log/vaultwarden
    ```

    - `8080:80` → Vaultwarden listens on 80 inside its Docker container; NPM forwards to port 8080 of the LXC
    - DOMAIN → the address users open. Vaultwarden uses it to build links (attachments, WebAuthn/passkeys)
    - SIGNUPS_ALLOWED → `true` only until your accounts are created (see step 8)
    - LOG_FILE → logs go to /var/log/vaultwarden, on the HDD through the /var/log bind mount
    - /data/vaultwarden is the bind mount from Prepare LXC step 4 (HDD)

6. Rotate the log file

    Vaultwarden doesn't rotate its own log file, so let logrotate (already installed in the Debian template) do it:

    ```bash
    nano /etc/logrotate.d/vaultwarden
    ```

    Content:

    ```
    /var/log/vaultwarden/*.log {
        weekly
        rotate 4
        compress
        missingok
        notifempty
        copytruncate
    }
    ```

    - Keeps 4 weeks of compressed logs. `copytruncate` is needed because Vaultwarden keeps the file open.

7. Bring up:

    ```bash
    cd /data
    docker compose up -d
    ```

8. Create your accounts, then close sign-ups

    Configure NPM and Pi-hole first (sections below), then open https://vault.countess-lovelace.duckdns.org, click *Create account* and create your user account(s).

    Master password:
    - Use a passphrase you can remember (4-6 random words; the Bitwarden generator has a *Passphrase* type) and write it on paper kept safe at home.
    - Don't store it in another password manager (Google, browser, etc.): that account would become the real key to the vault.
    - It can't be recovered: the vault is encrypted with it. If it's forgotten, the vault is lost.

    Then block new accounts, so nobody else on the network can register:

    ```bash
    cd /data
    sed -i 's/SIGNUPS_ALLOWED: "true"/SIGNUPS_ALLOWED: "false"/' docker-compose.yml
    docker compose up -d
    ```

    - `docker compose up -d` recreates the container with the new setting. Data is safe because it lives in /data/vaultwarden (HDD).
    - Existing accounts keep working. To add a user later, set it back to `"true"` temporarily.

### Configure Vaultwarden on NPM

#### SSL certificate

No certificate is created for Vaultwarden: the proxy host uses the wildcard certificate (`*.countess-lovelace.duckdns.org`), see [Certificates](#certificates-https).

#### Configure proxy host

In the NPM dashboard go to `Proxy Hosts → Add Proxy Host`.

"Details" Tab

- Domain Names: vault.countess-lovelace.duckdns.org
- Scheme: http
- Forward Hostname / IP: 192.168.1.12
- Forward Port: 8080
- Cache Assets: false
- Block Common Exploits: true
- Websockets Support: true

"SSL" Tab
- SSL Certificate: the wildcard certificate (`*.countess-lovelace.duckdns.org`)
- Force SSL: true
- HTTP/2 Support: true

### Configure DNS record in Pi-hole

Access the Pi-hole interface and go to *Settings* > *Local DNS Records*. Add:
- Domain: vault.countess-lovelace.duckdns.org
- IP Address: 192.168.1.13 (NPM, not Vaultwarden)

### Verify

Inside LXC 101:

```bash
cd /data
docker compose ps
```

Should show the `vaultwarden` container as `Up` (after about a minute, `(healthy)`).

```bash
tail /var/log/vaultwarden/vaultwarden.log
```

Should end with `Rocket has launched from http://0.0.0.0:80`.

In the browser, test:
> https://vault.countess-lovelace.duckdns.org

* Vaultwarden web interface should open

Notes:
* `http://192.168.1.12:8080` still answers, but the web vault doesn't work there: browsers only allow the encryption functions (WebCrypto) it needs over HTTPS. Always use https://vault.countess-lovelace.duckdns.org.

## Seafile

HTTPS: use the wildcard certificate in its proxy host (`seafile.countess-lovelace.duckdns.org`) following [Certificates](#certificates-https) (*For every service*), and follow [Logs on the HDD](#virtualization) from the start.

### Prepare LXC

1. Create the LXC container (step by step from [virtualization configuration](#virtualization), steps 1 to 3).

    General:
    - CT ID: 103
    - Hostname: seafile
    - Unprivileged: true
    - Nesting: true
    - Password: <define>
    - SSH public key: empty

    Template:
    - Debian 12

    Disks:
    - Storage: local-lvm
    - Disk size: 10GB (OS and Docker images only; user files go to the HDD)

    CPU:
    - Cores: 2

    Memory:
    - 2048MB (Seafile runs as several Docker containers: Seafile, MariaDB and a cache)
    - Swap: 1024MB

    Network:
    - Name: eth0
    - Bridge: vmbr0
    - IPv4: Static 192.168.1.14/24, Gateway 192.168.1.1
    - IPv6: None

    DNS:
    - DNS domain: lan
    - DNS servers: 192.168.1.11 1.1.1.1

    Or simply run:
    ```bash
    pct create 103 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
    --hostname seafile \
    --cores 2 \
    --memory 2048 \
    --swap 1024 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.14/24,gw=192.168.1.1,firewall=1,type=veth \
    --nameserver "192.168.1.11 1.1.1.1" \
    --searchdomain lan \
    --rootfs local-lvm:10 \
    --unprivileged 1 \
    --features keyctl=1,nesting=1 \
    --onboot 1 \
    --password
    ```

    - `keyctl=1,nesting=1` → required for Docker inside an unprivileged LXC. No extra `lxc.*` settings are needed (tested on NPM, see [TODOs](#todos)).
    - `--onboot 1` → starts on boot.
    - **Don't start it yet.** Logs go to the HDD first (next step).
    - Check `pct config 103` before going on: a wrong `ip=` (for example another container's address) would conflict on the first start. Fix it with `pct set 103 --net0 ...` (keep the same `hwaddr`).

2. Move logs to the HDD (before the first start)

    Follow [Logs on the HDD](#virtualization) with `ID=103; NAME=seafile; MP=mp0`.

    - The standard procedure says "next free mount point". The container had none yet, so logs take `mp0` and data takes `mp1` (the reverse of Vaultwarden and NPM, where data was added first). The number is only a label and doesn't change how it works.

3. Create persistent data directory and bind mount

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/seafile/data
    chown 100000:100000 /srv/data/containers/seafile/data
    pct set 103 -mp1 /srv/data/containers/seafile/data,mp=/opt/seafile-data
    ```

    - /srv/data/containers/seafile/data → /opt/seafile-data inside the container: Seafile's files, libraries and configuration (the database has its own mount, step 4). Docker itself (images, layers) stays on the container disk (SSD).
    - 100000 → the container's root user as seen by the host (unprivileged containers shift UIDs by 100000). Keep this ownership.
    - `/opt/seafile-data` is the path Seafile's own Docker documentation uses.

4. Create the database directory and bind mount

    Seafile's compose files keep MariaDB's data in a separate path (`/opt/seafile-mysql/db`), outside `/opt/seafile-data`. Without a mount there it would end up on the container disk (SSD). It holds the libraries' metadata and users, so it goes to the HDD too:

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/seafile/mysql
    chown 100000:100000 /srv/data/containers/seafile/mysql
    pct set 103 -mp2 /srv/data/containers/seafile/mysql,mp=/opt/seafile-mysql
    ```

    - /srv/data/containers/seafile/mysql → /opt/seafile-mysql inside the container (the compose file stores the DB in `/opt/seafile-mysql/db`).
    - After the first start MariaDB changes the owner of its own files to its `mysql` user (UID 999 in the image), which the host sees as `100999`. That's expected, don't "fix" it.
    - Both folders must be included in the manual backups (vzdump doesn't include bind mounts).

5. Verify the container config

    ```bash
    pct config 103
    ```

    Expect `mp0` (logs → /var/log), `mp1` (data → /opt/seafile-data), `mp2` (database → /opt/seafile-mysql), `onboot: 1`, `unprivileged: 1`, `features: keyctl=1,nesting=1`, `ip=192.168.1.14/24`, `nameserver: 192.168.1.11 1.1.1.1`, `searchdomain: lan`.

### Install service

Same first steps as [Vaultwarden](#install-service) (enter the container, update, install Docker from Docker's repository, set `data-root` to `/data/docker` with capped logs). Docker's images (Seafile's alone is ~2.5GB) stay on the container disk (SSD), hence its 10GB.

1. Download Seafile's compose files (official Docker setup for 13.0, https://manual.seafile.com/latest/setup/setup_ce_by_docker/)

    ```bash
    mkdir -p /opt/seafile && cd /opt/seafile
    wget -O .env https://manual.seafile.com/13.0/repo/docker/ce/env
    wget https://manual.seafile.com/13.0/repo/docker/ce/seafile-server.yml
    ```

    - The official guide also downloads `caddy.yml` and `seadoc.yml`. Neither is used here: NPM already does HTTPS (so no Caddy), and SeaDoc (collaborative documents) is off for now.

2. Edit `/opt/seafile/.env`

    | Variable | Value | Why |
    | :---- | :---- | :---- |
    | `COMPOSE_FILE` | `'seafile-server.yml'` | Only the server file. Without `caddy.yml` and `seadoc.yml` |
    | `SEAFILE_SERVER_HOSTNAME` | `seafile.countess-lovelace.duckdns.org` | Seafile builds its public URLs from it (it was `seafile.lan` until 2026-11-22) |
    | `SEAFILE_SERVER_PROTOCOL` | `https` | TLS ends at NPM, but the links Seafile gives out must be https |
    | `TIME_ZONE` | `Europe/Lisbon` | |
    | `ENABLE_SEADOC` | `false` | SeaDoc not used for now |
    | `JWT_PRIVATE_KEY` | random string, one line, 32+ characters | `openssl rand -hex 24` |
    | `SEAFILE_MYSQL_DB_PASSWORD` | strong password | Seafile's database user |
    | `INIT_SEAFILE_MYSQL_ROOT_PASSWORD` | strong password | Only used on the first start |
    | `INIT_SEAFILE_ADMIN_EMAIL` / `INIT_SEAFILE_ADMIN_PASSWORD` | the admin account | Only used on the first start |

    - Volumes keep the defaults: `/opt/seafile-data` (Seafile data, `mp1`) and `/opt/seafile-mysql/db` (MariaDB, `mp2`), both on the HDD.
    - `JWT_PRIVATE_KEY` must be a plain random string on **one line**. (An SSH private key pasted by mistake breaks the file: `docker compose` fails with "unexpected character in variable name".)
    - Keep the passwords in Vaultwarden. They are not in this guide.
    - `REDIS_PASSWORD` stays empty: Redis has no published port and is only reachable inside Seafile's Docker network.

3. Edit `/opt/seafile/seafile-server.yml`

    In the `seafile:` service, enable the port mapping (the file has it commented):

    ```yaml
        ports:
          - "80:80"
    ```

    - Keep the same indentation as `image:` and `volumes:` (4 spaces for the key, 6 for the list item). An extra space makes the file invalid (`mapping values are not allowed in this context`).
    - Delete the `labels:` block that mentions `caddy`, if the file has one.
    - Check the file and the `.env` without printing the secrets: `docker compose config -q` (it prints nothing when everything is fine).
    - Port 80 is the one NPM forwards to.

4. Back up before the first start

    The stack was empty, so this was a rollback point and a rehearsal, as in [Maintenance](#maintenance-updates): `vzdump 103 --storage data-hdd --mode snapshot --compress zstd` (~350MB) plus a copy of `/srv/data/containers/seafile/{data,mysql}`. Repeat it **before uploading real data**.

5. Bring up

    ```bash
    cd /opt/seafile
    docker compose up -d
    docker compose logs -f seafile
    ```

    - The first start downloads the images and creates the databases and the admin account (a few minutes). Wait for `Seahub is started`.
    - Check: `docker ps` shows `seafile`, `seafile-mysql` (both `healthy`) and `seafile-redis`; `curl -I http://192.168.1.14/` answers `302` to `/accounts/login/`.
    - The three containers use `restart: unless-stopped`, and the LXC has `onboot`, so they come back after a reboot.
    - After the first start the database's files belong to `100999` on the host (MariaDB's `mysql` user, UID 999 + 100000). That's expected.
    - To add an admin later: `docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh`

### Configure Seafile on NPM

#### SSL certificate

No certificate is created for Seafile: the proxy host uses the wildcard certificate (`*.countess-lovelace.duckdns.org`), see [Certificates](#certificates-https).

#### Configure proxy host

If an older proxy host for the same name exists, delete it first.

*Proxy Hosts* > *Add Proxy Host*:

"Details" Tab

- Domain Names: seafile.countess-lovelace.duckdns.org
- Scheme: http
- Forward Hostname / IP: 192.168.1.14
- Forward Port: 80
- Cache Assets: false
- Block Common Exploits: true
- Websockets Support: true

"SSL" Tab

- SSL Certificate: the wildcard certificate (`*.countess-lovelace.duckdns.org`)
- Force SSL: true
- HTTP/2 Support: true

Notes:
* No custom Nginx settings are needed for uploads. NPM's Nginx already allows request bodies up to 2000MB (`client_max_body_size` in `nginx.conf`), and Seafile uploads files in chunks.

### Configure DNS record in Pi-hole

`seafile.countess-lovelace.duckdns.org` → `192.168.1.13` (NPM, not the container), see [DNS (Pi-hole)](#dns-pi-hole).

### Verify

* From the host: `curl https://seafile.countess-lovelace.duckdns.org/api2/ping/` (no `--cacert` needed) answers `"pong"`, and `.../seafhttp/protocol-version` answers `{"version": 2}` (the file server path, used by uploads and downloads).
* From any device: https://seafile.countess-lovelace.duckdns.org opens without warnings and the admin account can log in.
* Create a library, upload and download a file, and create a download link. The link must start with `https://seafile.countess-lovelace.duckdns.org/seafhttp/`.
* Inside the container: name resolution uses Pi-hole (`getent hosts vault.countess-lovelace.duckdns.org` → `192.168.1.13`), see the DNS tab in [Virtualization](#virtualization).

Notes:
* **Name resolution inside the container:** `seafile.countess-lovelace.duckdns.org` resolves through Pi-hole to NPM (`192.168.1.13`), and NPM's certificate is public, so no CA is needed inside Docker. In Seafile 13 `SERVICE_URL` and `FILE_SERVER_ROOT` are built from `SEAFILE_SERVER_PROTOCOL` and `SEAFILE_SERVER_HOSTNAME` (`seahub/settings.py`, not in the web settings page or `seahub_settings.py`) and are only given to clients. Seafile's own internal calls use `http://127.0.0.1`. (With the old `.lan` name, Proxmox's `/etc/hosts` entry made `seafile.lan` resolve to the container itself, which didn't matter.)
* **Android client:** tested on 2026-11-22 with the public certificate and it works (it rejected the local CA).
* **Backups:** nothing is scheduled. Back up `/srv/data/containers/seafile/{data,mysql}` (vzdump doesn't include them) together with a vzdump of 103. The database (`mysql`) holds the libraries' metadata, so the two folders must be copied together, taken while Seafile is stopped (`docker compose stop` in `/opt/seafile`) for a consistent copy.
* **Logs:** the system logs are in `/srv/data/containers/seafile/logs` (the container's `/var/log`), Seafile's own logs in `/srv/data/containers/seafile/data/seafile/logs`, and Docker's logs are capped at 3 × 10MB.

## Maintenance (updates)

Updates are done by hand for now, so it's clear what changes. A rough cadence is once a month, or right away when a security fix is announced. Updates are one of the few things that can break a working service, so always follow the same order: **check → back up → update → test**.

### 1. Check what's pending

```bash
# Host:
apt update && apt list --upgradable

# Each LXC (100, 101, 102, 103):
pct exec <id> -- bash -c 'apt update && apt list --upgradable'

# Pi-hole's own components (core, web, FTL):
pct exec 100 -- pihole -v
```

`pihole -v` shows `(Latest: ...)` next to each version. Pi-hole is **not** updated by apt, so `apt list --upgradable` never shows it.

### 2. Back up first

```bash
vzdump <id> --storage data-hdd --mode snapshot --compress zstd
```

- vzdump **doesn't include bind mounts**, so also copy the service's folder: `cp -a /srv/data/containers/<service>/data /srv/data/backups/<service>-pre-update`
- Seafile (103): copy **both** `data` and `mysql` (`/srv/data/containers/seafile/{data,mysql}`), with the stack stopped (`pct exec 103 -- bash -c 'cd /opt/seafile && docker compose stop'`), so the files and the database stay consistent. Start it again afterwards with `docker compose up -d`.
- `pct snapshot` doesn't work on these containers ("snapshot feature is not available"), because of the bind mounts. vzdump plus the folder copy is the rollback point.

### 3. Update

| What | Command | Then |
| :---- | :---- | :---- |
| Host | `apt full-upgrade` | Reboot if a new kernel was installed (see below) |
| Debian packages in an LXC | `pct exec <id> -- bash -c 'apt update && apt full-upgrade -y'` | Restart the container if libc, systemd or openssl changed: `pct reboot <id>` |
| Pi-hole (core, web, FTL) | `pct exec 100 -- pihole -up` | Test DNS (below) |
| Vaultwarden / NPM (Docker) | `pct exec <id> -- bash -c 'cd /data && docker compose pull && docker compose up -d'` | Open the web interface |
| Seafile (Docker) | `pct exec 103 -- bash -c 'cd /opt/seafile && docker compose pull && docker compose up -d'` | Check `docker ps` and open https://seafile.countess-lovelace.duckdns.org |

Notes:

- Both compose files use the `:latest` tag, so `docker compose pull` fetches the newest release. If a new version breaks something, the data is safe on the HDD, but going back means pinning the previous version tag in `docker-compose.yml`.
- Seafile's images are set in `/opt/seafile/.env`: `seafile-mc:13.0-latest` (only moves within 13.0, so `pull` never jumps to a new major version), `mariadb:10.11` and `redis`. Moving to a new Seafile version (e.g. 14.0) means changing `SEAFILE_IMAGE` and the compose files, following Seafile's upgrade notes: do that on purpose, with a backup, not as part of the routine. Seafile may run database migrations on its first start after an update, so watch `docker compose logs -f seafile` until it is up.
- Old images pile up on the container disk. Clean them with `docker image prune -f` inside the container.
- Update **one guest at a time**, and **Pi-hole last or at a quiet time**: while it restarts, no device on the LAN can resolve names.

### 4. Test

```bash
# Pi-hole answers DNS and is blocking:
pct exec 100 -- pihole status
dig +short @192.168.1.11 example.com
```

- https://vault.countess-lovelace.duckdns.org opens and you can log in
- NPM's admin page opens (http://192.168.1.13:81) and the proxy hosts still work
- https://seafile.countess-lovelace.duckdns.org opens and you can log in; `pct exec 103 -- docker ps` shows `seafile` and `seafile-mysql` as `healthy`

### Host reboot

A kernel update only takes effect after a reboot (`uname -r` shows the running one, `proxmox-boot-tool kernel list` shows the installed ones).

1. Check that no VM is running (`qm list`) and tell whoever uses the LAN: DNS and passwords are down for a few minutes.
2. `reboot`
3. After 2-3 minutes: `uname -r`, `pct list` (100, 101, 102 have `onboot`, so they must be running), `systemctl --failed`.
4. If the new kernel doesn't work, pick the previous one in the boot menu and pin it: `proxmox-boot-tool kernel pin <version>`.

### Automating (later)

`unattended-upgrades` inside the containers can install Debian security updates by itself. It's left for later, after the manual routine feels familiar and backups exist.

## TODOs

### Host

- [x] **Fix the Proxmox repository on the host** (2026-11-17). `pve-no-subscription.list` pointed to `bookworm` (Proxmox 8) while the host runs Proxmox 9 (`trixie`), so Proxmox packages were never updated. Replaced by `/etc/apt/sources.list.d/proxmox.sources` (trixie), then `apt full-upgrade` (9.1 → 9.2).
- [x] Reboot the host to load the new kernel installed by the upgrade (6.17.2 → 7.0.14, the new default in Proxmox 9.2) (2026-11-18). Booted fine on 7.0.14-29-pve, all containers came back. If something breaks, pick `6.17.13` in the boot menu and pin it with `proxmox-boot-tool kernel pin 6.17.13-21-pve`.
- [x] Set NPM (102) to start on boot (2026-11-17). Before, it stayed stopped after every host reboot.
- [x] Add `nofail` to the `/srv/data` line in the host's `/etc/fstab` (2026-11-17).
- [x] Run `pvesm set data-hdd --is_mountpoint yes` on the host (2026-11-17).
- [x] Clean up leftovers on the HDD from earlier attempts (2026-11-17): `/srv/data/containers/npm/` (~19GB old Docker data), `/srv/data/npm/`, `/srv/data/containers/seafile/`.
- [x] Move NPM data to the HDD, `/srv/data/containers/npm/{data,letsencrypt}` with bind mounts (2026-11-17).
- [x] Delete the pre-migration copies inside LXC 102 (`/data/npm.pre-hdd`, `/data/letsencrypt.pre-hdd`) after confirming NPM works (2026-11-17).

### Services

- [x] Update Pi-hole (2026-11-18): core v6.3 → v6.6, web v6.4 → v6.6.1, FTL v6.4.1 → v6.9, with `pct exec 100 -- pihole -up`.
- [x] Update the Debian packages inside the Pi-hole container (100) (2026-11-19): ~70 packages, with `pct exec 100 -- bash -c 'apt update && apt full-upgrade'`. 101 and 102 had nothing pending.
- [x] Rebuild Vaultwarden (101) from scratch (2026-11-17): unprivileged, Docker from Docker's repository with compose, `DOMAIN` set, data and logs on the HDD, starts on boot. The old one was privileged and only held test data.
- [x] Vaultwarden: close sign-ups (`SIGNUPS_ALLOWED: "false"`) after creating the account (2026-11-17).
- [x] Delete the old Vaultwarden test data, `/srv/data/containers/vaultwarden.old` (2026-11-17).
- [x] Vaultwarden: enable two-step login (*Settings → Security → Two-step login*) with an authenticator app, and keep the recovery code on paper.
- [x] Test whether NPM (102) works without `lxc.apparmor.profile: unconfined`, `lxc.cgroup2.devices.allow: a` and `lxc.mount.auto: proc:rw sys:rw` (2026-11-20). It does: the three lines were commented out one at a time (full `pct stop` and `pct start` after each) and Docker, NPM and the proxy hosts kept working. 102 is now confined like the other containers (`/sys` read-only).
- [x] Migrate certificates to the local CA (2026-11-17). Replaced on 2026-11-22 by the public certificate, see [Certificates](#certificates-https).
- [x] Point the DNS of the containers and the host to Pi-hole (2026-11-20). 101 and 102 use `192.168.1.11` then `1.1.1.1`, Pi-hole (100) uses only `1.1.1.1`, and the host uses `192.168.1.11` then `1.1.1.1` (`pvesh set /nodes/pve/dns ...`). Before, they all used the router (`192.168.1.1`), which doesn't know `.lan` and sometimes times out (NPM logged 273 DNS timeouts since 2026-10-22).
- [x] Seafile: set up from scratch (2026-11-20). LXC 103 at 192.168.1.14, Seafile 13.0 in Docker, data, database and logs on the HDD, HTTPS at https://seafile.lan with the local CA. See [Seafile](#seafile). Moved to https://seafile.countess-lovelace.duckdns.org on 2026-11-22.
- [x] Test the Bitwarden and Seafile Android clients (2026-11-22): both work with the public certificate. They rejected the local CA, so the earlier TODO to test them with it was dropped.
- [ ] Seafile: test the desktop client with the public certificate and set up the clients' libraries.
- [x] Seafile: add it to the update routine in [Maintenance](#maintenance-updates) (2026-11-22): check, back up (`data` + `mysql` with the stack stopped), `docker compose pull` + `up -d` in `/opt/seafile`, and test.

- [ ] Set up Immich (photos): a new LXC (104, next free IP after Seafile's 192.168.1.14 outside the DHCP pool), data and logs on the HDD (`/srv/data/containers/immich/{data,logs}`), behind NPM with HTTPS, following the [new service checklist](#virtualization) (Certificates and Logs sections). Check the RAM first (15.5 GB host, VMs 200 and 201 need ~19 GB), and if it needs machine learning or GPU access, plan the CPU/RAM for it. Add it to the update routine in [Maintenance](#maintenance-updates) and to the backups.

### External access and certificates

- [x] **Step 1, public certificates with DuckDNS** (done on 2026-11-22 for Vaultwarden and Seafile, see [Certificates](#certificates-https)): wildcard certificate `*.countess-lovelace.duckdns.org` in NPM, Pi-hole local records and proxy hosts for `vault.countess-lovelace.duckdns.org` and `seafile.countess-lovelace.duckdns.org`, `DOMAIN` and `SEAFILE_SERVER_HOSTNAME` updated. The local CA stays as a fallback:
  - [x] Local CA removed from the phone (2026-11-22).
  - [x] `.lan` kept as a fallback instead of retired (2026-11-22): Pi-hole records and NPM proxy hosts recreated, local CA certificates kept in NPM and in `/root/ca`, CA to be used only on the main computer (check that it is installed there). See [Local CA](#local-ca-fallback-browser-only).
  - [x] Seafile and Vaultwarden login through `.lan` in the browser of the main computer (2026-11-22): works, after installing the CA in Chrome's own database, see step 6 of the local CA.
- [ ] **Step 2, external access:** decide whether and how, see the options in [External Access](#external-access). The preferred direction is a VPN (WireGuard or Tailscale), which needs no domain and exposes nothing. Only if a public option (Cloudflare Tunnel or port forwarding) is chosen: decide which services to expose (likely Vaultwarden and Seafile; Pi-hole and the Proxmox panel stay internal), add protection (Cloudflare Access, 2FA) and have working backups first. Cloudflare Tunnel also needs your own domain.

### Logs

- [x] Logs on the HDD for Pi-hole, Vaultwarden and NPM (2026-11-18): each container's /var/log is a bind mount of `/srv/data/containers/<service>/logs`, app logs moved under /var/log, Docker logs capped at 3 × 10MB. Pi-hole's folder was reorganized into `pihole/data` and `pihole/logs`.

### Backups

- [ ] Implement backups (the manual routine is in [Maintenance](#maintenance-updates)). Nothing is scheduled today. The only backups are manual ones of 100, 101 and 102 taken on 2026-11-17 in `/srv/data/dump` (before the Proxmox upgrade).
  - Note: LXC snapshots (`pct snapshot`) don't work on containers with these bind mounts ("snapshot feature is not available", seen on 100 on 2026-11-18), so vzdump plus a copy of the bind-mounted folders is the way to get a rollback point.
  - Note: Proxmox backups (vzdump) **don't include bind mounts**. Pi-hole's `/etc/pihole` and Vaultwarden's `/data` (in `/srv/data/containers/`) must be backed up separately.
  - Include the local CA (`/root/ca`, at least `cerebro-ca.key` and `cerebro-ca.crt`), kept as an option for internal-only services. Losing the key means installing a new CA on every device that uses it. Also keep NPM's data folder, it holds the DuckDNS token and the certificates.
  - Proxmox backup job for the LXCs (Datacenter > Backup) to data-hdd
  - Retention policy (data-hdd currently keeps all backups forever: `prune-backups keep-all=1`)
  - Copy to a second disk or off-site (data-hdd is on the same server, so it doesn't protect against losing the server)
  - Test a restore