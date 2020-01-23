# raspbian-media-center
Media server for the rasbian OS. Tested on a Rasberry Pi 3+.

The project is mainly for learning purposes.

## Features
- [Raspian Buster](https://www.raspberrypi.org/downloads/raspbian/)
- [Kodi](https://kodi.tv/)
- [Deluge](https://deluge-torrent.org/)
- [Sonarr](https://sonarr.tv/)
- [Radarr](https://radarr.video/)
- [Jackett](https://github.com/Jackett/Jackett)
- [Docker](https://www.docker.com/)

## Install and Configure Raspian Buster
Based the install guide located [here](https://www.makeuseof.com/tag/install-kodi-raspbian-media-center/)

### Create Rasbian Buster SD card
Download Raspian Buster [here](https://www.raspberrypi.org/downloads/raspbian/)

### Allow SSH into Raspberry Pi
Make an empty file called `SSH` in the `root` folder of SD card.

```bash
cd SD_ROOT
touch SSH
```

### Setup Wifi Connection Without Screen
> Seems to take a long time to connect on first boot

Put the configuration below in `wpa_supplicant.conf` file in the `root` folder of SD card.
```bash
cd SD_ROOT
vi wpa_supplicant.conf
```

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US

network={
	ssid="YOUR_NETWORK_SSID"
	psk="YOUR_WPA/WPA2_KEY"
	key_mgmt=WPA-PSK
}
```

### SSH into server.
- Username: pi
- Password: raspberry

```bash
ssh pi@IP
```

### Update and Upgrade OS Packages
```bash
sudo apt update
sudo apt upgrade -y
```

### Run the rasbian configuration program
```bash
sudo raspi-config
```

- Expand Filesystem
- Memory Split, allocate 256 RAM to GPU
- Finally, you need to enable certain video codecs that don’t run as standard. These include VP6, VP8, MJPEG, and Theora, among others. To do this, you need to enable the camera. While no camera needs to be connected, enabling this feature will ensure the codecs can be used.
- Advanced Options > GL Driver and set Original non-GL desktop driver

### Extend Default Pagefile Limit (optional)
```bash
sudo vi /etc/dphys-swapfile
```
Comment out the static swap size
```bash
sudo dphys-swapfile setup
```

### Use Googles DNS Servers (optional)
```
sudo vim /etc/dhcpcd.conf
```

add the following to the end of the file
```txt
static domain_name_servers=8.8.8.8 8.8.4.4
```

```bash
sudo service dhcpcd restart
cat /etc/resolv.conf
```

## Install and Configure Docker
### Install Docker Using Convenience Script
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### Allow pi user to run Docker
```bash
sudo usermod -aG docker pi
```

### Disable Docker service autostart on boot
See issue [here](https://github.com/docker/for-win/issues/1192) (same for linux)
```bash
sudo systemctl disable docker
```

### Set docker to launch 60 seconds after boot
This is to allow OS to automount required drives before Docker starts.
```bash
crontab -e
```

```txt
@reboot sleep 60 && systemctl start docker
```

### Install Docker Compose
```bash
sudo apt install docker-compose
```

## Install Kodi
```bash
sudo apt update
sudo apt install kodi
```

## Auto-start Kodi on Boot With Desktop
```bash
echo "@kodi" >> ~/.config/lxsession/LXDE-pi/autostart
```

https://www.raspberrypi.org/forums/viewtopic.php?f=66&t=251645&sid=53d4d04842fd8e61f2c8cea77268b25e
## Auto-start Kodi on Boot Without Desktop
[Code Credit](https://www.raspberrypi.org/forums/viewtopic.php?f=66&t=251645&sid=53d4d04842fd8e61f2c8cea77268b25e)

Disable desktop.
```bash
sudo raspi-config
```
choose the option to boot to CLI autologin.

```bash
sudo tee -a /lib/systemd/system/kodi.service <<_EOF_
[Unit]
Description = Kodi Media Center
After = remote-fs.target network-online.target
Wants = network-online.target

[Service]
User = pi
Group = pi
Type = simple
ExecStart = /usr/bin/kodi-standalone
Restart = on-abort
RestartSec = 5

[Install]
WantedBy = multi-user.target
_EOF_

sudo systemctl enable kodi.service
```



## Create [Deluge](https://hub.docker.com/r/linuxserver/deluge/) Docker Container (ARM)
> The admin interface is available at IP:8112 with a default user/password of admin/deluge.

- --net=host - Shares host networking with container, required.
- -v /config - deluge configs
- -v /downloads - torrent download directory
- -e PGID for GroupID - see below for explanation
- -e PUID for UserID - see below for explanation
- -e UMASK_SET for umask setting of deluge, optional , default if left unset is 022.
- -e TZ for timezone information, eg [Europe/London](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

## [Sonarr](https://hub.docker.com/r/linuxserver/sonarr/) Docker Container (ARM)
> The admin interface is available at IP:8989

- -p 8989 - the port sonarr webinterface
- -v /config - database and sonarr configs
- -v /tv - location of TV library on disk
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important - information
- -e PGID for for GroupID - see below for explanation
- -e PUID for for UserID - see below for explanation

## [Radarr](https://hub.docker.com/r/linuxserver/radarr/) Docker Container (ARM)
> The admin interface is available at IP:7878

- -p 7878 - the port(s)
- -v /config - Radarr Application Data
- -v /downloads - Downloads Folder
- -v /movies - Movie Share
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important information
- -e PGID for for GroupID - see below for explanation
- -e PUID for for UserID - see below for explanation

## Create Jackett Docker Container (ARM)
> The admin interface is available at IP:9117

- -p 9117 - the port(s)
- -v /config - where Jackett should store its config file.
- -v /downloads - Path to torrent blackhole
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important - information
- -e RUN_OPTS - Optionally specify additional arguments to be passed. EG. - --ProxyConnection=10.0.0.100:1234
- -e PGID for GroupID - see below for explanation
- -e PUID for UserID - see below for explanation


## Commands
### Create and Start Docker Containers
```bash
docker-compose up
```

### Start Containers
```bash
sudo docker-compose start
```

### Stop Containers
```bash
sudo docker-compose stop
```

## Failures (these attempts didn't seem to work for unknown reasons)
### Modify Docker Service to use RequiresMountsFor
Always started Docker before the mount was up.
```bash
sudo systemctl edit docker.service
```
```
[Unit]
RequiresMountsFor=MOUNT_PATH
```

```bash
sudo systemctl daemon-reload
```

### Add cron to wait for Mount
Always started Docker before the mount was up.
```bash
@reboot until [ -d MOUNT_PATH ]; do sleep 1; done; systemctl start docker
```
