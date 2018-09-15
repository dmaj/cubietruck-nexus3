# Nexus 3 on Cubietruck

Nexus on Cubietruck. How do you get such an idea? The SSDs are now very cheap (20€ for 120GB). So it doesn't matter if you install an SD card or an SSD. Thus I supplied a Raspberry and my Cubietrucks with SSDs. And I wanted to know if it would work.

## First of all
Nexus on Cubietruck is not fast. The start alone takes 10 minutes.
Once it's up and running, you can work with it surprisingly well. I always use it as a "travel repository".
But once again: You can't expect miracles.
The whole thing isn't set up perfectly either. Some configurations are already very fuzzy. I wouldn't use something like this for larger workgroups. But in my small projects it's been doing that for several months now. On a Raspberry I was not successful. 1GB RAM are then nevertheless too small. Unfortunately
## Installation
The installation consists of several steps. The first step is to load a current Debian onto an SD card. Then we configure the system on the SD card. The next step is to install the configured system on an SSD. After that we add a docker. Finally the containers for the Nexus 3 are built and started. I also want to access the Cubietruck via Wlan, because I can place it in any corner and don't need a port on my router.
### Installing Debian on SD card
Download Debian (Armbian) stretch for Cubietruck (mainline kernel 4.14.y or 4.17.y).
https://www.armbian.com/cubietruck/

Install the image on an SD.
I did it with Etcher under Windows.
https://etcher.io/

If someone prefers a different operating system to put the Debian system on the SD, there are several instructions available on the net.
#### Configuring the Debian
To set up the Wlan you first need a connection via LAN (with cable). Insert the SD card. Boot the operating system. The ssh service is already running after system startup. So you can login with ssh or via the console. Log in as root (password 1234). Set a new root password and create users (e.g. cubie). You will be guided by an installation process of Armbian.
#### Making the "non-free" Debian packages available
Logon with root privileges.
Execute the following command:
```
sudo printf '\ndeb http://ftp.de.debian.org/debian stretch main non-free\n' >> /etc/apt/sources.list
```
This makes the "non-free" packages of Debian available.
More info can be found here:
https://packages.debian.org/search?keywords=firmware-linux-nonfree
Update everything now:
```
sudo apt update
sudo apt upgrade
```
#### Set up the Wlan
Now you can install the firmware for the Wlan.
```
sudo apt install firmware-brcm80211 
```
Ensure that the network layer module is loaded at boot time. 
```
sudo echo b43 >> /etc/modules
```
Now everything should be ready to establish a Wlan connection.
To make sure the changes work as intended, we reboot the system.
```
sudo reboot
```

After the reboot you can check what the network is doing.
```
sudo ifconfig
```
You should see the adapters eth0, lo, wlan0.
Create the access data for the Wlan now.
```
sudo wpa_passphrase <ROUTER_SSID> <router_password> >/etc/wpa.conf
```
Add the interface.
```
sudo printf '\nallow-hotplug wlan0
auto wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa.conf\n' \
>> /etc/network/interfaces
```
After a restart of the network layer, the Wlan interface should have an IP.
```
sudo systemctl restart networking
```
If an error indicates that the driver is already loaded, simply execute the following command.
```
sudo pkill wpa_supplicant
```
And then restart the network again.
More information about setting up the network can be found here:
https://wiki.debianforum.de/WLAN_Configure

If everything has worked, then simply reboot.
Ideally the Wlan should be up after the reboot. But usually does not.
```
sudo ifconfig
```
Unfortunately shows no IP at the Wlan. This can be fixed by restarting the network.
```
sudo systemctl restart networking
```
But it would be nice if the Wlan would work immediately.
I don't always want to have a console or an lan cable on it.
I have a solution for that. It's not good, but it usually works.
The following command must be executed as "root".
```
sudo (echo "@reboot sleep 5 && /bin/systemctl restart networking") | crontab -
```
If the Wlan is not started reliably, increase the waiting time slightly.
To do this you have to edit the crontab.
```
sudo crontab -e
```
After a reboot the Wlan should be active.

Now there are some general configurations.
I don't want IPv6.
```
sudo printf 'net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv6.conf.wlan0.disable_ipv6 = 1
net.ipv6.conf.eth0.disable_ipv6 = 1\n'\
> /etc/sysctl.d/10-noipv6.conf
```
Test if it works.
```
sudo sysctl -p
```
Unfortunately the current kernel doesn't load the values at boot time!
So also here a hack.
```
sudo mv /etc/rc.local /etc/rc.local.bak
```
```
sudo printf '#!/bin/bash
/etc/init.d/procps restart
exit 0\n'\
> /etc/rc.local
 ```
