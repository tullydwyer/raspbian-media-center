# raspbian-media-center
Media server for the rasbian OS. Tested on a Rasberry Pi 3+.
## Features
- [Raspian Stretch](https://www.raspberrypi.org/downloads/raspbian/)
- [Kodi](https://kodi.tv/)
- [Deluge](https://deluge-torrent.org/)
- [Sonarr](https://sonarr.tv/)
- [Radarr](https://radarr.video/)
- [Jackett](https://github.com/Jackett/Jackett)
- [Docker](https://www.docker.com/)

## Create Rasbian Stretch SD card
Download Raspian Stretch [here](https://www.raspberrypi.org/downloads/raspbian/)

## Allow SSH into Raspberry Pi
Make an empty file called 'SSH' in the 'root' folder of SD card.

```sh
touch SSH
```

## Setup Wifi Connection Without Screen*
\* This has never worked for me.

```sh
touch wpa_supplicant.conf
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

# Configure Raspian Stretch
Based the install guide located [here](https://www.makeuseof.com/tag/install-kodi-raspbian-media-center/)

SSH into server.
- Username: pi
- Password: raspberry

```sh
ssh pi@IP
```

Update and install ntfs drivers.

```sh
sudo apt update
sudo apt upgrade -y
# Support NTFS drives
sudo apt install ntfs-3g -y
```

then run the rasbian configuration program.

```sh
sudo raspi-config
```

- Expand Filesystem
- Memory Split, allocate max RAM to GPU
- Finally, you need to enable certain video codecs that don’t run as standard. These include VP6, VP8, MJPEG, and Theora, among others. To do this, you need to enable the camera. While no camera needs to be connected, enabling this feature will ensure the codecs can be used.
- Advanced Options > GL Driver and set Original non-GL desktop driver

# Install Docker
> For Raspbian, installing using the repository is not yet supported. You must instead use the [convenience script](https://docs.docker.com/install/linux/docker-ce/debian/#install-using-the-convenience-script).

```sh
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker pi
```

# Install Kodi
```sh
sudo apt-get update
sudo apt-get install -y kodi
# Auto start Kodi on boot
echo "@kodi" >> ~/.config/lxsession/LXDE-pi/autostart
```

# Install [Deluge](https://hub.docker.com/r/linuxserver/deluge/)
> The admin interface is available at IP:8112 with a default user/password of admin/deluge.

```sh
cd ~
docker stop deluge
docker rm deluge
git clone https://github.com/linuxserver/docker-deluge
sed -i 's@FROM lsiobase/alpine:edge@FROM lsiobase/alpine.armhf:edge@' docker-deluge/Dockerfile
docker build -t deluge.armhf docker-deluge/
docker create \
  --name deluge \
  --net=host \
  -e PGID=1000 \
  -e PUID=1000 \
  -e TZ=Australia/Brisbane \
  -e UMASK_SET=000 \
  -v <MEDIA_LOCATION>/Torrent_Downloads:/downloads \
  -v /var/lib/deluge/.config/deluge:/config \
  --restart unless-stopped \
  deluge.armhf

docker start deluge
```

- --net=host - Shares host networking with container, required.
- -v /config - deluge configs
- -v /downloads - torrent download directory
- -e PGID for GroupID - see below for explanation
- -e PUID for UserID - see below for explanation
- -e UMASK_SET for umask setting of deluge, optional , default if left unset is 022.
- -e TZ for timezone information, eg [Europe/London](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

# Install [Sonarr](https://hub.docker.com/r/linuxserver/sonarr/)
> The admin interface is available at IP:8989

```sh
cd ~
docker stop sonarr
docker rm sonarr
git clone https://github.com/linuxserver/docker-sonarr
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-sonarr/Dockerfile
docker build -t sonarr.armhf docker-sonarr/
docker create \
    --name sonarr \
    --net=host \
    -e PGID=1000 \
    -e PUID=1000 \
    -v /etc/localtime:/etc/localtime:ro \
    -v /var/lib/sonarr/config:/config \
    -v <MEDIA_LOCATION>/TV_Shows:/tv \
    -v <MEDIA_LOCATION>/Torrent_Downloads:/downloads \
    --restart unless-stopped \
    sonarr.armhf

docker start sonarr
```

- -p 8989 - the port sonarr webinterface
- -v /config - database and sonarr configs
- -v /tv - location of TV library on disk
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important - information
- -e PGID for for GroupID - see below for explanation
- -e PUID for for UserID - see below for explanation

# Install [Radarr](https://hub.docker.com/r/linuxserver/radarr/)
> The admin interface is available at IP:7878

```sh
cd ~
docker stop radarr
docker rm radarr
rm -rf docker-radarr/
git clone https://github.com/linuxserver/docker-radarr
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-radarr/Dockerfile
docker build -t radarr.armhf docker-radarr/
docker create \
  --name=radarr \
    -e PGID=1000 \
    -e PUID=1000 \
    -v /var/lib/radarr/config:/config \
    -v <MEDIA_LOCATION>/Torrent_Downloads:/downloads \
    -v <MEDIA_LOCATION>/Movies:/movies \
    -v /etc/localtime:/etc/localtime:ro \
    --restart unless-stopped \
    --net=host \
    radarr.armhf
docker start radarr
```

- -p 7878 - the port(s)
- -v /config - Radarr Application Data
- -v /downloads - Downloads Folder
- -v /movies - Movie Share
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important information
- -e PGID for for GroupID - see below for explanation
- -e PUID for for UserID - see below for explanation

# Install Jackett
> The admin interface is available at IP:9117

```sh
cd ~
git clone https://github.com/linuxserver/docker-jackett
sed -i 's@FROM lsiobase/mono:xenial@FROM lsiobase/mono.armhf:xenial@' docker-jackett/Dockerfile
docker build -t jackett.armhf docker-jackett/
docker rm jackett
docker create \
  --name=jackett \
  -v /var/lib/jackett/config:/config \
  -v /var/lib/jackett/downloads:/downloads \
  -v /etc/localtime:/etc/localtime:ro \
  -p 9117:9117 \
  jackett.armhf
docker start jackett
```

- -p 9117 - the port(s)
- -v /config - where Jackett should store its config file.
- -v /downloads - Path to torrent blackhole
- -v /etc/localtime for timesync - see Localtime for important information
- -e TZ for timezone information, Europe/London - see Localtime for important - information
- -e RUN_OPTS - Optionally specify additional arguments to be passed. EG. - --ProxyConnection=10.0.0.100:1234
- -e PGID for GroupID - see below for explanation
- -e PUID for UserID - see below for explanation
