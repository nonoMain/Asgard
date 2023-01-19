# Home server description

For private reasons this isn't the exact layout of my home server
but it's close enough and includes most of the things I run

This doc file includes

![](https://img.shields.io/badge/Proxmox-C76810?style=flat&logo=proxmox&logoColor=white)
![](https://img.shields.io/badge/TrueNAS-0385BD?style=flat&logo=truenas&logoColor=white)
![](https://img.shields.io/badge/PhotonOS-167C80?style=flat&logo=vmware&logoColor=white)
![](https://img.shields.io/badge/Docker-113E77?style=flat&logo=docker&logoColor=white)
![](https://img.shields.io/badge/Alpine_Linux-0D597F?style=flat&logo=alpine-linux&logoColor=white)

![](https://img.shields.io/badge/Nginx-028534?style=flat&logo=nginx&logoColor=white)
![](https://img.shields.io/badge/Wireguard-771517?style=flat&logo=wireguard&logoColor=white)

## 1. Install base system (PVE)

The base system I picked is [PVE (proxmox)](https://www.proxmox.com/en/) its a type 1 hypervisor with a free and open source version
**by default the web console runs on port 8006**

### Special notes

#### cancel the annoying subscription prompt:
---

run the command inside the proxmox shell

```bash
sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
```

#### enable iommu (passthrough)
---

from [proxmox wiki](https://pve.proxmox.com/wiki/Pci_passthrough#Enable_the_IOMMU):

edit the next line to /etc/default/grub:
`GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"`

edit the file /etc/modules and add these:

```txt
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

update the grub and initramfs :
```bash
update-grub && update-initramfs -u -k all
```

and check bios for virt and virt.io if it still doesn't work before checking anything else

#### enable updates without the enterprise repository:
---

add the line below to `/etc/apt/sources.list`

```repo
deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription
```

comment out the lines in `/etc/apt/sources.list.d/pve-enterprise.list`

```txt
# make sure all the lines in there are commented
# in my current version there is only one line..
```

now you can run:

```bash
apt-get update
apt dist-upgrade
```

#### run monitor (glances)
---

glances is good for running a web monitor interface to connect your server monitor into another dashboard (like homepage)

install the program

```bash
pip install setuptools bottle glances
glances -w -t 1 # test
```

if you are on systemd

create a service to run at start at `/etc/systemd/system/glances.service`

```toml
[Unit]  
Description=Web monitor
After=network.target  
StartLimitIntervalSec=0

[Service]  
Type=simple  
Restart=always  
RestartSec=1  
User=root
ExecStart=/usr/local/bin/glances -w --enable-light -t 5
  
[Install]  
WantedBy=multi-user.target
```

now enable it by `systemctl enable --now glances.service`

if you are on freeBSD
install:

```bash
pkg install py39-bottle sysutils/py-glances
```

note: disable local repos if you are on something like TrueNAS:

```bash
sed 's/enabled: yes/enabled: no/' /usr/local/etc/pkg/repos/local.conf
sed 's/enabled: no/enabled: yes/' /usr/local/etc/pkg/repos/FreeBSD.conf
```

create the service at `/usr/local/etc/rc.d/glances.sh`

```bash
#!/bin/sh

#PROVIDE: glances
#REQUIRE: NETWORKING
#KEYWORD: shutdown

. /etc/rc.subr

name="glances" rcvar="glances_enable" command="/usr/local/bin/glances -w --enable-light -t 5"

load_rc_config $name run_rc_command "$1"
```

now enable it by adding the next line to `/etc/rc.conf`

```bash
sysrc glances_enable="YES"
```

## 2. Install containers host (PhotonOS)

The next thing we need to do is to install an operating system which will run all of our containers (docker containers)

I picked [PhotonOS]( https://github.com/vmware/photon) for my docker host because its light and open source

### Special notes

#### allow ssh login to root user (unsafe but for the initialization of the server its easy and fast)
---

edit the `PermitRootLogin` to 'yes' in `/etc/ssh/sshd_config`

```txt
# Authentication:

#LoginGraceTime 2m
PermitRootLogin yes <---------------------- follow the arrow <-----------------------
#StrictModes yes
#MaxAuthTries 6
#MaxSessions 10
```

#### setup docker
---

enable the service

```bash
tdnf install docker docker-compose
systemctl enable --now docker
```

configure network

install `bindutils` for dig, nslookup, etc
```bash
tdnf install bindutils
```

allow containers to talk through host published ports:

```bash
iptables -A INPUT -m iprange --src-range 172.0.1.1-172.255.1.255 -j ACCEPT
iptables-save > /etc/systemd/scripts/ip4save
```

configure the DNS in `/etc/systemd/network/10-*-static*.network`

```toml
[Match]
Name=e*

[Network]
Address=192.168.0.2/24
Gateway=192.168.0.1
DNS=192.168.0.1
DNS=1.1.1.1
```

## 3. Install and configure containers

In this stage we'll deploy our [docker](https://www.docker.com/)  environment which means running all of our services

an example template for `docker-compose.yml`:
```yaml
version: '3.8'
services:
    portainer-ce:
        container_name: 'portainer'
        ports:
            - '8000:8000'
            - '9443:9443'
        volumes:
            - '/var/run/docker.sock:/var/run/docker.sock'
            - '/containers/portainer/data:/data'
        restart: unless-stopped
        image: 'portainer/portainer-ce:latest'

networks:
  default:
    name: 'asgard'
    driver: bridge
```

### Web manager (Portainer)
---

The [portainer](https://www.portainer.io/) service is the best web manager for containers from a remote connection

to run the portainer service all you need to do run the container

run the container (compose version)

```yaml
version: '3.8'
services:
    portainer-ce:
        hostname: 'portainer'
        container_name: 'portainer'
        ports:
            - '8000:8000'
            - '9443:9443'
        volumes:
            - '/var/run/docker.sock:/var/run/docker.sock'
            - '/containers/portainer/data:/data'
        restart: unless-stopped
        image: 'portainer/portainer-ce:latest'
```

the portainer service will run on port 9443

### Monitor (glances)
---

The glances monitor is a nice one if you know what options to give it

```yaml
version: '3.8'
services:
    glances:
        hostname: 'glances'
        container_name: 'glances'
        ports:
            - '61208:61208'
        environment:
            - GLANCES_OPT=-w --enable-light -t 5
        volumes:
            - '/var/run/docker.sock:/var/run/docker.sock:ro'
        restart: unless-stopped
        image: 'nicolargo/glances:latest-full'
```

### DNS Server (AdGuard)
---

The [AdGurad](https://adguard.com/en/welcome.html) DNS server is the best one I found for secure dns (hidden from my ISP and spoofing proof) and it also blocks ads..

it's web ui runs on port 3000 (make sure to configre it that way in the first time)

```yaml
version: '3.8'
services:
    adguard:
        hostname: 'adguard'
        container_name: 'adguard'
        ports:
            - 3000:3000
            - 53:53/udp
            - 53:53/tcp
            - 853:853/udp
            - 853:853/tcp
            - 784:784/udp
            - 8853:8853/udp
            - 5443:5443/udp
            - 5443:5443/tcp
        volumes:
            - '/containers/adguard/work:/opt/adguardhome/work'
            - '/containers/adguard/config:/opt/adguardhome/conf'
        restart: unless-stopped
        image: 'adguard/adguardhome'
```

upstream secure dns
```text
https://dns.quad9.net/dns-query
https://dns.google/dns-query
https://dns.cloudflare.com/dns-query
```

**Enable DNSSEC on DNS Settings!!!**

note: if you get the `listen tcp4 0.0.0.0:53: bind: address already in use` error then just run this:

```bash
systemctl stop systemd-resolved  
systemctl disable systemd-resolved
```

note: I disabled the dhcp ports it runs so if you wan't to use that you'll have to enable it

### WireGuard (home VPN)
---

The [WireGuard](https://www.wireguard.com/) service is the best way to access your home network securly from a different network (any network) because it makes your home a VPN of sort


I'm currently trying to use web consoles for my services so for this use case I'll use [wg-easy](https://github.com/WeeJeWel/wg-easy/) which is a wireguard + web ui
which in this case I'll run on local port 5000

wg-easy:

```yaml
version: '3.8'
services:
    wg-easy:
        hostname: 'wg-easy'
        container_name: 'wg-easy'
        cap_add:
            - 'NET_ADMIN'
            - 'SYS_MODULE'
        environment:
            - TZ=Asia/Jerusalem
            - WG_HOST='your.domain.example'
            - WG_PORT=<public port number>
            - PASSWORD='verygoodpassword'
            - WG_DEFAULT_DNS=adguard # container
        ports:
            - '51820:51820/udp'
            - '51821:51821/tcp'
        volumes:
            - '/containers/wg-easy/data:/etc/wireguard'
        restart: unless-stopped
        image: 'weejewel/wg-easy'
```

in case you want just wireguard use this: (taken from [here](https://docs.linuxserver.io/images/docker-wireguard))

```yaml
version: '3.8'
services:
    wireguard:
        hostname: 'wireguard'
        container_name: 'wireguard'
        cap_add:
            - 'NET_ADMIN'
            - 'SYS_MODULE'
        environment:
            - PUID=1000
            - PGID=1000
            - TZ=Asia/Jerusalem
            - SERVERPORT=51820
            - PEERS=1
            - PEERDNS=auto
            - INTERNAL_SUBNET=10.13.13.0
            - ALLOWEDIPS=0.0.0.0/0
            - LOG_CONFS=true
        ports:
            - '51820:51820/udp'
        volumes:
            - '/containers/wireguard/config:/config'
        restart: unless-stopped
        image: 'lscr.io/linuxserver/wireguard:latest'
```

now if you don't want to run a web console you can see the QR code by running

```bash
docker logs wireguard
```

### Reverse proxy (Nginx Proxy Manager)
---

Reverse proxy will help you redirect your public ip connection to your inner services securely

default email (also used as username): admin@example.com
default password: changeme

```yaml
version: '3.8'
services:
    nginx:
        hostname: 'nginx'
        container_name: 'nginxproxymanager'
        ports:
            - '1180:80'
            - '11443:443'
            - '5000:81'
        volumes:
            - /containers/nginxproxymanager/data:/data
            - /containers/nginxproxymanager/letsencrypt:/etc/letsencrypt
        restart: unless-stopped
        image: 'jc21/nginx-proxy-manager:latest'
```

### Dashboard (HomePage)
---

The [HomePage](https://github.com/benphelps/homepage) dashboard is the nicest looking dashboard with nice support for common services

from it's README.md:

```yaml
version: '3.8'
services:
    homepage:
        hostname: 'homepage'
        container_name: 'homepage'
        ports:
            - '8080:3000'
        volumes:
            - '/containers/homepage/config:/app/config'
            - '/containers/homepage/icons:/app/public/icons'
            - '/var/run/docker.sock:/var/run/docker.sock'
        restart: unless-stopped
        image: 'ghcr.io/benphelps/homepage:latest'
```
 
look [here](https://gethomepage.dev/en/configs/services/) for examples on how to configure the homepage

## Budget manager (FireFly III)
---

The [Firefly III](https://www.firefly-iii.org/) is the nicest foss one I found and it does the job for now

to run it you'll need two containers:

```yaml
version: '3.8'
services:
    fireflyiii:
        hostname: 'fireflyiii'
        container_name: 'fireflyiii'
        environment:
            - APP_KEY=CHANGEME_32_CHARS
            - DB_HOST=fireflyiiidb
            - DB_PORT=3306
            - DB_CONNECTION=mysql
            - DB_DATABASE=firefly
            - DB_USERNAME=firefly
            - DB_PASSWORD=secret_password
        ports:
            - '21280:8080'
        volumes:
            - '/containers/fireflyiii/upload:/var/www/html/storage/upload'
        depends_on:
            - fireflyiiidb
        restart: unless-stopped
        image: 'fireflyiii/core:latest'

    fireflyiiidb:
        hostname: 'fireflyiiidb'
        container_name: 'fireflyiiidb'
        environment:
            - MYSQL_RANDOM_ROOT_PASSWORD=yes
            - MYSQL_USER=firefly
            - MYSQL_PASSWORD=secret_password
            - MYSQL_DATABASE=firefly
        volumes:
            - '/containers/fireflyiiidb/mysql:/var/lib/mysql'
        restart: unless-stopped
        image: 'mariadb'
```

here is a way to generate the key:

```bash
head /dev/urandom | LC_ALL=C tr -dc 'A-Za-z0-9' | head -c 32 && echo
```

### DYNDNS Client (DDClient)
---

The [noip](https://www.noip.com/) ddns (dynamic dns) is the best free ddns service that will give you a free domain (one free domain but still)

you need to run a service that will update that domain meaning a service that will send updates every once in a while to the noip servers from your server so that they will know what ip your server is on, I'll use [ddclient](https://hub.docker.com/r/linuxserver/ddclient)

```yaml
version: '3.8'
services:
    ddclient-noip:
        hostname: 'ddclient'
        container_name: 'ddclient'
        volumes:
            - '/containers/ddclient/config:/config'
        environment:
            - PUID=1000
            - PGID=1000
            - TZ=Asia/Jerusalem
        restart: unless-stopped
        image: 'lscr.io/linuxserver/ddclient:latest'
```

### Monitoring (Uptime Kuma)
---

[Uptime Kuma]() is the nicest monitoring service I found

```yaml
version: '3.8'
services:
    uptime-kuma:
        hostname: 'uptimekuma'
        container_name: 'uptime-kuma'
        ports:
            - '3011:3001'
        volumes:
            - '/containers/uptime-kuma/data:/app/data'
        restart: unless-stopped
        image: 'louislam/uptime-kuma:latest'
```

## Final layout

### Devices

| Device | Desc                                                               |      OS      |   IP addr    | Manager             | Port |
|:------ |:------------------------------------------------------------------ |:------------:|:------------:|:------------------- |:----:|
| Server | Large server - mainly for hosting VMs                              | PVE(proxmox) | 192.168.0.10 | proxmox web console | 8006 |

### VMs

| Name    | Desc        | OS           | IP addr      | Manager             | Port |
| ------- | ----------- | ------------ | ------------ | ------------------- | ---- |
| TrueNAS | NAS Server  | TrueNAS Core | 192.168.0.20 | TrueNAS web console |  443 |
| Photon  | Docker host | PhotonOS     | 192.168.0.30 | Portainer           | 9443 |

### Server services

| Name    | Desc                                              | EndPoint           |
|:------- | ------------------------------------------------- |:------------------ |
| Glances | Web monitor - mainly for porting into a dashboard | 192.168.0.10:61208 |

### TrueNAS services

| Name    | Desc                                              | EndPoint           |
|:------- | ------------------------------------------------- |:------------------ |
| Glances | Web monitor - mainly for porting into a dashboard | 192.168.0.20:61208 |

### PhotonOS services

| Name                | Desc                                               | EndPoint           |
|:------------------- | -------------------------------------------------- |:------------------ |
| HomePage            | dashboard for viewing machine's stats and services | 192.168.0.30:8080  |
| Glances             | Web monitor - mainly for porting into a dashboard  | 192.168.0.30:61208 |
| Portainer           | Manage and view your containers from a web console | 192.168.0.30:9443  |
| AdGuard             | Secure DNS server (DNSSEC) and adblocker           | 192.168.0.30:53    |
| WireGuard (wg-easy) | home VPN - use to enter your LAN by a tunnel       | my.domain.com:8000 |
| Nginx Proxy Manager | reverse proxy manager                              | my.domain.com:443  |

## Useful links
[linuxserver.io](https://www.linuxserver.io/) - good place to look for services configurations and tricks
[my github](https://github.com/nonoMain) - good place to look for further configuration and tools

## Useful commands
check your connection speed through docker (using [speedtest.net](https://speedtest.net))
```bash
docker run --rm -it gists/speedtest-cli
```