```
sudo chmod 766 /etc/rc.local
```
I don't want avahi either (disturbs Wlan).
```
sudo printf 'AVAHI_DAEMON_START=0
AVAHI_DAEMON_DETECT_LOCAL=1\n'\
> /etc/default/avahi-daemon.conf
```
```
>sudo update-rc.d -f avahi-daemon remove
```
#### I don't like the visual mode of vi, and I want an "ll."
Switch off Visual Mode at vi.
```
printf ":set mouse-=a" > .vimrc
```
Alias for ll.
```
printf 'alias ll=\"ls -l\"\n' >> ~/.profile
```
### Installing Debian on SSD
Now prepare the transfer to the SSD.
```
sudo ln -s /usr/bin/sunxi-nand-part /usr/bin/nand-part
```
You only need this if the SSD has to be partitioned:
```
sudo update-command-not-found
```
And then execute the transfer:
```
sudo nand-sata-install
```
#### Result:
Cubietruck with current Debian and working Wlan.
### Installing docker
The next step is to add Docker to the system.
```
sudo wget -O get-docker.sh get.docker.com && chmod 766 get-docker.sh
```
```
sudo ./get-docker.sh
```
Prevent docker logs from flooding the system.
```
sudo printf '{
  \"log-driver\": "json-file",
  \"log-opts\": {
    \"max-size\": \"500k\",
    \"max-file\": \"3\"
  }
}\n' \
> /etc/docker/daemon.json
```
Add the previously created user to the docker group.
(/etc/group)
Then as this user continue working. But does not have to be. But it is better.
###  Build the containers. 
I do not provide the containers on hub.docker.com.
With Oracle this could easily lead to problems.
#### First Oracle Java
The openjdk is too slow for some reason. We take the Java from Oracle.
Change to "~/java/oracle". Place the Dockerfile.
Download Java:
```
curl -L -b "oraclelicense=a" -o jdk-8u181-linux-arm32-vfp-hflt.tar.gz http://download.oracle.com/otn-pub/java/jdk/8u181-b13/96a7b8442fe848ef90c96a2fad6ed6d1/jdk-8u181-linux-arm32-vfp-hflt.tar.gz

```
```
mkdir ~/java/oracle/opt
```
```
tar -xzvf jdk-8u181-linux-arm32-vfp-hflt.tar.gz -C ~/java/oracle/opt
```
```
rm ~/java/oracle/jdk-8u181-linux-arm32-vfp-hflt.tar.gz
```
```
mv ~/java/oracle/opt/jdk1.8.0_181 ~/java/oracle/opt/jdk
```
Cleaning up the oracle distribution
```
rm -rf                  opt/jdk/*src.zip \
                        opt/jdk/lib/missioncontrol \
                        opt/jdk/lib/visualvm \
                        opt/jdk/lib/*javafx* \
                        opt/jdk/jre/plugin \
                        opt/jdk/jre/bin/javaws \
                        opt/jdk/jre/bin/jjs \
                        opt/jdk/jre/bin/orbd \
                        opt/jdk/jre/bin/pack200 \
                        opt/jdk/jre/bin/policytool \
                        opt/jdk/jre/bin/rmid \
                        opt/jdk/jre/bin/rmiregistry \
                        opt/jdk/jre/bin/servertool \
                        opt/jdk/jre/bin/tnameserv \
                        opt/jdk/jre/bin/unpack200 \
                        opt/jdk/jre/lib/javaws.jar \
                        opt/jdk/jre/lib/deploy* \
                        opt/jdk/jre/lib/desktop \
                        opt/jdk/jre/lib/*javafx* \
                        opt/jdk/jre/lib/*jfx* \
                        opt/jdk/jre/lib/amd64/libdecora_sse.so \
                        opt/jdk/jre/lib/amd64/libprism_*.so \
                        opt/jdk/jre/lib/amd64/libfxplugins.so \
                        opt/jdk/jre/lib/amd64/libglass.so \
                        opt/jdk/jre/lib/amd64/libgstreamer-lite.so \
                        opt/jdk/jre/lib/amd64/libjavafx*.so \
                        opt/jdk/jre/lib/amd64/libjfx*.so \
                        opt/jdk/jre/lib/ext/jfxrt.jar \
                        opt/jdk/jre/lib/ext/rhinoceros.jar \
                        opt/jdk/jre/lib/oblique-fonts \
                        opt/jdk/jre/lib/plugin.jar
```
Building the image for Oracle.
```
docker build -t dmaj/java:8 .
```
### Second nexus 3
Switch to "~/nexus". Copy and build the docker file.
```
docker build --rm=true --tag=dmaj/nexus3 .
```
```
mkdir /nexus-data && chown 1000:1000 /nexus-data
```

## Start Nexus.
```
docker run -d -p 8081:8081 --name nexus --restart always -v /nexus-data:/nexus-data dmaj/nexus3
```
