# emonSD (emonPi & emonBase) SD card build

**For download: See [emonSD releases page (Git wiki)](https://github.com/openenergymonitor/emonpi/wiki/emonSD-pre-built-SD-card-Download-%26-Change-Log)**.

***

## emonPi/emonBase Resources

- [**User docs & setup guide**](http://guide.openenergymonitor.org)
- [**Purchase Pre-built SD card**](http://shop.openenergymonitor.com/emonsd-pre-loaded-raspberry-pi-sd-card/)
- [**Purchase emonPi**](shop.openenergymonitor.com/emonpi-3/)
- [**Pre-built image repository & changelog**](https://github.com/openenergymonitor/emonpi/wiki/emonSD-pre-built-SD-card-Download-&-Change-Log)

***

# emonSD Features  

- Base image RASPBIAN STRETCH LITE
- 8GB min SD card size
- Read-only root file system
- Stable Emoncms Core
- Apache web-server
- MQTT
- NodeRed
- OpenHAB


***

# Contents

<!-- toc -->

- [Initial Linux Setup](#initial-linux-setup)
  * [Update](#update)
  * [Change password, set international options & expand File-system](#change-password-set-international-options--expand-file-system)
  * [Change hostname](#change-hostname)
  * [Change Password](#change-password)
  * [Root filesystem read-only with RW data partition](#root-filesystem-read-only-with-rw-data-partition)
  * [Custom MOTD (message of the day)](#custom-motd-message-of-the-day)
  * [Memory Tweak](#memory-tweak)
- [RasPi Serial port setup](#raspi-serial-port-setup)
  * [Use custom cmdline.txt](#use-custom-cmdlinetxt)
  * [Disable serial console boot](#disable-serial-console-boot)
  * [Enable serial uploads with avrdude and autoreset](#enable-serial-uploads-with-avrdude-and-autoreset)
  * [Raspberry Pi 3 Compatibility](#raspberry-pi-3-compatibility)
- [Setup Read-only filesystem](#setup-read-only-filesystem)
- [Install emonPi Services](#install-emonpi-services)
  * [emonPi LCD service](#emonpi-lcd-service)
  * [Setup emonPi update](#setup-emonpi-update)
  * [Setup NTP Update](#setup-ntp-update)
  * [Fix Random seed](#fix-random-seed)
- [Install mosquitto MQTT](#install-mosquitto-mqtt)
- [emonHub](#emonhub)
  * [Install emonHub (emon-pi) variant:](#install-emonhub-emon-pi-variant)
- [Install Emoncms Core](#install-emoncms-core)
  * [emonPi specific Emoncms settings](#emonpi-specific-emoncms-settings)
  * [Move MYSQL database location](#move-mysql-database-location)
  * [Low write mode Emoncms optimisations](#low-write-mode-emoncms-optimisations)
  * [Install Emoncms Modules](#install-emoncms-modules)
    + [Configure Emoncms WiFi Module](#configure-emoncms-wifi-module)
      - [Install wifi-check script](#install-wifi-check-script)
    + [Setup Emoncms Backup & import module](#setup-emoncms-backup--import-module)
  * [Emoncms Language Support](#emoncms-language-support)
- [Install Emoncms MQTT input service](#install-emoncms-mqtt-input-service)
- [Lightwave RF MQTT service](#lightwave-rf-mqtt-service)
- [Configure Firewall](#configure-firewall)
- [Install NodeRED](#install-nodered)
- [Install openHAB](#install-openhab)
- [GSM HiLink Huawei USB modem dongle support](#gsm-hilink-huawei-usb-modem-dongle-support)
- [emonPi Setup Wifi AP mode](#emonpi-setup-wifi-ap-mode)
- [Symlink scripts](#symlink-scripts)
  * [emonSDexpand:](#emonsdexpand)
  * [Fstab:](#fstab)
- [Clean Up](#clean-up)
  * [Remove unused packages](#remove-unused-packages)
  * [Run Factory reset](#run-factory-reset)

<!-- tocstop -->

***

# Initial Linux Setup

## Update

	sudo apt-get update -y
	sudo apt-get upgrade -y

## Change password, set international options & expand File-system

Using raspi-config utility:

	sudo raspi-config

Raspbian minimal image is 2GB by default, expand to fill 4GB. Then reboot

## Change hostname

From 'raspberry' to 'emonpi'

	sudo nano /etc/hosts
	sudo nano /etc/hostname

## Change Password

From 'raspberrypi' to 'emonpi2016'

	sudo passwd

## Root filesystem read-only with RW data partition


## Custom MOTD (message of the day)

Use custom motd to alert users they are logging into an emonPi with RW / RO toggle instructions:

```
sudo rm /etc/motd
sudo ln -s /home/pi/emonpi/motd /etc/motd
```

## Memory Tweak

Append `gpu_mem=16` to `/boot/config.txt` this caps the RAM available to the GPU. Since we are running headless this will give us more RAM at the expense of the GPU


***

# RasPi Serial port setup


To allow the emonPi to communicate with the RasPi via serial we need to disconnect the terminal console from /tty/AMA0.

## Use custom cmdline.txt

	sudo mv /boot/cmdline.txt /boot/original.cmdline.txt
	sudo cp /home/pi/emonpi/cmdline.txt /boot/

This changes the  a custom cmdline.txt file:


From:

	dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait

(`In STRETCH the primary UART is referred to as serial0 instead of ttyAMA0 but same considerations apply.`)

To:

	dwc_otg.lpm_enable=0 console=tty1 elevator=noop root=/dev/mmcblk0p2 rootfstype=ext4 fsck.repair=yes rootwait

Note changing `elevator=deadline` to `elevator=noop` disk scheduler. Noop that is best recommend for flash disks, this will result in a reduction in disk I/O performance

"Noop: Inserts all the incoming I/O requests to a First In First Out queue and implements request merging. Best used with storage devices that does not depend on mechanical movement to access data (yes, like our flash drives). Advantage here is that flash drives does not require reordering of multiple I/O requests unlike in normal hard drives" - [Full article](http://androidmodguide.blogspot.co.uk/p/io-schedulers.html). [Forum topic discussion](http://openenergymonitor.org/emon/node/11695).

## Disable serial console boot

 	sudo systemctl stop serial-getty@ttyAMA0.service
 	sudo systemctl disable serial-getty@ttyAMA0.service
	sudo systemctl stop serial-getty@serial0.service
 	sudo systemctl disable serial-getty@serial0.service

`As above this may need to be changed to serial0`

## Enable serial uploads with avrdude and autoreset

	git clone https://github.com/openenergymonitor/avrdude-rpi.git ~/avrdude-rpi && ~/avrdude-rpi/install

## Raspberry Pi 3 Compatibility

The emonPi communicates with the RasPi via GPIO 14/15 which on the Model B,B+ and Pi2 is mapped to UART0. However on the Pi3 these pins are mapped to UART1 since UART0 is now used for the bluetooth module. However UART1 is software UART and baud rate is dependant to clock speed which can change with the CPU load, undervoltage and temperature; therefore not stable enough. One hack is to force the CPU to a lower speed ( add `core_freq=250` to `/boot/cmdline.txt`) which cripples the Pi3 performance. A better solution for the emonPi is to swap BT and map UART1 back to UART0 (ttyAMA0) so we can talk to the emomPi in the same way as before.  

RasPi 3 by default has bluetooth (BT) mapped to `/dev/AMA0`. To allow use to use high performace hardware serial (dev/ttyAMA0) and move bluetooth to software serial (/dev/ttyS0) we need to add an overlay to `config.txt`. See `/boot/overlays/README` for more details. See [this post for RasPi3 serial explained](https://spellfoundry.com/2016/05/29/configuring-gpio-serial-port-raspbian-jessie-including-pi-3/).

	sudo nano /boot/config.txt

Add to the end of the file

	dtoverlay=pi3-miniuart-bt

We also need to run to stop BT modem trying to use UART

	sudo systemctl disable hciuart

See [RasPi device tree commit](https://github.com/raspberrypi/firmware/commit/845eb064cb52af00f2ea33c0c9c54136f664a3e4) for `pi3-disable-bt` and [forum thread discussion](https://www.raspberrypi.org/forums/viewtopic.php?f=107&t=138223)


Reboot then test serial comms with:

	sudo minicom -D /dev/ttyAMA0 -b38400

(CHANGE TO serial0 on debian stretch??)

You should see data from emonPi ATmega328, sending serial `v` should result in emonPi returning it's firmware version and config settings.

To fix SSHD bug (when using the on board WiFi adapter and NO Ethernet). [Forum thread](https://openenergymonitor.org/emon/node/12566). Edit ` /etc/ssh/sshd_config ` and append:

	IPQoS cs0 cs0

`DOES this still apply in stretch?`

***

# Setup Read-only filesystem

# Enable root filesystem read-only mode

## Implications for read-only root mode in the systemd transition

One of the (many) design goals of systemd is to facilitate `volatile` and `stateless` Linux installations which have applications in security-sensitive and embedded applications. To this end systemd-based Linux are generally able to populate an empty /var at boot without issue. This aligns well with the requirement to reduce IO to prolong the life of SD memory. 

In a `volatile` configuration /etc is used for local configuration of the system which persists across reboots.

One aspect of the previous emonSD setup (pre-Stretch) which is inconsistent with systemd approach is that /var should be RW at all times. This means that /var should be mounted as tmpfs and all data created at runtime that needs to be persisted across reboots must be stored in /home/pi/data. 

## Setup Data partition

An alternative to following section is to use the [sdpart script](https://github.com/emoncms/usefulscripts) which will create & format the SD card partitions as necessary.

Assuming creating 300Mb data partition and starting with SD card image expanded to fill SD card (4GB in this example).

### Reduce size of root partition using Gparted

Using Gparted on Ubuntu reduce size of root partition by 300Mb to make space for data partition. Recommend leaving 10Mb free space at end of SD card 

### Creating 3rd partition:

    sudo fdisk -l
    Note end of last partition (assume 7391231)
    sudo fdisk /dev/mmcblk0
    enter: n->p->3
    enter: 7391232
    enter: default or 7821312
    enter: w (write partition to disk)
    fails with error, will write at reboot
    sudo reboot
    
    On reboot, login and run:
    sudo mkfs.ext2 -b 1024 /dev/mmcblk0p3
    
**Note:** *We create here an ext2 filesystem with a blocksize of 1024 bytes instead of the default 4096 bytes. A lower block size results in significant write load reduction when using an application like emoncms that only makes small but frequent and across many files updates to disk. Ext2 is chosen because it supports multiple linux user ownership options which are needed for the mysql data folder. Ext2 is non-journaling which reduces the write load a little although it may make data recovery harder vs Ext4, The data disk size is small however and the downtime from running fsck is perhaps less critical.*
   
Create a directory that will be a mount point for the rw data partition

    mkdir /home/pi/data
    
## Read-only mode

Then run these commands to make changes to filesystem

    sudo mv /etc/fstab /etc/fstab.orig
    sudo ln -s /home/pi/emonpi/fstab /etc/fstab
    sudo chmod a+x /etc/fstab
    sudo mv /etc/mtab /etc/mtab.orig
    sudo ln -s /proc/self/mounts /etc/mtab
    
The Pi will now run in Read-Only mode from the next restart. The following fstab is installed:

```
tmpfs           /tmp            tmpfs   nodev,nosuid,size=30M,mode=1777 0  0
tmpfs           /var            tmpfs   nodev,nosuid,size=300M,mode=1777 0  0
proc            /proc           proc    defaults 0 0
/dev/mmcblk0p1  /boot           vfat    defaults,noatime,nodiratime 0 2
/dev/mmcblk0p2  /               ext4    defaults,ro,noatime,nodiratime,errors=remount-ro 0 1
/dev/mmcblk0p3  /home/pi/data   ext2    defaults,rw,noatime,nodiratime,errors=remount-ro 0 2
```
Before restarting create two scripts to switch between read-only and write access modes. These scripts are in the emonPi git repo and can be installed with:

Firstly “ rpi-rw “ will be the command to unlock the filesystem for editing, and "rpi-ro" will put the system back to read-only mode:

    sudo ln -s /home/pi/emonpi/rpi-ro /usr/bin/rpi-ro
    sudo ln -s /home/pi/emonpi/rpi-rw /usr/bin/rpi-rw
        
Lastly reboot for changes to take effect

    sudo shutdown -r now
    
Login again, change data partition permissions:

    sudo chmod -R a+w data
    sudo chown -R pi data
    sudo chgrp -R pi data

Add message at shell login to alert users to RO mode:

	sudo nano /etc/motd

Add the line:

	The file system is in Read Only (RO) mode. If you need to make changes, use the command rpi-rw to put the file system in Read Write (RW) mode. Use rpi-ro to return to RO mode. The /home/pi/data directory is always in RW mode

## Tweak's to make system work with RO root FS

### Override Debian attempting to recreate symbolic link from /etc/mtab to /proc/self/mounts

Debian attempts to recreate the symbolic link from /etc/mtab to /proc/self/mounts which causes systemd-tmpfiles-create.service to exit with an error (regarded as error by some downstream e.g. https://bugs.launchpad.net/ubuntu/+source/systemd/+bug/1547033). This can be fixed by overriding /usr/lib/tmpfiles.d/debian.conf by placing the following in /etc/tmpfiles.d/debian.conf:

	# Created to override default to support read only root fs operation
	# Type Path    Mode UID  GID  Age Argument
	L /run/initctl -    -    -    -   /run/systemd/initctl/fifo
	L /run/shm     -    -    -    -   /dev/shm
	d /run/sendsigs.omit.d 0755 root root -

	L /etc/mtab   -    -    -    -  ../proc/self/mounts   #!<------ CHANGED!

### Create /var/lib/dbus at run time

(Maybe also a Raspbian/Debian bug?) systemd-tmpfiles-create.service attempts to put a symlink in /var/lib/dbus but where /var is tmpfs the /var/lib/dbus folder does not persist across reboots so this throws an error. This can be fixed by forcing creation of /var/lib/dbus every boot by overriding /usr/lib/tmpfiles.d/dbus.conf by putting the following in /etc/tmpfiles.d/dbus.conf :

	# Created to override default to support read only root fs operation
	# Type Path                     Mode    UID     GID     Age     Argument
	d /var/lib/dbus 0755 root root -
	L /var/lib/dbus/machine-id      -       -       -       -       /etc/machine-id

### systemd-timesyncd tmpfiles

systemd-timesyncd needs to be able to write the time to a file (/var/lib/systemd/timesync/clock).

A problem is referenced here but its cause is unknown (but likely due to /var not persisting across reboots):
https://bugs.freedesktop.org/show_bug.cgi?id=89217

Add the following in /etc/tmpfiles.d/systemd-timesyncd.conf:

	# Type Path    Mode UID  GID  Age Argument
	L /var/tmp -    -    -    -   /tmp
	d /var/lib/systemd/timesync 0755 root root -
	L /var/lib/systemd/timesync/clock - - - - /home/pi/data/clock

### dpkg tmpfiles

dpkg needs various files/directories to be created under /var/lib/dpkg in order to work properly.

	# Type Path                     Mode    UID     GID     Age     Argument
	d /var/lib/dpkg 0755 root root -
	d /var/lib/dpkg/updates 0755 root root -
	d /var/lib/dpkg/info 0755 root root -
	d /var/lib/dpkg/alternatives 0755 root root -
	f /var/lib/dpkg/status 0755 root root - 

### DNS Resolve fix

**Issue:** Linux needs to write to /etc/resolv.conf and /etc/resolv.conf.dhclient-new to save network DNS settings 

**Solution:** move files to ~/data RW partition and symlink (a modded dhclient-script file is not required in Debian Stretch).

#### Move resolv.conf to RW partition 
	cp /etc/resolv.conf /home/pi/data/
	sudo rm /etc/resolv.conf 
	sudo ln -s /home/pi/data/resolv.conf /etc/resolv.conf

#### Create resolv.conf.dhclient-new file in RW partition and symlink to /etc
	touch /home/pi/data/resolv.conf.dhclient-new
	sudo chmod 777 /home/pi/data/resolv.conf.dhclient-new 
	sudo rm /etc/resolv.conf.dhclient-new
	sudo ln -s /home/pi/data/resolv.conf.dhclient-new /etc/resolv.conf.dhclient-new

### NTP time fix (alternate solution needed under Debian Stretch)

Enables NTP and fake-hwclock to function on a Pi with a read-only file system

1. move the fake-hwclock back to it's original location if used on OEM SD card image
2. comment out the existing fake-hwclocks cron entry and create a ntp-backup cron entry
3. add an init script to "backup" current time & drift value at shutdown and by cron
4. remove these ntp-backup setup files once installation is done
5. get correct time from ntp servers
6. backup the current time to fake-hwclock

Install with:

	git clone https://github.com/openenergymonitor/ntp-backup.git ~/ntp-backup && ~/ntp-backup/install

[Discussion Thread](http://openenergymonitor.org/emon/node/5877)

### Fix Random seed (not needed in Debain Stretch)

## Install Docker under rootfs read only 

**Issue:** 
Docker typically runs from /var/lib. 

**Solution:**
Move the Docker directory to /home/pi/data.

## Move MYSQL database

After MYSQL has been installed (see Raspberry Pi Emoncms install) we will need to move the MYSQL database location to the RW data partition:

Move the database:

	mkdir /home/pi/data/mysql
	sudo cp -rp /var/lib/mysql/. /home/pi/data/mysql

Change MYSQL config to use database in new RW location change line `datadir` to `/home/pi/data/mysql`

	sudo nano /etc/mysql/my.cnf
# Install emonPi Services

	sudo apt-get install git-core -y
	cd /home/pi
	git clone https://github.com/openenergymonitor/emonpi
	git clone https://github.com/openenergymonitor/RFM2Pi

## emonPi LCD service

Enable I2C module, install packages needed for emonPi LCD service `python-smbus i2c-tools python-rpi.gpio python-pip redis-server` and `pip install uptime redis paho-mqtt`, and install emonPi LCD python scrip service to run at boot

	sudo emonpi/lcd/./install

Restart and test if I2C LCD is detected, it should be on address `0x27`:

	sudo i2cdetect -y 1

## Setup emonPi update

	Add Pi user cron entry:

		crontab -e

	Add the cron entries to check if emonpi update or emonpi backup has been triggered once every 60s:

	```
	MAILTO=""

	# # Run emonPi update script ever min, scrip exits unless update flag exists in /tmp
	* * * * * /home/pi/emonpi/service-runner >> /var/log/service-runner.log 2>&1
	```

	To enable triggering update on first factory boot (when emonpiupdate.log does not exist) add entry to `rc.local`:

		/home/pi/emonpi/./firstbootupdate

	This line should be present already if the emonPi speicifc `rc.local` file has been symlinked into place, if not:

		sudo rm /etc/rc.local
		sudo ln -s /home/pi/emonpi/emonpi/rc.local_jessieminimal /etc/rc.local

	The `update` script looks for a flag in `/tmp/emonpiupdate` which is set when use clicks Update in Emoncms. If flag is present then the update script runs `emonpiupdate`, `emoncmsupdate` and `emonhubupdate` and logs to `~/data/emonpiupdate.log`. If log file is not present then update is ran on first (factory) boot.

## Setup NTP Update

See [forum thread](https://community.openenergymonitor.org/t/emontx-communication-with-rpi/3659/2)

`sudo crontab -e`

Add

```
# Force NTP update every 1 hour

0 * * * * /home/pi/emonpi/ntp_update.sh >> /var/log/ntp_update.log 2>&1
```
***

# Install mosquitto MQTT

Install & configure Mosquitto MQTT server, PHP MQTT
```
wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key
sudo apt-key add mosquitto-repo.gpg.key
cd /etc/apt/sources.list.d/
sudo wget http://repo.mosquitto.org/debian/mosquitto-jessie.list
sudo apt-get update
sudo apt-get install mosquitto mosquitto-clients libmosquitto-dev -y
sudo pecl install Mosquitto-beta
(​Hit enter to autodetect libmosquitto location)
```

To check which version of Mosquitto pecl has installed run `$ pecl list-upgrades`. Currently emonSD (Oct17) is running Mosquitto 0.3. See this [forum post](https://community.openenergymonitor.org/t/raspbian-stretch/5096/60?u=glyn.hudson) and [this one](https://community.openenergymonitor.org/t/upgrading-emonsd-php-mosquito-version/6265/13) discussing the choice of Mosquitto alpa.

```
pecl list-upgrades
Channel pear.php.net: No upgrades available
Channel pear.swiftmailer.org: No upgrades available
pecl.php.net Available Upgrades (stable):
=========================================
Channel      Package   Local          Remote         Size
pecl.php.net dio       0.0.6 (beta)   0.1.0 (beta)   37kB
pecl.php.net Mosquitto 0.3.0 (beta)   0.4.0 (beta)   24kB
pecl.php.net redis     2.2.5 (stable) 3.1.6 (stable) 196kB
```

To upgrade to Mosquitto 0.4 run (currently not fully tested as of Jan 18): *Update: seems to work fine*

`$ sudo pecl install Mosquitto-0.4.0`

Install PHP Mosquitto extension:

    printf "extension=mosquitto.so" | sudo tee /etc/php5/mods-available/mosquitto.ini 1>&2
    sudo php5enmod mosquitto

Turn off Mosquitto persistence and enable authentication:

	sudo nano /etc/mosquitto/mosquitto.conf

Set `persistence false` and add the lines:

	allow_anonymous false
	password_file /etc/mosquitto/passwd

Create a password file for MQTT user `emonpi` with:

	sudo mosquitto_passwd -c /etc/mosquitto/passwd emonpi

Enter password `emonpimqtt2016` (default)

**Test MQTT**

Open *another shell window* to subscribe to a test topic:

	mosquitto_sub -v -u 'emonpi' -P 'emonpimqtt2016' -t 'test/topic'

 In the first shell oublish to the test topic :

	mosquitto_pub -u 'emonpi' -P 'emonpimqtt2016' -t 'test/topic' -m 'helloWorld'

If all is working we should see `helloWorld` in the second shell

***

# emonHub

Install other EmonHub dependencies if they have not been installed already by emonPi LCD service

```
sudo apt-get install -y python-pip python-serial python-configobj”
sudo pip install paho-mqtt pydispatcher
```

## Install emonHub (emon-pi) variant:

	cd ~/
	git clone https://github.com/openenergymonitor/emonhub.git && emonhub/install


Check log file:

	tail /var/log/emonhub/emonhub.log

We can check data is being posted to MQTT by subscribing to the base topic `emon/#` if new node variable MQTT topic structure is used or `emonhub/#` if old MQTT CSV topic structure is used

	mosquitto_sub -v -u 'emonpi' -P 'emonpimqtt2016' -t 'emon/#'


***

# Install Emoncms Core

*V9 Core (stable branch)*

Follow [Emoncms Raspberry Pi Raspbian Jessie install guide](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/readme.md)

## emonPi specific Emoncms settings

* MYSQL root password: `emonpimysql2016`
* MYSQL `emoncms` user password: `emonpiemoncmsmysql2016`
* Use emonPi default settings:
	* `cd /var/www/emoncms && cp default.emonpi.settings.php settings.php`
* Create feed directories in RW partition `/home/pi/data` instead of `/var/lib`:
	* `sudo mkdir /home/pi/data/{phpfina,phptimeseries}`
	* `sudo chown www-data:root /home/pi/data/{phpfina,phptimeseries}`

*Note: at time of writing the version of `php5-redis` included in the Raspbian Jessie sources (2.2.5-1) caused Apache to crash (segmentation errors in Apache error log). Installing the latest stable version (2.2.7) of php5-redis from github fixed the issue. This step probably won't be required in the future when the updated version of php5-redis makes it's way into the sources. The check the version in the sources: `sudo apt-cache show php5-redis`*

```
git clone --branch 2.2.7 https://github.com/phpredis/phpredis
cd phpredis
(check the version we are about to install:)
​cat php_redis.h | grep VERSION
phpize
./configure
sudo make
sudo make install
```


## Move MYSQL database location

The MYSQL database for Emoncms is usually in `/var/lib`, since the emonPi runs the root FS as read-only we need to move the MYSQL database to the RW partition `home/pi/data/mysql`. Follow stepsL [Moving MYSQL database in Emoncms Read-only documentation](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/read-only.md#move-mysql-database)


## Low write mode Emoncms optimisations

[Related forum discussion thread](http://openenergymonitor.org/emon/node/11695)

Follow [Raspberry Pi Emoncms Low-Write guide](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/Low-write-mode.md) from [Setting up logging on read-only FS](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/Low-write-mode.md#setup-logfile-environment) onwards (we have earlier setup read-only mode):

* [Setting up logging on read-only FS](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/Low-write-mode.md#setup-logfile-environment)
	* After running install we want to use emonPi specific rc.local instead:
		* `sudo rm /etc/rc.local`
		* `sudo ln -s /home/pi/emonpi/rc.local_jessieminimal /etc/rc.local`

* [Move PHP sessions to tmpfs (RAM)](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/Low-write-mode.md#move-php-sessions-to-tmpfs-ram)
* [Configure Redis](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/Low-write-mode.md#configure-redis)
	* After redis config **Ensure all redis databases have been removed from `/var/lib/redis`**
* Disable apache access log:
	* `sudo nano /etc/apache2/conf-enabled/other-vhosts-access-log.conf`
	* comment out the access log
* Ensure apache error log goes to /var/log/apache2/error.log
 	*  `sudo nano /etc/apache2/envvars`
 	*  Ensure `export APACHE_LOG_DIR=/var/log/apache2/$SUFFIX`

* [Reduce garbage in syslog due to Raspbian bug](https://openenergymonitor.org/emon/node/12566):

	* Edit the `/etc/rsyslog.conf` and comment out the last 4 lines that use xconsole:

```
#daemon.*;mail.*;\

#       news.err;\

#       *.=debug;*.=info;\

#       *.=notice;*.=warn       |/dev/xconsole
```

No need to [Enable Low-write mode in emoncms](https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/Low-write-mode.md#enable-low-write-mode-in-emoncms) since these changes to `settings.php` are already set in `default.emonpi.settings.php` that we installed earlier.

## Install Emoncms Modules

```
cd /var/www/emoncms/Modules
git clone https://github.com/emoncms/dashboard.git
git clone https://github.com/emoncms/app.git
git clone https://github.com/emoncms/wifi.git
git clone https://github.com/emoncms/config
git clone https://github.com/emoncms/graph

cd /home/pi/
git clone https://github.com/emoncms/backup
git clone https://github.com/emoncms/postprocess
git clone https://github.com/emoncms/usefulscripts
```

- After installing modules check and apply database updates in Emoncms Admin.

### Configure Emoncms WiFi Module

Follow install instructions in [WiFi module Readme](https://github.com/emoncms/wifi/blob/9.0/README.md) to give web user permission to execute system WLAN commands.

Move `wpa_supplicant.conf` (file where WiFi network authentication details are stored) to RW partition with symlink back to /etc:

	sudo mv /etc/wpa_supplicant/wpa_supplicant.conf /home/pi/data/wpa_supplicant.conf
	sudo ln -s /home/pi/data/wpa_supplicant.conf /etc/wpa_supplicant/wpa_supplicant.conf  

#### Install wifi-check script

To check if wifi is connected every 5min and re-initiate connection if not.

Add wifi-check script to `/user/local/bin`:

	sudo ln -s /home/pi/emonpi/wifi-check /usr/local/bin/wifi-check

Make wifi check script run as root crontab every 5min:

	sudo crontab -e

Add the line:

``*/5 * * * * /usr/local/bin/wifi-check > /var/log/wificheck.log 2>&1" mycron ; ``

### Setup Emoncms Backup & import module

Follow [instructions on emonPi backup module page](https://github.com/emoncms/backup) to symlink the Emoncms module from the folder in /home/pi/backup that also contains the backup shell scripts

## Emoncms Language Support

```
sudo apt-get install gettext
sudo dpkg-reconfigure locales
```
Select required languages from the list by pressing [Space], hit [Enter to install], see [language translations supported by Emoncms](https://github.com/emoncms/emoncms/tree/master/Modules/user/locale)

`sudo apache2 restart`

[more info on gettext](https://github.com/emoncms/emoncms/blob/master/docs/gettext.md)

***

# Install Emoncms MQTT input service

To enable data posted to base topic `emon/` topic to appear in Emoncms inputs e.g `emon/emontx/power1 10` creates an input from emonTx node with power1 with value 10.

[Follow install guide]( https://github.com/emoncms/emoncms/blob/master/docs/RaspberryPi/MQTT.md)

	sudo service mqtt_input start

[Forum Thread discussing emonhub MQTT support](http://openenergymonitor.org/emon/node/12091)

*Previously EmonHub posted decoded data to MQTT topic with CSV format to  `emonhub/rx/5/values 10,11,12` which the Nodes module suscribed to and decoded. Using the MQTT input service in conjunction with the new EmonHub MQTT node topic structure depreciates the Nodes module*

# Lightwave RF MQTT service

[Follow LightWave RF MQTT GitHub Repo Install Guide](https://github.com/openenergymonitor/lightwaverf-pi)

***

# Configure Firewall

Apache web server `sudo ufw allow 80/tcp` and`sudo ufw allow 443/tcp`

SSH server: `sudo ufw allow 22/tcp`

Mosquitto MQTT: `sudo ufw allow 1883/tcp`

Redis: `sudo ufw allow 6379/tcp`

OpenHAB: `sudo ufw allow 8080/tcp`

NodeRed: `sudo ufw allow 1880/tcp`

	sudo ufw enable

***

# Install NodeRED

[Follow OEM NodeRED install guide](https://github.com/openenergymonitor/oem_node-red) with OEM examples.

Default flows admin user: `emonpi` and password `emonpi2016`

***

# Install openHAB

[Follow OEM openHAB install guide](https://github.com/openenergymonitor/oem_openHab) with OEM examples

# GSM HiLink Huawei USB modem dongle support

[Follow Huawei Hi-Link RasPi setup guide](https://github.com/openenergymonitor/huawei-hilink-status/blob/master/README.md) to setup HiLink devices and useful status utility. The emonPiLCD now uses the same Huawei API to display GSM / 3G connection status and signal level on the LCD.

# emonPi Setup Wifi AP mode

Wifi Access Point mode is useful when using emonPi without a interent connection of with a 3G dongle.

[Follow guide to install hostpad and DHCP](https://github.com/openenergymonitor/emonpi/blob/master/docs/wifiAP.md)

Including installing start/ stop script to start the AP and also bridge the 3G dongle interface on eth1 to wlan0


***

# Symlink scripts

## emonSDexpand:

`sudo ln -s /home/pi/usefulscripts/sdpart/sdpart_imagefile /sbin/emonSDexpand`

## Fstab:

`sudo ln -s /home/pi/emonpi/fstab /etc/fstab`

***

# Clean Up

## Remove unused packages

`$ sudo apt-get clean all`

## Run Factory reset

```
sudo su
/home/pi/emonpi/.factoryreset
```
