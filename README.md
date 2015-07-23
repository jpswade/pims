# pims
Pi Media Server

![Image of Pi Media Server](https://igcdn-photos-e-a.akamaihd.net/hphotos-ak-xaf1/t51.2885-15/11429724_983434871678140_1191806836_n.jpg)

## Introduction

The goal is to offload the Plex Media Server software from a power hungry 
 standard desktop computer to a low power Raspberry Pi.

The objective is to save money, power and resources.

## Prerequisites

### Hardware

* [Raspberry Pi 2 Model B Desktop (Quad Core CPU 900 MHz, 1 GB RAM, Linux)](http://amzn.to/1CPrKyf)
* [Samsung Memory 32GB Evo MicroSDHC UHS-I Grade 1 Class 10 Memory Card](http://amzn.to/1CPrL5c)
* [WD Elements 2TB USB 2.0 External Desktop Hard Drive - Black](http://amzn.to/1efVnNd)
* [Edimax EW-7811UN 150Mbps Wireless Nano USB Adapter](http://amzn.to/1efVoAG)
* [RASPBERRY PI 2 Power Supply (5v 2A PSU Charger)](http://amzn.to/1efVJmU)
* [Sugru](http://amzn.to/1eg2sNJ) (optional)

### Operating System

* Download [NOOBS_v*.zip](http://downloads.raspberrypi.org/NOOBS_latest), extract zip contents onto the SD card
* Power up the Pi, from the [NOOBS menu choose to Install Raspbian](https://www.raspberrypi.org/documentation/installation/noobs.md)
* Keep the [Command Line Boot Behaviour](http://elinux.org/RPi_raspi-config#boot_behaviour_-_Start_desktop_on_boot.3F)

### Connectivity

* Use the LAN port until Wireless is setup
* Connect to your Pi over SSH using [PuTTY](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html) (ie: `putty -ssh pi@raspberrypi.local`)

### Client

* A Plex compatible client, such as:
* [Roku 3 Streaming Media Player](http://amzn.to/1MmI3V3)
* [NOW TV Box - Sky](http://amzn.to/1MmI7nP) (rebranded Roku LT player) and [Roku Plex Client](https://github.com/plexinc/roku-client-public)
* [Plex for Android](http://amzn.to/1Ikqsvk)

## Install

### Wireless Setup
```
SSID=MyRouter
WKEY=WirelessKey
sudo sh -c "wpa_passphrase $SSID $WKEY>>/etc/wpa_supplicant/wpa_supplicant.conf"
sudo cp /etc/network/interfaces /etc/network/interfaces.bak
sudo sed -i 's/manual/dhcp/g' /etc/network/interfaces
sudo ifdown wlan0 && sudo ifup wlan0
```

### Raspberry Pi Update
```
sudo apt-get update && sudo apt-get upgrade -y
```

### Add Wireless Check
```
sudo touch /usr/local/bin/checkwifi.sh
echo "ping -c4 8.8.8.8 > /dev/null" | sudo tee -a /usr/local/bin/checkwifi.sh
echo 'if [ $? != 0 ]' | sudo tee -a /usr/local/bin/checkwifi.sh
echo "then" | sudo tee -a /usr/local/bin/checkwifi.sh
echo "  logger 'No network connection, restarting wlan0'" | sudo tee -a /usr/local/bin/checkwifi.sh
echo "  /sbin/ifdown 'wlan0'" | sudo tee -a /usr/local/bin/checkwifi.sh
echo "  sleep 5" | sudo tee -a /usr/local/bin/checkwifi.sh
echo "  /sbin/ifup --force 'wlan0'" | sudo tee -a /usr/local/bin/checkwifi.sh
echo "fi" | sudo tee -a /usr/local/bin/checkwifi.sh
sudo chmod 775 /usr/local/bin/checkwifi.sh
sudo crontab -l | sudo tee /root/root.cron
echo "*/5 * * * * /usr/bin/sudo -H /usr/local/bin/checkwifi.sh >> /dev/null 2>&1" | sudo tee -a /root/root.cron
sudo crontab /root/root.cron
sudo rm -fr /root/root.cron
```

### Setup Log Rotation
```
sudo cp /etc/logrotate.conf /etc/logrotate.conf.bak
echo "size 1G" | sudo tee -a /etc/logrotate.conf
sudo sed -i 's/#compress/compress/g' /etc/logrotate.conf
```

### Mount NTFS USB storage device
```
sudo apt-get install ntfs-3g exfat-utils -y
FSTYPE=ntfs
DEVICE=`sudo blkid | grep $FSTYPE | cut -d' ' -f1 | cut -d: -f1`
LABEL=`blkid -s LABEL -o value $DEVICE`
UUID=`blkid -s UUID -o value $DEVICE`
sudo mkdir /mnt/$LABEL
sudo chown -R pi:pi /mnt/$LABEL
sudo cp /etc/fstab /etc/fstab.bak
echo "UUID=$UUID /mnt/$LABEL $FSTYPE uid=pi,gid=pi 0 0" | sudo tee -a /etc/fstab
sudo cp /boot/cmdline.txt /boot/cmdline.txt.bak
echo "$(cat /boot/cmdline.txt) rootdelay=5"  | sudo tee /boot/cmdline.txt
sudo mount -a
```

### Install Plex Media Server
```
sudo apt-get update && sudo apt-get install apt-transport-https -y --force-yes
wget -O - https://dev2day.de/pms/dev2day-pms.gpg.key | sudo apt-key add -
echo "deb https://dev2day.de/pms/ wheezy main" | sudo tee /etc/apt/sources.list.d/pms.list
sudo apt-get update
sudo apt-get install plexmediaserver -y
sudo apt-get install libexpat1 -y
sudo cp "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml" "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml.bak"
sudo sed -i 's#/># DlnaEnabled="1"/>#g' "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml"
sudo service plexmediaserver restart
sudo update-rc.d plexmediaserver defaults
echo http://raspberrypi.local:32400/web/
```

#### Install ITV-Player
```
wget https://github.com/ReallyFuzzy/ITV-Player.bundle/archive/master.zip
unzip master.zip
rm -f master.zip
sudo mv ITV-Player.bundle-master "/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Plug-ins/ITV-Player.bundle"
sudo service plexmediaserver restart
```

### Install Samba
```
sudo apt-get install samba samba-common-bin -y
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
echo "[$LABEL]" | sudo tee -a /etc/samba/smb.conf
echo "        comment = $LABEL share" | sudo tee -a /etc/samba/smb.conf
echo "        path = /mnt/$LABEL" | sudo tee -a /etc/samba/smb.conf
echo "        browseable = yes" | sudo tee -a /etc/samba/smb.conf
echo "        read only = no" | sudo tee -a /etc/samba/smb.conf
testparm
sudo service samba restart
(echo raspberry; echo raspberry) | sudo smbpasswd -s pi
sudo apt-get install swat -y
echo http://raspberrypi.local:901/
```

### Install Transmission
```
sudo apt-get install transmission-daemon -y
sudo usermod -a -G pi debian-transmission
sudo service transmission-daemon stop
sudo cp /etc/transmission-daemon/settings.json /etc/transmission-daemon/settings.json.bak
mkdir /mnt/$LABEL/downloads/_torrents/
mkdir /mnt/$LABEL/downloads/_torrents/_complete
mkdir /mnt/$LABEL/downloads/_torrents/_incomplete
mkdir /mnt/$LABEL/downloads/_torrents/_watch
sudo sed -i 's#"download-dir": "/var/lib/transmission-daemon/downloads",#"download-dir": "/mnt/'$LABEL'/downloads/_torrents/_complete",#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"download-dir": "/home/debian-transmission/Downloads",#"download-dir": "/mnt/'$LABEL'/downloads/_torrents/_complete",#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"incomplete-dir": "/root/Downloads",#"incomplete-dir": "/mnt/'$LABEL'/downloads/_torrents/_incomplete",#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"incomplete-dir": "/home/debian-transmission/Downloads",#"incomplete-dir": "/mnt/'$LABEL'/downloads/_torrents/_incomplete",#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"incomplete-dir-enabled": false,#"incomplete-dir-enabled": true,#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"rpc-authentication-required": true,#"rpc-authentication-required": false,#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"rpc-whitelist": "127.0.0.1",#"rpc-whitelist": "127.0.0.1,192.168.*",#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#"utp-enabled": true#"utp-enabled": true,#g' /etc/transmission-daemon/settings.json
sudo sed -i 's#}##g' /etc/transmission-daemon/settings.json
echo '    "watch-dir": "/mnt/'$LABEL'/downloads/_torrents/_watch",' | sudo tee -a /etc/transmission-daemon/settings.json
echo '    "watch-dir-enabled": true' | sudo tee -a /etc/transmission-daemon/settings.json
echo "}" | sudo tee -a /etc/transmission-daemon/settings.json
sudo service transmission-daemon reload
sudo service transmission-daemon start
sudo update-rc.d transmission-daemon defaults
echo http://raspberrypi.local:9091
```

### Install BitTorrent Sync
```
sudo touch /etc/apt/sources.list.d/btsync.list
echo "deb http://debian.yeasoft.net/btsync wheezy main contrib non-free" | sudo tee -a /etc/apt/sources.list.d/btsync.list
echo "deb-src http://debian.yeasoft.net/btsync wheezy main contrib non-free" | sudo tee -a /etc/apt/sources.list.d/btsync.list
sudo gpg --keyserver pgp.mit.edu --recv-keys 6BF18B15
sudo gpg --armor --export 6BF18B15 | sudo apt-key add -
sudo apt-get update
sudo apt-get install btsync -y
sudo sh -c "/usr/lib/btsync/btsync-daemon --dump-sample-config > /etc/btsync/btsync.conf"
sudo sed -i 's#// "storage_path" : "/home/user/.sync",# "storage_path" : "/root/.sync",#g' /etc/btsync/btsync.conf
sudo service btsync start
sudo update-rc.d btsync defaults
echo Add /mnt/$LABEL/Sync via http://raspberrypi.local:8888
```

### Install ZoneMinder
```
sudo apt-get install zoneminder -y
sudo ln -s /etc/zm/apache.conf /etc/apache2/conf.d/zoneminder.conf
wget https://gist.githubusercontent.com/jpswade/a567831cb2305ad9f190/raw/d5200677385a30d47e9138cb81f22424b1cc0da7/IPCAM.pm
sudo mv IPCAM.pm /usr/share/perl5/ZoneMinder/Control/IPCAM.pm
sudo /etc/init.d/apache2 force-reload
sudo service zoneminder restart
echo http://www.zoneminder.com/wiki/index.php/Foscam_Clones#General_Zoneminder_Setup
echo http://raspberrypi.local/zm
```

# See also

* [Pi Media Server Image](https://instagram.com/p/4t_cmWrQY3/)
* [Potteries Hackspace](http://potterieshackspace.org/)
* [/r/raspberry_pi](https://www.reddit.com/r/raspberry_pi/)