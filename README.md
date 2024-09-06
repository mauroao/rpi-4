<div align="center">
  <img src="./img/logo.png" alt="logo" /> <br /> <br />
  <h1>Raspberry Pi 4 as a Home Server</h1>
</div>
<br />

## Table of contents

- [Prepare SSD](#prepare-ssd)
- [Connect via SSH](#connect-via-ssh)
- [Configure USB Sata Adapter](#configure-usb-sata-adapter)
- [Update Ubuntu](#update-ubuntu)
- [Configure static IP](#configure-static-ip)
- [Install Docker](#install-docker)
- [Install Portainer](#install-portainer)
- [Install Samba](#install-samba)
- [Install File Browser](#install-file-browser)
- [Install MiniDLNA](#install-minidlna)
- [Install PiHole](#install-pihole)
- [Install Transmission](#install-transmission)
- [Install JDownloader2](#install-jdownloader2)
- [Other](#other)

## Prepare SSD

1. Open "Raspberry PI Imager" app;
2. Choose Ubuntu Server LTS 64bits;
3. Configure ssh;
4. Configure hostname: rpi-4
5. Configure user and pass: pi / pi
6. Write image;
7. Connect disk to USB 2.0;

 
## Connect via SSH

  ```Shell
  ssh pi@pi_ip_address
  password: pi
  ```

## Configure USB Sata Adapter 

1. Connect SSD via USB 2.0 port;
2. Type below command:
   ```shell
   lsusb
   ```
3. Check the output:
   ```
   Bus 002 Device 002: ID 152d:0583 JMicron Technology Corp. / JMicron USA Technology Corp. JMS583Gen 2 to PCIe Gen3x2 Bridge
   ```
4. Copy the ID (eg: `152d:0583`);
5. Run below command:
   ```shell
   sudo vim /boot/firmware/cmdline.txt
   ```
6. Insert below snipet at the begining of line (use the ID):
   ```
   usb-storage.quirks=152d:0583:u
   ``` 
7. Save the file;
8. Shutdonw Raspberry Pi; 
9. Disconnect Sata Adapter from USB 2.0, connect it to USB 3.0;
10. Turn it on;
 

## Update Ubuntu

  ```shell
  sudo apt update
  sudo apt -y upgrade
  ```

## Configure static IP

- Edit `/etc/netplan/01-netcfg.yaml ` file:
  ```yml
  network:
    version: 2
    ethernets:
      eth0:
        dhcp4: false
        addresses: [192.168.0.25/24]
        routes:
          - to: default
            via: 192.168.0.1
        nameservers:
          addresses: [8.8.8.8, 1.1.1.1]
  ```
- Run command:
  ```shell
  sudo netplan apply
  ```
- Check if adapter accepted configuration:
  ```
  ip addr show eth0
  ```

## Install Docker

- https://docs.docker.com/engine/install/ubuntu/
- https://docs.docker.com/engine/install/linux-postinstall/

## Install Portainer

  ```shell
  mkdir -p /home/pi/docker/app_data/portainer
  docker run -d -p 8000:8000 -p 9000:9000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /home/pi/docker/app_data/portainer:/data portainer/portainer-ce:latest
  ```
> https://docs.portainer.io/start/install-ce/server/docker/linux

## Install Samba

- Create folders:

  ```shell
  mkdir -p /home/pi/public
  sudo chmod -R 0777 /home/pi/public
  sudo chown -R nobody:nogroup /home/pi/public
  ```

- Install samba service:

  ```shell
  sudo apt update
  sudo apt install samba -y
  samba -V
  systemctl status smbd
  sudo smbpasswd -a pi
  ```

- Edit `/etc/samba/smb.conf` file:

  ```
  [global]
  workgroup = WORKGROUP
  server string = Samba Server %v
  security = user
  map to guest = Bad User
  dns proxy = no

  [public]
  path = /home/pi/public
  valid users = pi
  read only = no
  create mode = 0777
  directory mode = 0777
  ```

- Test samba configuration

  ```shell
  testparm
  ```

- Restart samba

  ```shell
  sudo systemctl restart smbd.service
  ```

## Install File Browser

- Create db and configuration files:
  ```shell
  mkdir -p /home/pi/docker/app_data/filebrowser/
  mkdir -p /home/pi/public/
  touch /home/pi/docker/app_data/filebrowser/filebrowser.db
  touch /home/pi/docker/app_data/filebrowser/settings.json
  ```

- Fill settings.json file with bellow content:
  ```yml
  {
    "port": 80,
    "baseURL": "",
    "address": "",
    "log": "stdout",
    "database": "/database/filebrowser.db",
    "root": "/srv"
  }
  ```

- Create stack on Portainer:
  ```yml
  version: "2.1"

  services:
    filebrowser:
      image: filebrowser/filebrowser:s6
      container_name: filebrowser 
      restart: unless-stopped
      environment:
        - PUID=1000
        - PGID=1000
        - TZ=America/Sao_Paulo
      volumes:
        - /home/pi/public:/srv
        - /home/pi/docker/app_data/filebrowser/filebrowser.db:/database/filebrowser.db
        - /home/pi/docker/app_data/filebrowser/settings.json:/config/settings.json
      ports:
        - 7070:80
  ```
- Add USER environment variables;
- Start the stack;

## Install MiniDLNA

- Create folders:

  ```shell
  mkdir -p /home/pi/public/media/movies
  mkdir -p /home/pi/public/media/tv
  mkdir -p /home/pi/public/media/other
  mkdir -p /home/pi/public/pics
  ```

- Create stack on Portainer:
  ```yml
  version: "2.1"
  services:
    minidlna:
      image: vladgh/minidlna
      container_name: minidlna
      network_mode: "host"
      environment:
        - PUID=1000
        - PGID=1000
        - TZ=America/Sao_Paulo
        - MINIDLNA_FRIENDLY_NAME=minidlna
        - MINIDLNA_MEDIA_DIR_1=V,/media
        - MINIDLNA_MEDIA_DIR_2=P,/pics
      volumes:
        - /home/pi/public/media:/media
        - /home/pi/public/pics:/pics
      restart: unless-stopped
  ```
- Start the stack.

## Install PiHole

- Create folders:

  ```shell
  mkdir -p /home/pi/docker/app-data/etc-dnsmasq.d
  mkdir -p /home/pi/docker/app-data/etc-pihole
  ```

- Create stack on Portainer:
  ```yml
  version: "3"

  services:
    pihole:
      container_name: pihole
      image: pihole/pihole:latest
      ports:
        - "53:53/tcp"
        - "53:53/udp"
        - "80:80/tcp"
      environment:
        - TZ=America/Sao_Paulo
        - WEBPASSWORD=admin
      volumes:
        - '/home/pi/docker/app-data/etc-pihole:/etc/pihole'
        - '/home/pi/docker/app-data/etc-dnsmasq.d:/etc/dnsmasq.d'
      restart: unless-stopped
    ```
- Start the stack.
- Go to http://rpi4-ip/admin

Throubleshooting:   
IF port 53 already in use, disable systemd-resolved service and change /etc/resolv.conf. 
> From: https://discourse.pi-hole.net/t/docker-unable-to-bind-to-port-53/45082/7 

## Install Transmission

- Create folders:

  ```shell
  mkdir -p /home/pi/public/torrents/watch
  ```

- Create stack on Portainer:
  ```yml
  version: "2.1"

  volumes:
    transmission-data: {}

  services:
    transmission:
      image: lscr.io/linuxserver/transmission:latest
      container_name: transmission
      restart: unless-stopped
      environment:
        - PUID=1000
        - PGID=1000
        - TZ=America/Sao_Paulo
      volumes:
        - /home/pi/public/torrents:/downloads
        - /home/pi/public/torrents/watch:/watch
        - transmission-data:/config
      ports:
        - 9595:9091
        - 51413:51413
        - 51413:51413/udp
  ```

## Install JDownloader2

- Create folders:

  ```shell
  mkdir -p    - /home/pi/public/downloads
  mkdir -p    - /home/pi/docker/app-data/jdownloader
  ```

- Create stack on Portainer:
  ```yml
  version: "2.1"

  services:
    jdownloader2:
      image: jlesage/jdownloader-2
      container_name: jdownloader-2
      restart: unless-stopped
      volumes:
        - /home/pi/public/downloads:/output:rw
        - /home/pi/docker/app-data/jdownloader:/config:rw
      ports:
        - 5800:5800
  ```
  
## Install Glances

- Create stack on Portainer:
  ```yml
  services:
    monitoring:
      image: nicolargo/glances:latest
      container_name: glances
      restart: always
      pid: host
      ports:
        - 61208-61209:61208-61209
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
        - /etc/os-release:/etc/os-release:ro
      environment:
        - "GLANCES_OPT=-w"
  ```

## Other

- https://askubuntu.com/questions/1263284/apt-update-throws-signature-error-in-ubuntu-20-04-container-on-arm 
- https://www.zdnet.com/article/raspberry-pi-extending-the-life-of-the-sd-card/
- https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/msd.md
- Measure temp: `vcgencmd measure_temp`
- https://xavierberger.github.io/RPi-Monitor-docs/index.html

