# Stellar Quickstart Docker Image for Raspberry Pi (RPi)

## Prerequisite
* Raspberry Pi Model 3 B/B+ (stellar-core supports only 64-bit operating systems, therefore you have only two options if you want to run stellar-core on RPi and that is RPi 3 Model B and B+.)
* 32 GB, Class 10 Grade 3 Micro SD card. (Stellar/Horizon does heavy I/O to persist data, running it on slower card causes it to perform very badly. I recommend at least Grade 3 Micro SD card i.e. ~90-100 MB/s write speed).
* Network cable to connect RPi to the router (apparently wifi driver doesn't work reliably with Fedora)
* HDMI cable and monitor (needed only during initial setup step)

## Installing Fedora
Fedora project has a [guide](https://fedoraproject.org/wiki/Architectures/ARM/Raspberry_Pi) explaining installation steps of Fedora on RPi. I followed that guide word-by-word and able to run it on RPi. I used the aarch64 image of [Fedora Server 28](https://dl.fedoraproject.org/pub/fedora/linux/releases/28/Server/aarch64/images/Fedora-Server-28-1.1.aarch64.raw.xz). (Note: aarch64 is important since as explained above stellar-core runs only on 64-bit OS.)

On Mac (Windows/Linux need different commands), write the image on sd-card with following command:

```
xz -d Fedora-IMAGE-NAME.aarch64.raw.xz
sudo dd bs=4m if=Fedora-IMAGE-NAME.aarch64.raw of=/dev/rdisk2 conv=sync,noerror
```

Where `Fedora-Server-IMAGE-NAME.aarch64.raw` is extracted file and `/dev/rdiskN` is the disk name which you can see using disk utility or use the diskutil command on Mac:

```
$ diskutil list
/dev/disk2 (internal, physical):
   #:     TYPE NAME                        SIZE       IDENTIFIER
   0:     FDisk_partition_scheme          *31.9 GB    disk2
   1:             Windows_FAT_32 NO NAME   31.9 GB    disk2s1
```

Once the disk image is written, just plug the card into the raspberry pi and via HDMI port connect it to monitor to finish the user creation step (you will need monitor only for this step):
![Screen-1](https://raw.githubusercontent.com/hard-codr/assets/master/first.jpg)
![Screen-2](https://raw.githubusercontent.com/hard-codr/assets/master/second.jpg)

```
Choose appropriate option as shown in screen:
1. Choose option 5 and then option 1 to create user.
2. Choose option 4 to enable password
3. Choose option 6 to make the user 'Administrator'
4. Choose option 3 to enter the username.
5. Choose option 5 to enter the password.
6. Enter 'c' to create user
```

After this you don't need a monitor, you can just ssh remotely into the RPi. Username and password chosen above can be used to log into raspberry pi via ssh.

After you log in, you will see very less available space on disk (e.g. by running `df -h .`), this is because OS is not aware of all the available space in your SD card and you need to expand the filesystem to reclaim all that space. The instruction given in Fedora distribution page didn’t work for me. I used following instead (depending upon what is your root partition number, mine was 3):

**Run this on RPi**
```
$ sudo growpart /dev/mmcblk0 3 # '3' here is root partition number
$ sudo pvresize /dev/mmcblk0p3 # again mmcblk0p3 is root partition
$ sudo lvresize /dev/fedora/root /dev/mmcblk0p3
$ sudo xfs_growfs -d /
```

After the first installation, do package update by running:
```
$ sudo dnf update
```
(This may take ages to complete depending upon your network speed)

## Configure docker on Fedora

Download the docker installer script, verify it and then run it to install docker:
```
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh
```

## Running stellar validator

The docker image steller-core uses the following software:

- Postgresql 9.5 is used for storing both stellar-core and horizon data
- [stellar-core](https://github.com/stellar/stellar-core)
- [horizon](https://github.com/stellar/horizon)
- Supervisord is used for managing the processes of the services above.

***WARNING: SDF doesn't provide official binaries of stellar-core and horizon for the arm64 platform. The binaries used in this docker image are built using above provided repositories on the arm64 and distributed [here](https://github.com/hard-codr/docker-stellar-core-horizon-rpi/releases). If you don't trust, then you can build those yourself. [Here](https://github.com/hard-codr/stellar-core) are instructions to build the stellar-core and horizon on RPi.***

Run interactive persistent validator on testnet
```
mkdir /home/pi/stellar/testnet
sudo docker run --rm -it -p "8000:8000" -p "11626:11626"  -p "11625:11625" -v "/home/pi/stellar/testnet/:/opt/stellar:Z" --name stellar hardcodr/stellar-quickstart --testnet
```

Run interactive persistent validator on pubnet
```
mkdir /home/pi/stellar/pubnet
sudo docker run --rm -it -p "8000:8000" -p "11626:11626"  -p "11625:11625" -v "/home/pi/stellar/pubnet/:/opt/stellar:Z" --name stellar hardcodr/stellar-quickstart --pubnet
```

This will ask you to provide a password for newly created Postgres instance, which stellar internally uses. This is just to do the initial setup of stellar-core and horizon. Following are steps to set up a persistent mode node that will run in the background:

1. Run an interactive session of the container at first, ensuring that all services start and run correctly (as explained above).
2. Shut down the interactive container (using Ctrl-C).
3. Start a new container using the same host directory in the background as follows.

```
$ sudo docker run -d \
    -v "/home/pi/stellar/pubnet/:/opt/stellar:Z" \
    -p "8000:8000" \
    -p "11625:11625" \
    -p "11626:11626" \
    --name stellar \
    hardcodr/stellar-quickstart --pubnet
```
***WARNING: You might not want to expose Adminstrator port 11626. Read [this](https://github.com/stellar/docker-stellar-core-horizon#security-considerations) before exposing port to outside world***

You can log into the container where all the processes are running using if you want to get your hand dirty:
```
$ sudo docker exec -it stellar bash
```

To access horizon API endpoint, point your browser to `http://192.168.2.42:8000/`, where `192.168.2.42` is your RPi IP address.


## Using horizon endpoint

You can use the horizon endpoint with your own choice of the library like you use normal SDF horizon endpoint. E.g. you can use the above endpoint with [Sirius](https://github.com/hard-codr/sirius) python library as follow:

```
import stellar

stellar.setup_custom_network('http://192.168.2.42:8000', 'Public Global Stellar Network ; September 2015')

t = stellar.transactions().fetch()
for i in t.records():
	print i
```
