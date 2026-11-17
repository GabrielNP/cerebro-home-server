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
* |__[External Access (Cloudflare)](#external-access-cloudflare)
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
* [Certificates (internal HTTPS)](#certificates-internal-https)
* [Vaultwarden](#vaultwarden)
* [Seafile](#seafile)
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
* Standard structure: /srv/data/containers/<service>
* Bind-mount from host to LXC (usually mounted as /data inside the container)

#### Logs

- **Premise:** logs should go to sdb (1TB HDD), avoiding overloading sda (240GB SSD).
- Standardize this by sending each service's logs to its folder under /srv/data/containers/<name>.
- Pi-hole, Vaultwarden and NPM don't follow this yet (see [TODOs](#todos)). Every new service (Seafile, Immich, ...) must follow it from the start.

### IP Addressing

- Proxmox Host (pve): 192.168.1.10
- Pi-hole: 192.168.1.11
- Vaultwarden: 192.168.1.12
- Nginx Proxy Manager: 192.168.1.13
- Seafile: 192.168.1.14 (planned)
- IPs defined outside the router's DHCP range to avoid conflicts

### DNS (Pi-hole)

- Pi-hole configured as primary DNS on the network
- Local records:
  vault.lan -> 192.168.1.13
  seafile.lan -> 192.168.1.13
- Reason: all access goes through the reverse proxy

### Reverse Proxy (Nginx Proxy Manager)

- Running in a dedicated LXC
- Ports: 80/443 (proxy), 81 (admin panel)
- Responsible for all internal HTTPS
- Self-signed internal certificates (OpenSSL), see [Certificates](#certificates-internal-https)
- Let's Encrypt NOT used for .lan domains

### External Access (Cloudflare)

- Not yet configured. To be done in the future.

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

* Leave default (use host)

  Click Next.

Confirm Tab

* Check everything
* Uncheck "start after created"

  Click Finish.

3) Verification

The container assumes a CT ID and appears as Stopped.

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
    - Default

    Or simply run:
    ```bash
    pct create 100 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
    --hostname pihole \
    --cores 1 \
    --memory 512 \
    --swap 512 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.11/24,gw=192.168.1.1,firewall=1,type=veth \
    --rootfs local-lvm:8 \
    --unprivileged 1 \
    --features nesting=1 \
    --password
    ```

2. Start on boot

    Pi-hole is the network DNS: if it's down, name resolution on the LAN is down.

    ```bash
    pct set 100 --onboot 1
    ```

    Or in the GUI: *100 (pihole)* > *Options* > *Start at boot* > Yes.

3. Create persistent data directory (before installing Pi-hole)

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/pihole
    chown 100000:100000 /srv/data/containers/pihole
    ```

    - 100000 → the container's root user as seen by the host (unprivileged containers shift UIDs by 100000)
    - The installer needs to write into this folder

4. Configure bind mount for persistent data

    ```bash
    pct set 100 -mp0 /srv/data/containers/pihole,mp=/etc/pihole
    ```

    - /srv/data/containers/pihole → host (HDD)
    - /etc/pihole → inside the container: Pi-hole config (`pihole.toml`), blocklists (`gravity.db`) and the long-term query database (`pihole-FTL.db`, which grows over time, so it's good that it's on the HDD)

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
    | vault.lan | 192.168.1.13 | Vaultwarden, through NPM |
    | seafile.lan | 192.168.1.13 | Seafile (planned), through NPM |

    All records point to NPM, never directly to the service.

    Or from the container shell:

    ```bash
    pihole-FTL --config dns.hosts '["192.168.1.13 vault.lan","192.168.1.13 seafile.lan"]'
    ```

4. Router: make devices use Pi-hole

    In the router's DHCP/LAN settings, set the DNS server handed out to devices to `192.168.1.11`.
    Without this step, devices keep using the router as DNS and nothing is filtered, and `*.lan` names don't resolve.
    Devices pick up the new DNS when they renew their DHCP lease (reconnect to the network).

### Verify

From the Proxmox host or any device on the network:

```bash
dig +short @192.168.1.11 vault.lan
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
    - Default

2. Ensure container configuration is

    ```bash
    nano /etc/pve/lxc/102.conf
    ```

    Content (keep the lines Proxmox generated, make sure these are present):
    ```ini
    features: keyctl=1,nesting=1
    unprivileged: 1

    lxc.apparmor.profile: unconfined
    lxc.cgroup2.devices.allow: a
    lxc.mount.auto: proc:rw sys:rw
    ```

    - `keyctl=1,nesting=1` → required for Docker inside an unprivileged LXC
    - The last 3 lines remove most of the container's isolation. They were added during troubleshooting and may not be needed (see [TODOs](#todos))

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

    - /srv/data/containers/npm/data → /data/npm inside the container: NPM config, database, proxy hosts, uploaded certificates and logs
    - /srv/data/containers/npm/letsencrypt → /data/letsencrypt inside the container: Let's Encrypt certificates (not used for .lan)
    - Docker itself (/data/docker: images, layers, cache) stays on the container disk (SSD). It can always be downloaded again.

5. Start on boot

    NPM is the single entry point for every service (`*.lan` → 192.168.1.13): if it's down, no service is reachable by name.

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
      "data-root": "/data/docker"
    }
    ```

    ```bash
    systemctl start docker
    docker info | grep "Docker Root Dir"
    ```

    Should show `Docker Root Dir: /data/docker`.

    - /data/docker is a normal folder inside the container disk (SSD), not a bind mount. Docker images, layers and cache are stored there (~1GB).

5. Configure NPM

    ```bash
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
    ```

    - /data/npm and /data/letsencrypt are the bind mounts from step 4 (HDD)

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

## Certificates (internal HTTPS)

`.lan` is not a public domain, so Let's Encrypt can't issue certificates for it. Internal HTTPS uses certificates we create ourselves with OpenSSL and upload to NPM as *Custom* certificates.

### Current method (in use)

One self-signed certificate per service, valid for 10 years. Generated inside NPM's Docker container, so the files end up in `/data/npm/certs/<service>/` inside LXC 102:

```bash
# On LXC 102:
docker exec -it nginx-proxy-manager bash

# Inside the NPM docker container:
mkdir -p /data/certs/vault
cd /data/certs/vault

openssl req -x509 -nodes -days 3650 \
  -newkey rsa:2048 \
  -keyout vault.lan.key \
  -out vault.lan.crt \
  -subj "/CN=vault.lan"
```

Then copy the two files to your computer, for example from that computer:

```bash
ssh root@192.168.1.10 "pct exec 102 -- cat /data/npm/certs/vault/vault.lan.crt" > vault.lan.crt
ssh root@192.168.1.10 "pct exec 102 -- cat /data/npm/certs/vault/vault.lan.key" > vault.lan.key
```

and upload them in NPM: *SSL Certificates* > *Add SSL Certificate* > *Custom* (see the Vaultwarden section).

**Limitations of this method**

- The certificate has only a Common Name (CN), no *Subject Alternative Name* (SAN). Modern browsers (Chrome, Edge, Safari) and Android ignore the CN and reject the certificate even if you trust it manually. Only "accept the risk and continue" works.
- Self-signed certificates are rejected by apps: the Bitwarden mobile/desktop apps (Vaultwarden) and the Seafile sync clients will refuse to connect.
- Every service has its own certificate, so every device has to trust every certificate one by one.

### Recommended method: local Certificate Authority (CA)

Create your own small CA once, install it on each device once, and sign one certificate per service with it. Every device then trusts every service, now and in the future.

1. Create the CA (once, on the Proxmox host)

    ```bash
    mkdir -p /root/ca && chmod 700 /root/ca
    cd /root/ca

    openssl req -x509 -nodes -newkey rsa:4096 -days 3650 \
      -keyout cerebro-ca.key \
      -out cerebro-ca.crt \
      -subj "/CN=Cerebro Home CA"

    chmod 600 cerebro-ca.key
    ```

    - `cerebro-ca.key` → the CA's private key. Anyone who has it can create certificates your devices trust. Never copy it to other machines; back it up somewhere safe.
    - `cerebro-ca.crt` → the CA certificate, the file you install on devices.

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
    ```

    - `-days 825` → Apple devices reject server certificates valid for longer than 825 days. Renew before it expires.
    - Check the result: `openssl x509 -in $NAME.crt -noout -ext subjectAltName` should show `DNS:vault.lan`.

3. Upload to NPM: *SSL Certificates* > *Add SSL Certificate* > *Custom*
    - Certificate Key → `<name>.key`
    - Certificate → `<name>.crt`
    - Intermediate Certificate → `cerebro-ca.crt`

4. Install `cerebro-ca.crt` on each device (once)
    - **Windows:** double-click the file > *Install Certificate* > *Local Machine* > *Place all certificates in the following store* > *Trusted Root Certification Authorities*
    - **Android:** *Settings* > *Security* > *Encryption & credentials* > *Install a certificate* > *CA certificate*
    - **iPhone/iPad:** open the file to install the profile, then *Settings* > *General* > *About* > *Certificate Trust Settings* > enable it
    - **Linux:** `cp cerebro-ca.crt /usr/local/share/ca-certificates/ && update-ca-certificates`
    - **Firefox** (any OS) uses its own store: *Settings* > *Privacy & Security* > *Certificates* > *View Certificates* > *Authorities* > *Import*

### Alternative for the future

When Cloudflare is configured with a real domain (e.g. `vault.mydomain.com`), NPM can get free Let's Encrypt certificates using the *DNS challenge* with a Cloudflare API token, even for services that are only reachable inside the LAN. Then no device needs a CA installed.

## Vaultwarden

> This container is currently **privileged** (`Unprivileged: false`). It works, but see [TODOs](#todos) about converting it to unprivileged before exposing it to the internet.

### Prepare LXC
 1. Create LXC container (step by step from [virtualization configuration](#virtualization)).

    General:
    - CT ID: 101
    - Hostname: vaultwarden
    - Unprivileged: false
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
    - Default

    Or simply run:
    ```bash
    pct create 101 local:vztmpl/debian-12-standard_12.12-1_amd64.tar.zst \
    --hostname vaultwarden \
    --cores 1 \
    --memory 512 \
    --swap 512 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.12/24,gw=192.168.1.1,type=veth \
    --rootfs local-lvm:8 \
    --unprivileged 0 \
    --features nesting=1 \
    --password
    ```

 2. Create persistent data directory

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/vaultwarden
    ```

    - Owner stays `root:root`. The container is privileged, so its root user is the host's root and needs no UID shifting.

 3. Configure bind mount for persistent data

    ```bash
    pct set 101 -mp0 /srv/data/containers/vaultwarden,mp=/data
    ```

    - /srv/data/containers/vaultwarden → host (HDD)
    - /data → inside the container (Vaultwarden will use this directory for data)

 4. Start the container and check the mount

    ```bash
    pct start 101
    pct enter 101
    ls -l /data
    ```

    The directory should exist and you should have no permission errors.

### Install service

1. Update container (inside the container)

    ```bash
    apt update
    apt upgrade -y
    ```

2. Install dependencies

    ```bash
    apt install -y curl docker.io
    systemctl enable --now docker
    ```

    - `docker.io` is Debian's own Docker package. It's enough here because Vaultwarden runs with a single `docker run` command (no compose).

    If something like this appears

    ```plaintext
    perl: warning: Setting locale failed.
    perl: warning: Please check that your locale settings:
            LANGUAGE = (unset),
            LC_ALL = (unset),
            LANG = "en_US.UTF-8"
        are supported and installed on your system.
    perl: warning: Falling back to the standard locale ("C").
    ```

    they are just locale warnings and do not prevent Docker from working.

    **Optional: fix locale warnings**

    ```bash
    apt install -y locales
    dpkg-reconfigure locales
    ```

    * Choose `en_US.UTF-8` as default

3. Run Vaultwarden via Docker

    ```bash
    docker run -d \
      --name vaultwarden \
      -v /data:/data \
      -e ROCKET_PORT=8080 \
      -e DOMAIN=https://vault.lan \
      -p 8080:8080 \
      --restart unless-stopped \
      vaultwarden/server:latest
    ```

    * -v /data:/data → persistent data on HDD (database `db.sqlite3`, keys, attachments)
    * -e ROCKET_PORT=8080 → Vaultwarden listens on 8080 inside the docker container
    * -e DOMAIN=https://vault.lan → the address users open. Vaultwarden uses it to build links (emails, attachments, WebAuthn/passkeys)
    * -p 8080:8080 → access to Vaultwarden web UI on port 8080
    * --restart unless-stopped → container starts automatically with LXC

    > The running container was created without `DOMAIN` (see [TODOs](#todos)).

### Configure Vaultwarden on NPM

#### Create SSL certificate

Create the certificate for `vault.lan` as described in [Certificates](#certificates-internal-https), then import it into NPM:

1. Go to SSL Certificates
2. Click Add SSL Certificate
3. Choose Custom

Fill in:
- Certificate name: vault-internal
- Certificate Key → upload vault.lan.key
- Certificate → upload vault.lan.crt
- Intermediate Certificate → leave empty (or `cerebro-ca.crt` if using the local CA)

#### Configure proxy host

In the NPM dashboard go to `Proxy Hosts → Add Proxy Host`.

"Details" Tab

- Domain Names: vault.lan
- Scheme: http
- Forward Hostname / IP: 192.168.1.12
- Forward Port: 8080
- Cache Assets: false
- Block Common Exploits: true
- Websockets Support: true

"SSL" Tab
- SSL Certificate: vault-internal
- Force SSL: true
- HTTP/2 Support: true

### Configure DNS record in Pi-hole

Access the Pi-hole interface and go to *Settings* > *Local DNS Records*. Add:
- Domain: vault.lan
- IP Address: 192.168.1.13 (NPM, not Vaultwarden)

### Verify

Inside LXC 101:

```bash
docker ps
```

Should show something like:

|CONTAINER ID|IMAGE|STATUS|PORTS|
| :---- | :---- | :---- | :---- |
|xxxxxxx|vaultwarden/server|Up 2s|0.0.0.0:8080->8080/tcp|

In the browser, test:
> https://vault.lan

* Vaultwarden web interface should open
* Click *Create account* and create your user account

Notes:
* `http://192.168.1.12:8080` still answers, but the web vault doesn't work there: browsers only allow the encryption functions (WebCrypto) it needs over HTTPS. Always use https://vault.lan.

### Hardening (after creating your accounts)

Not applied yet (see [TODOs](#todos)).

1. Disable new sign-ups, so nobody else on the network can create an account.
2. Set `DOMAIN` (if the container was created without it).

Docker containers can't change their settings in place: remove and recreate. Data is safe because it lives in /data (HDD).

```bash
# Inside LXC 101:
docker stop vaultwarden
docker rm vaultwarden

docker run -d \
  --name vaultwarden \
  -v /data:/data \
  -e ROCKET_PORT=8080 \
  -e DOMAIN=https://vault.lan \
  -e SIGNUPS_ALLOWED=false \
  -p 8080:8080 \
  --restart unless-stopped \
  vaultwarden/server:latest
```

* SIGNUPS_ALLOWED=false → blocks new accounts. Existing accounts keep working.

Take a backup before doing this: `vzdump 101 --storage data-hdd --mode snapshot --compress zstd` on the Proxmox host.

## Seafile

> To be set up from scratch. A previous attempt was removed.

## TODOs

### Host

- [x] **Fix the Proxmox repository on the host** (2026-11-17). `pve-no-subscription.list` pointed to `bookworm` (Proxmox 8) while the host runs Proxmox 9 (`trixie`), so Proxmox packages were never updated. Replaced by `/etc/apt/sources.list.d/proxmox.sources` (trixie), then `apt full-upgrade` (9.1 → 9.2).
- [ ] Reboot the host to load the new kernel installed by the upgrade (6.17.2 → 7.0.14, the new default in Proxmox 9.2). If something breaks, pick `6.17.13` in the boot menu and pin it with `proxmox-boot-tool kernel pin 6.17.13-21-pve`.
- [x] Set NPM (102) to start on boot (2026-11-17). Before, it stayed stopped after every host reboot.
- [x] Add `nofail` to the `/srv/data` line in the host's `/etc/fstab` (2026-11-17).
- [x] Run `pvesm set data-hdd --is_mountpoint yes` on the host (2026-11-17).
- [x] Clean up leftovers on the HDD from earlier attempts (2026-11-17): `/srv/data/containers/npm/` (~19GB old Docker data), `/srv/data/npm/`, `/srv/data/containers/seafile/`.
- [x] Move NPM data to the HDD, `/srv/data/containers/npm/{data,letsencrypt}` with bind mounts (2026-11-17).
- [x] Delete the pre-migration copies inside LXC 102 (`/data/npm.pre-hdd`, `/data/letsencrypt.pre-hdd`) after confirming NPM works (2026-11-17).

### Services

- [ ] Update Pi-hole (core v6.3 → v6.5, web v6.4 → v6.6.1, FTL v6.4.1 → v6.8): `pct exec 100 -- pihole -up`.
- [ ] Vaultwarden hardening: `SIGNUPS_ALLOWED=false` and `DOMAIN=https://vault.lan` (see [Hardening](#hardening-after-creating-your-accounts)).
- [ ] Decide whether to convert Vaultwarden (101) to unprivileged. Must be done before exposing it to the internet (Cloudflare).
- [ ] Test whether NPM (102) works without `lxc.apparmor.profile: unconfined`, `lxc.cgroup2.devices.allow: a` and `lxc.mount.auto: proc:rw sys:rw` (remove one at a time, restart, test).
- [ ] Migrate certificates to the local CA (see [Certificates](#certificates-internal-https)). Required before using the Bitwarden apps or Seafile sync clients.
- [ ] Seafile: set up from scratch (IP 192.168.1.14, DNS record already exists in Pi-hole).

### Logs

- [ ] Send logs to the HDD for Pi-hole, Vaultwarden and NPM. Today they are written inside each container disk (SSD):
  - Pi-hole: `/var/log/pihole` (~40MB)
  - Vaultwarden: Docker logs inside LXC 101
  - NPM: its own logs (`/data/npm/logs`) are on the HDD since the 2026-11-17 migration. Docker logs inside LXC 102 are still on the SSD
- [ ] Premise for every new service (Seafile, Immich, ...): logs go to `/srv/data/containers/<name>` from the start.

### Backups

- [ ] Implement backups. Nothing is scheduled today. The only backups are manual ones of 100, 101 and 102 taken on 2026-11-17 in `/srv/data/dump` (before the Proxmox upgrade).
  - Note: Proxmox backups (vzdump) **don't include bind mounts**. Pi-hole's `/etc/pihole` and Vaultwarden's `/data` (in `/srv/data/containers/`) must be backed up separately.
  - Proxmox backup job for the LXCs (Datacenter > Backup) to data-hdd
  - Retention policy (data-hdd currently keeps all backups forever: `prune-backups keep-all=1`)
  - Copy to a second disk or off-site (data-hdd is on the same server, so it doesn't protect against losing the server)
  - Test a restore