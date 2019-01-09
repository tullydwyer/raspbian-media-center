# raspbian-media-center
Media server for the rasbian OS. Tested on a Rasberry Pi 3+.

The project is mainly for learning purposes.

## Features
- [Raspian Stretch](https://www.raspberrypi.org/downloads/raspbian/)
- [Kodi](https://kodi.tv/)
- [Deluge](https://deluge-torrent.org/)
- [Sonarr](https://sonarr.tv/)
- [Radarr](https://radarr.video/)
- [Jackett](https://github.com/Jackett/Jackett)
- [Docker](https://www.docker.com/)

## Install and Configure Raspian Stretch
Based the install guide located [here](https://www.makeuseof.com/tag/install-kodi-raspbian-media-center/)

### Create Rasbian Stretch SD card
Download Raspian Stretch [here](https://www.raspberrypi.org/downloads/raspbian/)

### Allow SSH into Raspberry Pi
Make an empty file called `SSH` in the `root` folder of SD card.

```sh
cd SD_ROOT
touch SSH
```

### Setup Wifi Connection Without Screen
> Seems to take a long time to connect on first boot

Put the configuration below in `wpa_supplicant.conf` file in the `root` folder of SD card.
```sh
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

```sh
ssh pi@IP
```

### Update and Upgrade OS
```sh
sudo apt update
sudo apt upgrade -y
```

### Install NTFS Drivers (Optional)
```sh
sudo apt install ntfs-3g -y
```

### Run the rasbian configuration program
```sh
sudo raspi-config
```

- Expand Filesystem
- Memory Split, allocate max RAM to GPU
- Finally, you need to enable certain video codecs that don’t run as standard. These include VP6, VP8, MJPEG, and Theora, among others. To do this, you need to enable the camera. While no camera needs to be connected, enabling this feature will ensure the codecs can be used.
- Advanced Options > GL Driver and set Original non-GL desktop driver

### Extend Default Pagefile Limit
```sh
sudo vi /etc/dphys-swapfile
```
Comment out the static swap size
```sh
sudo dphys-swapfile setup
```

### Use Googles DNS Servers
```
static domain_name_servers=8.8.8.8 8.8.4.4
```

## Install and Configure Docker
### Install Docker
> For Raspbian, installing using the repository is not yet supported. You must instead use the [convenience script](https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-convenience-script).

```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### Allow pi user to run Docker
```sh
sudo usermod -aG docker pi
```

### Disable Docker service autostart on boot
See issue [here](https://github.com/docker/for-win/issues/1192) (same for linux)
```sh
sudo systemctl disable docker
```

### Set docker to launch 60 seconds after boot
This is to allow OS to automount required drives before Docker starts.
```sh
@reboot sleep 60 && systemctl start docker
```

### Install Docker Compose
```sh
sudo apt install docker-compose
```

## Install Kodi
```sh
sudo apt-get update
sudo apt-get install -y kodi
## Auto start Kodi on boot
echo "@kodi" >> ~/.config/lxsession/LXDE-pi/autostart
```

## Create [Deluge](https://hub.docker.com/r/linuxserver/deluge/) Docker Container (ARM)
> The admin interface is available at IP:8112 with a default user/password of admin/deluge.

```sh
cd ~
git clone https://github.com/linuxserver/docker-deluge
sed -i 's@FROM lsiobase/alpine:edge@FROM lsiobase/alpine.armhf:edge@' docker-deluge/Dockerfile
docker build -t deluge.armhf docker-deluge/
```

- --net=host - Shares host networking with container, required.
- -v /config - deluge configs
- -v /downloads - torrent download directory
- -e PGID for GroupID - see below for explanation
- -e PUID for UserID - see below for explanation
- -e UMASK_SET for umask setting of deluge, optional , default if left unset is 022.
- -e TZ for timezone information, eg [Europe/London](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

## Create [Sonarr](https://hub.docker.com/r/linuxserver/sonarr/) Docker Container (ARM)
> The admin interface is available at IP:8989

```sh
cd ~
git clone https://github.com/linuxserver/docker-sonarr
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-sonarr/Dockerfile
docker build -t sonarr.armhf docker-sonarr/
```

- -p 8989 - the port sonarr webinterface
- -v /config - database and sonarr configs
- -v /tv - location of TV library on disk
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important - information
- -e PGID for for GroupID - see below for explanation
- -e PUID for for UserID - see below for explanation

## Create [Radarr](https://hub.docker.com/r/linuxserver/radarr/) Docker Container (ARM)
> The admin interface is available at IP:7878

```sh
git clone https://github.com/linuxserver/docker-radarr
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-radarr/Dockerfile
docker build -t radarr.armhf docker-radarr/
```

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

```sh
cd ~
git clone https://github.com/linuxserver/docker-jackett
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-jackett/Dockerfile
docker build -t jackett.armhf docker-jackett/
```

- -p 9117 - the port(s)
- -v /config - where Jackett should store its config file.
- -v /downloads - Path to torrent blackhole
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important - information
- -e RUN_OPTS - Optionally specify additional arguments to be passed. EG. - --ProxyConnection=10.0.0.100:1234
- -e PGID for GroupID - see below for explanation
- -e PUID for UserID - see below for explanation


## Commands
### Build Docker containers on local ARM architecture
```sh
cd ~
git clone https://github.com/linuxserver/docker-deluge
sed -i 's@FROM lsiobase/alpine:edge@FROM lsiobase/alpine.armhf:edge@' docker-deluge/Dockerfile
docker build -t deluge.armhf docker-deluge/

git clone https://github.com/linuxserver/docker-sonarr
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-sonarr/Dockerfile
docker build -t sonarr.armhf docker-sonarr/

git clone https://github.com/linuxserver/docker-radarr
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-radarr/Dockerfile
docker build -t radarr.armhf docker-radarr/

git clone https://github.com/linuxserver/docker-jackett
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-jackett/Dockerfile
docker build -t jackett.armhf docker-jackett/
```
### Create and Start Docker Containers
```sh
sudo docker-compose up # sudo for local networking
```

### Start Containers
```sh
sudo docker-compose start
```

### Stop Containers
```sh
sudo docker-compose stop
```

## Failures (these attempts didn't seem to work for unknown reasons)
### Modify Docker Service to use RequiresMountsFor
Always started Docker before the mount was up.
```sh
sudo systemctl edit docker.service
```
```
[Unit]
RequiresMountsFor=MOUNT_PATH
```

```sh
sudo systemctl daemon-reload
```

### Add cron to wait for Mount
Always started Docker before the mount was up.
```sh
@reboot until [ -d MOUNT_PATH ]; do sleep 1; done; systemctl start docker
```
