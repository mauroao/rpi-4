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
- [Install File Server](#install-file-server)
- [Install MiniDLNA](#install-minidlna)
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

  ```
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

## Install Docker

- https://docs.docker.com/engine/install/ubuntu/
- https://docs.docker.com/engine/install/linux-postinstall/

## Install Portainer

- https://docs.portainer.io/start/install-ce/server/docker/linux

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
## Install File Server

- Create db and configuration files:
  ```shell
  mkdir -p /home/pi/volumes/filebrowser/
  touch /home/pi/volumes/filebrowser/filebrowser.db
  touch /home/pi/volumes/filebrowser/settings.json
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
        - /home/pi/volumes/filebrowser/filebrowser.db:/database/filebrowser.db
        - /home/pi/volumes/filebrowser/settings.json:/config/settings.json
      ports:
        - 7070:80
  ```
- Start the stack.

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

## Other

- https://askubuntu.com/questions/1263284/apt-update-throws-signature-error-in-ubuntu-20-04-container-on-arm 
- https://www.zdnet.com/article/raspberry-pi-extending-the-life-of-the-sd-card/
- https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/msd.md
- Measure temp: `vcgencmd measure_temp`
- https://xavierberger.github.io/RPi-Monitor-docs/index.html

