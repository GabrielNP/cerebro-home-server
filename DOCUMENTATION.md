# <center>Cerebro Home Server</center>

# <center>Architecture and Installation Documentation</center>

## Table of Contents

* [Introduction](#introduction)
* |__[Objective](#objective)
* |__[Architecture - General]()
* |__[Architecture - Layers]()
* [Storage]()
* [IP Addressing]()
* [DNS (Pi-hole)]()
* [Reverse Proxy (Nginx Proxy Manager)]()
* [External Access (Cloudflare)]()
* [Vaultwarden]()
* [Seafile – Correct Decisions]()
* [Seafile – Docker Compose (Correct Model)]()
* [Nginx Proxy Manager – Seafile]()
* [Identified Errors and Lessons Learned]()
* [Backups (Planning)]()
* [Final Stable State]()
* [Hypervisor Proxmox VE]()
* [Prerequisites:]()
* [Dependencies:]()
* [Installation]()
* [Configuration]()
* [Adjust repositories]()
* [Register storage]()
* [Folder structure]()


## Introduction

### Objective

This document describes, in a consolidated manner, all the architecture decisions and the correct steps executed for creating a Home Server based on Proxmox VE, as discussed throughout the project. The goal is to allow manual reconstruction of an identical environment, avoiding already identified errors.

### Architecture - General

* Hypervisor: Proxmox VE (host called 'pve')  
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

* SSD Disk (~240GB) - sda: used for Proxmox system and LXC containers  
* HDD Disk (~1TB) - sdb: used for persistent data and logs
* ext4 instead of ZFS  
* HDD mounted on host at /srv/data  
* Standard structure: /srv/data/containers/<service>  
* Bind-mount from host to LXC mounted as /data inside the container

#### Logs

- Logs should be directed to sdb (1TB HDD), avoiding overloading sda (240GB SSD).
- Standardize this by creating volumes mounted at /srv/data/containers/<name>

### IP Addressing

- Proxmox Host (pve): 192.168.1.10  
- Pi-hole: 192.168.1.11  
- Vaultwarden: 192.168.1.12  
- Nginx Proxy Manager: 192.168.1.13  
- Seafile: 192.168.1.14  
- IPs defined outside DHCP range to avoid conflicts

### DNS (Pi-hole)

- Pi-hole configured as primary DNS on the network  
- Local records:  
  vault.lan -> 192.168.1.13  
  seafile.lan -> 192.168.1.13  
- Reason: all access goes through the reverse proxy

### Reverse Proxy (Nginx Proxy Manager)

- Running in a dedicated LXC  
- Default port: 80/443  
- Responsible for all internal HTTPS  
- Self-signed internal certificates (OpenSSL)  
- Let's Encrypt NOT used for .lan domains

### External Access (Cloudflare) {#external-access-(cloudflare)}

- Not yet configured. To be done in the future.

### Vaultwarden

- Running via Docker inside LXC  
- Access only via domain (vault.lan)  
- HTTPS mandatory  
- Access via IP stops working after HTTPS (expected behavior)  
- WebCrypto works correctly only under HTTPS

### Seafile – Correct Decisions

- Running via Docker Compose inside dedicated LXC  
- Database: MariaDB in separate container  
- Persistent data in /data/seafile  
- External access ONLY via NPM  
- Never access Seafile via IP after setting domain  
- Definitive URLs defined from first boot

### Seafile – Docker Compose (Correct Model)

Components:  
- seafile (internal port 80 / seahub 8000 / fileserver 8082)  
- seafile-db (MariaDB)  
- Critical variables:  
  SEAFILE_SERVER_HOSTNAME=seafile.lan  
  SEAFILE_SERVER_PROTOCOL=https  
  SERVICE_URL=https://seafile.lan  
  FILE_SERVER_ROOT=https://seafile.lan/seafhttp

### Nginx Proxy Manager – Seafile

- Forward Hostname: seafile.lan  
- Forward IP: 192.168.1.14  
- Forward Port: 80  
- Websockets: enabled  
- SSL: internal certificate  
- Custom Nginx Configuration:  
  location /seafhttp/ -> fileserver  
  location / -> seahub

### Identified Errors and Lessons Learned

- Never change domain/protocol after Seafile's first boot  
- Don't mix Seafile's internal HTTPS with NPM's HTTPS  
- Don't use incorrect rewrite on /seafhttp  
- CSRF is a symptom of URL inconsistency, not the cause  
- Bad Gateway usually indicates wrong port  
- Broken upload usually indicates incorrect FILE_SERVER_ROOT

### Backups (Planning)

- Not yet implemented  
- Future strategy:  
  - LXC Snapshot  
  - Backup to additional dedicated disk

### Final Stable State

- Pi-hole working as DNS  
- Vaultwarden accessible via HTTPS  
- Seafile accessible via HTTPS with upload and sync working  
- NPM as single entry point  
- Persistent data on HDD

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

By default, Proxmox uses "enterprise" (paid) repositories. Let's change to free repositories.

Click on *pve* and then *shell*. Run the following commands.

To disable enterprise repository:

```bash
mv /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/
```

Add no-subscription repository:

```bash
nano /etc/apt/sources.list.d/pve-no-subscription.list
```

Add to the file:

```bash
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

Save and exit.

Run to update and upgrade:

```bash
apt update && apt full-upgrade -y
```

This ensures you are in the free repository and receive regular updates.

You may get some unauthorized error, which means proxmox still tries to access some enterprise repository.

Search with:

`grep -r "enterprise.proxmox.com" /etc/apt/`

If anything appears, we need to disable it. Rename by adding ".disabled" to the end of the filename. Run the update and upgrade commands again.

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

This doesn't create anything on sda, just a logical directory.

7) Temporarily mount HDD (test):

```bash
mount /dev/sdb1 /srv/data
```

If something like below appears when running `df -h | grep data` it means it worked.

> /dev/sdb1   916G   ...  /srv/data

8) Permanently mount HDD:

Get UUID with blkid /dev/sdb1

Edit fstab with `nano /etc/fstab` and add at the end

> UUID=xxxx-xxxx-xxxx  /srv/data  ext4  defaults  0  2

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

Verify that data-hdd storage appears in the list, is enabled:yes and has no alerts or errors in red.

Up to here we integrated a data disk (sdb) and have a disk (sda) isolated for system.

#### Folder structure

1) Run `mkdir -p /srv/data/{backups,containers,media,services}`

This creates:

* backups → Proxmox and services backups  
* containers → persistent LXC data  
* media → Plex / Immich  
* services → configs (Pi-hole, Vaultwarden, etc.)

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

Download `debian-12-standard_*.tar.zst`. Wait for it to finish.

2) Create the container.

Click on Create CT

Fill in:

General Tab

* CT ID: <id usually starting from 100>  
* Hostname: <service name>  
* Password  
* SSH public key: <leave empty>  
  
  Click Next.

Template Tab

* Select Debian 12 template  
  
  Click Next.

Disks Tab

* Storage: "local-lvm" if you want to create on system disk; <disk_name>, in our case "data-hdd", on data disk.  
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
* IPv4: DHCP  
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
    - Core 1

    Memory:
    - 512MB
    - Swap: 512MB

    Network:
    - Name: eth0
    - Bridge: vmbr0
    - IPv4: Static 192.168.1.13/24
    - IPv6: None

    DNS:
    - Default

  2. Ensure container configuration is

      ```bash
      nano /etc/pve/lxc/102.conf
      ```

      Content:
      ```ini
      features: keyctl=1,nesting=1
      unprivileged: 1

      lxc.apparmor.profile: unconfined
      lxc.cgroup2.devices.allow: a
      lxc.mount.auto: proc:rw sys:rw
      ```

  2. Create persistent data directory

      ```bash
      # On Proxmox host:
      mkdir -p /srv/data/containers/npm/{data,letsencrypt}
      ```
  
  3. Configure bind mount for persistent data

      ```bash
      # Edit container configuration file:
      nano /etc/pve/lxc/102.conf

      # Add line at the end:
      mp0: /srv/data/containers/npm/data,mp=/data
      mp1: /srv/data/containers/npm/letsencrypt,mp=/etc/letsencrypt
      ```

  4. Adjust permissions

      ```bash
      chown -R 0:0 /srv/data/containers/npm
      ```
  
  5. Adjust NPM docker data-root to save images, layers, cache and persistent volumes on sdb (data) - Step 1

      ```bash
      mkdir -p /data/docker
      ```
  
  ### Install service

  1. Enter the container

      ```bash
      pct start 102
      pct enter 102
      ```

  2. Install docker
      
      ```bash
      # Update container
      apt update
      apt upgrade -y

      apt install -y docker.io docker-compose
      systemctl enable --now docker
      ```
  
  3. Configure NPM

      ```bash
      nano docker-compose.yml
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
            - "443:443"
            - "81:81"
          volumes:
            - /data/npm:/data
            - /data/letsencrypt:/etc/letsencrypt
      ```

  4. Adjust NPM docker data-root to save images, layers, cache and persistent volumes on sdb (data) - Step 2

      ```bash
      systemctl stop docker
      nano /etc/docker/daemon.json
      ```

      Content:
      
      ```json
      {
        "data-root": "/data/docker"
      }
      ```

  5. Bring up:

      ```bash
      systemctl daemon-reload
      systemctl start docker
      docker-compose up -d
      ```

  ### Verify

  1. Create admin account
  2. Access the dashboard

## Vaultwarden

### Prepare LXC
 1. Create LXC container (step by step from [virtualization configuration](#virtualization)).

    General:
    - CT ID: 101
    - Hostname: vaultwarden
    - Unprivileged: false
    - Password: <define>
    - SSH public key: empty

    Template:
    - Debian 12

    Disks:
    - Storage: local-lvm
    - Disk size: 8GB

    CPU:
    - Core 1

    Memory:
    - 512MB
    - Swap: 512MB

    Network:
    - Name: eth0
    - Bridge: vmbr0
    - IPv4: Static (we will define the IP manually)
    - IPv6: None

    DNS:
    - Default

    Or simply run:
    ```bash
    pct create 101 local:vztmpl/debian-12-standard_12.3-1_amd64.tar.gz \
    --hostname vaultwarden \
    --cores 1 \
    --memory 512 \
    --swap 512 \
    --net0 name=eth0,bridge=vmbr0,ip=192.168.1.12/24,gw=192.168.1.1,type=veth \
    --rootfs local-lvm:8 \
    --unprivileged 0 \
    --features nesting=1
    ```

 2. Create persistent data directory

    ```bash
    # On Proxmox host:
    mkdir -p /srv/data/containers/vaultwarden
    ```

 3. Configure bind mount for persistent data

    ```bash
    # Edit container configuration file:
    nano /etc/pve/lxc/101.conf

    # Add line at the end:
    mp0: /srv/data/containers/vaultwarden,mp=/data
    ```

    - /srv/data/containers/vaultwarden → host (HDD)  
    - /data → inside the container (Vaultwarden will use this directory for data)
 
 4. Set static IP in container

    ```bash
    # In the same file (101.conf), locate net0: and configure fixed IP, example:
    net0: name=eth0,bridge=vmbr0,ip=192.168.1.12/24,gw=192.168.1.1,type=veth
    ```

    - Choose a free IP on the network, outside the DHCP range  
    - Gateway is the same as your network (192.168.1.1)

 5. Adjust bind mount permissions

    ```bash
    # On Proxmox host:
    chown -R 101000:101000 /srv/data/containers/vaultwarden
    ```

    - 101000:101000 → mapping of unprivileged container root
    - This prevents permission issues when installing Vaultwarden

 6. Start the container

    ```bash
    pct enter 101
    ls -l /data
    ```

    The directory should exist and you should have no permission errors.

### Install service

1. Enter the container

    ```bash
    pct enter 101
    ls -l /data
    ```
    
2. Update container

    ```bash
    apt update
    apt upgrade -y
    ```

3. Install dependencies

    ```bash
    apt install -y curl docker.io
    systemctl enable --now docker
    ```

    If something like this appears

    ```plaintext
    Synchronizing state of docker.service with SysV service script with /lib/systemd/systemd-sysv-install.
    Executing: /lib/systemd/systemd-sysv-install enable docker
    perl: warning: Setting locale failed.
    perl: warning: Please check that your locale settings:
            LANGUAGE = (unset),
            LC_ALL = (unset),
            LANG = "en_US.UTF-8"
        are supported and installed on your system.
    perl: warning: Falling back to the standard locale ("C").
    perl: warning: Setting locale failed.
    perl: warning: Please check that your locale settings:
            LANGUAGE = (unset),
            LC_ALL = (unset),
            LANG = "en_US.UTF-8"
        are supported and installed on your system.
    perl: warning: Falling back to the standard locale ("C").
    ```

    are just locale warnings and do not prevent Docker from working.

    **Optional: fix locale warnings**

    ```bash
    apt install -y locales
    dpkg-reconfigure locales
    ```

    * Choose `en_US.UTF-8` as default

    ```bash
    export LANG=en_US.UTF-8
    export LC_ALL=en_US.UTF-8
    ```

4. Create data directory (bind mount already exists, just ensure permission inside container)

    ```bash
    mkdir -p /data
    chown -R 1000:1000 /data
    ```

    - 1000:1000 → default user the container will use for Docker Vaultwarden


5. Run Vaultwarden via Docker

    ```bash
    docker run -d \
      --name vaultwarden \
      -v /data:/data \
      -e ROCKET_PORT=8080 \
      -p 8080:8080 \
      --restart unless-stopped \
      vaultwarden/server:latest
    ```

    * -v /data:/data → persistent data on HDD
    * -p 8080:8080 → access to Vaultwarden web UI on port 8080
    * --restart unless-stopped → container starts automatically with LXC


### Configure Vaultwarden on NPM


#### Configure proxy host

In the NPM dashboard go to `Proxy Hosts → Add Proxy Host`.

"Details" Tab

- Domain Names (Example: vault.lan)
- Scheme: http
- Forward Hostname / IP: 192.168.1.12
- Forward Port: 8080
- Cache Assets: false
- Block Common Exploits: true
- Websockets Support: true

"SSL" Tab
- SSL Certificate: <empty>
- Force SSL: false
- HTTP/2 support: false

#### Create SSL certificate

1. Enter the NPM container

    ```bash
    docker exec -it nginx-proxy-manager bash
    ```
2. Generate the certificate

    ```bash
    mkdir -p /data/certs/vault
    cd /data/certs/vault

    openssl req -x509 -nodes -days 3650 \
      -newkey rsa:2048 \
      -keyout vault.lan.key \
      -out vault.lan.crt \
      -subj "/CN=vault.lan"
    ```

This creates two files:
- vault.lan.key  → private key
- vault.lan.crt  → certificate

Let's import this certificate into Nginx Proxy Manager using the Custom option.

In Nginx Proxy Manager:

1. Go to SSL Certificates
2. Click Add SSL Certificate
3. Choose Custom

Fill in:
- Certificate name: vault-internal
- Certificate Key → upload vault.lan.key  
- Certificate → upload vault.lan.crt  
- Intermediate Certificate → leave empty

This creates an internal certificate in NPM valid for HTTPS.

#### Associate SSL with proxy host

Now:
1. Go to Proxy Hosts
2. Edit the vault.lan proxy
3. Go to SSL tab

Configure:
- SSL Certificate: vault-internal
- Force SSL: true
- HTTP/2 Support: true

### Configure DNS record in Pi-hole

Access the Pi-hole interface and go to `Local DNS → DNS Records`. Add:
- Domain: vault.lan
- IP Address: <NPM IP>

### Verify

```bash
docker ps
```

Should show something like:

|CONTAINER ID|IMAGE|STATUS|PORTS|
| :---- | :---- | :---- | :---- |
|xxxxxxx|vaultwarden/server|Up 2s|0.0.0.0:8080->8080/tcp|

In the browser, you can test:
> https://vault.lan

* Vaultwarden web interface should open
* Initial login: create the admin account (you define it)

## Seafile
