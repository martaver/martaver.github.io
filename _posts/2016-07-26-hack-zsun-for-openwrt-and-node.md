---
title: "A ZSun Captive Portal - Part 1 (OpenWRT, SSH, SFTP & Bridged Internet)"
modified: 2016-07-26T16:20:02-05:00
categories:  
  - projects  
tags:  
  - hacking
  - IoT
  
---

Not long ago, I spent some time tinkering with a [ZSun WiFi SD card reader](https://www.amazon.com/zsun-Memory-Reader-MicroSD-Flash/dp/B00U70J1BM) for [Koulu on Fire](http://kouluonfire.com/)'s 2016 Burning Man project. The idea was to power it from a USB power bank, flash it with [OpenWRT](https://openwrt.org/) and host a custom captive portal which allowed burners to take & share selfies. If you don't already know, a captive portal is a wifi network that redirects internet requests to a central page, like the login page of an Airport's WiFi network. OpenWRT is an open-source router firmware replacement that can be installed on most routers and has a massive community supporting it. Turned out to be quite fiddly, so I made this write up based on my lab notes. 

Follow this guide to transform a ZSun WiFi SD Card Reader into an OpenWRT router, ready to use with SSH, SFTP and a bridged internet connection.
 
In the next part of this series, we'll set the device up as a PHP Web Server that acts as a custom captive portal.

| Note: This guide is done from Windows - but many parts of this can be done easier from *nix.

## Flashing the ZSun with an OpenWRT image

First step is to flash the ZSun with a compatible build of OpenWRT. The following procedure worked, from [this guide](https://wiki.hackerspace.pl/projects:zsun-wifi-card-reader:factory-update). I added my notes to each step to help anybody who was as clueless as I was. Thanks and kudos to the guys at Hackerspace for this.

### Power up with an SD card

Before anything, do a fresh, deep FAT32 format of the SD card (128MB is plenty). I had some issues with a successful flash bricking a ZSun and that SD card wasn't empty when I tried.

> Insert FAT32-formatted microSD card (128MB is more than enough)

To power up the ZSun, just insert it into an empty USB slot. It matters whether you insert the SD card before or after powering up:

- If you insert the card before powering up, there will be no Wifi visible.
- If you power up first, the Wifi network will appear. You can insert the SD card afterwards.
 
 Power up first, then insert the SD card.
 
### Connect

> Connect to WiFi

You will find a WiFi network under 'ZSun'. Connect to it.

### Disable Workmode

> http://10.168.168.1:8080/goform/Setcardworkmode?workmode=0

By default, the ZSun's gateway is at 10.168.168.1 and most of its commands run of a simple HTTP API underneath the main app on port 8080. This command seems to disable the SD card reader functionality, allowing for an SMB Windows share with access to the SD card.

In your browser, go to [http://10.168.168.1:8080/goform/Setcardworkmode?workmode=0](http://10.168.168.1:8080/goform/Setcardworkmode?workmode=0).

After executing the command, the response visible should be `{"status":"0"}`. The card reader functionality disappears and you won't be able to access the SD card from Explorer anymore.

### Set up the SD card to flash Firmware

> Create .update directory on Zsun SMB/Windows Share (on Windows call it .update., don't ask why - it'll work) and copy SD100-openwrt.tar.gz (https://hackerspace.pl/~informatic/SD100-openwrt.tar.gz) there

This procedure uses the ZSun firmware's own firmware updater to flash it with OpenWRT. Pretty nifty. Here are the detailed steps that worked for me:

1. Download the tarball from [https://hackerspace.pl/~informatic/SD100-openwrt.tar.gz](https://hackerspace.pl/~informatic/SD100-openwrt.tar.gz)
1. Open Windows Explorer and browse to SMB Windows Share: `\\10.168.168.1`.
1. Explorer should prompt for a username & password, enter u:`admin` p:`admin`.
1. The `public` folder should appear. Open it.
1. In `public`, create a folder called `.update.`. Don't forget the `.` at the beginning and end - this is a trick with folder naming conventions in Windows.
1. Don't panic - instead of a folder called `.update.` a wierd looking folder called `UPDAT~WY.__` will appear. Open it.
1. Copy the tarball you downloaded into that folder.

### Run the firmware update

> http://10.168.168.1:8080/goform/upFirmWare

1. Now, browse to [http://10.168.168.1:8080/goform/upFirmWare](http://10.168.168.1:8080/goform/upFirmWare) to run the firmware update.

After executing the command, the response visible should be `{"status":2}`. This means the update was successful!

The ZSun will reboot automatically after this... watch the flashes to tell whether the flash worked.

### Wait for the reboot into OpenWRT

> Wait for long LED flash, then multiple fast flashes - now OpenWRT is booting for the first time.

There will be a long period of (normal slow) flashing, then one long flash, then a whole bunch of very fast flashes. The ZSun Wifi network disappears, and eventually re-appears as OpenWRT.

### Connect to LuCI

> … PROFIT!

LuCi is OpenWRT's web interface. If OpenWRT flashed successfully, you should see the WiFi network `OpenWRT`. Connect to it.

Notice that now your gateway IP is now `192.168.1.1`. If you browse to [http://192.168.1.1](http://192.168.1.1) you should be redirected to the LuCi web interface. Do this.

## The LuCi interface
Welcome to LuCi, the browser based management console for OpenWRT.

| Note: If you browse to LuCi under HTTPS, you will get an SSL warning, because OpenWRT uses a self-signed certificate. 

You can login with u:`root` and no password. You'll be prompted to set up a root password. Do this and keep the password for later.

## Factory Reset - How to reset OpenWRT if we stuff up?
First up, OpenWRT is router firmware and as you play around with it, you will most likely lock yourself out of your own WiFi network. To cover this, many guides online tell us to do a 'factory reset'. So, here are the facts:

1. The factory reset is a feature of Hackerspace's OpenWRT build.
1. The factory reset will return the device to the state immediately after the successful OpenWRT flash.
1. The factory reset won't help you recover a bricked device from a failed OpenWRT flash.

On a ZSun with Hackerspace's OpenWRT build, you can do a factory reset of the device by inserting the SD card during boot. This can be a little fiddly and the timing is tricky to get right, so here is a demonstration of how to 'spam' the factory reset and what the flashes look like when successfully reset:

<iframe width="640" height="360" src="https://www.youtube-nocookie.com/embed/8FRQjDHATRc?controls=0&amp;showinfo=0" frameborder="0" allowfullscreen></iframe>

## Accessing the OpenWRT terminal & file system

Initially, we can telnet into 192.168.1.1 to access OpenWRT's terminal, but OpenWRT is configured to disable telnet once a root password is set. After this point we have to use SSH & SFTP.

Using SSH is easy: [Install Putty](http://www.putty.org/) and connect to 192.168.1.1. 
Login with u:`root` p:`[yourrootpassword]` at the terminal. You should see the OpenWRT Chaos Calmer 15.05 build splash:

```
BusyBox v1.23.2 (2016-01-02 10:46:55 CET) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 CHAOS CALMER (15.05.1, r48532)
 -----------------------------------------------------
  * 1 1/2 oz Gin            Shake with a glassful
  * 1/4 oz Triple Sec       of broken ice and pour
  * 3/4 oz Lime Juice       unstrained into a goblet.
  * 1 1/2 oz Orange Juice
  * 1 tsp. Grenadine Syrup
 -----------------------------------------------------
root@OpenWrt:/# 
```

To access the file system, it's handy to use SFTP. For this we need to install some packages for OpenWRT. OpenWRT has a package manager called `opkg`, but it normally uses an internet connection to download and install new packages. To install packages offline, we'll need to use the SD card to install SFTP & its dependencies.

| Note: In OpenWRT, Windows Share SMB won't work anymore. It might be possible to set up, but I just settled for using SFTP instead.

So here's how to use an SD card with OpenWRT...

### Mounting & working with SD cards

After the flash, ZSun can will no longer act as an SD card reader for you out of the box. OpenWRT CAN read the SD cards you insert into the ZSun and also mount them in the file system, but it just won't appear in Explorer as an SD card reader when plugged into the computer via USB.

To use the SD card to copy files into OpenWRT, you'll need to copy files onto it from your machine using a separate SD card reader.
Insert the SD card into the Zsun, then, in LuCi, you can go to Mounting points and configure the SD card there.
You should see the SD card list under 'Mount Points'. Just check the checkbox in front of it and click save & apply.

By default it's mounted to the folder: `/mnt/sda1/`. For the *nixly challenged, like me, in SSH, we can use the commands `cd` and `ls -a` to change directory and list all respectively. 

Now, we can use the SD card and this mounting point to move files to/from our OpenWRT instance. Later, we can configure our web server to host from a folder you set up on that card.

### Setting up SFTP & accessing the OpenWRT file system
To use SFTP, we're going to have to install an SFTP Server. As mentioned earlier, we'll need to use `opkg` in offline mode, and install our packages from our SD card.
 
| Remember: we mounted the SD card using the LuCi interface, so in an SSH session we can find the mounted SD card under /mnt/sda1.
 
1. First, download the required packages from [OpenWRT downloads](https://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/generic/packages/):
 - [openssh-sftp-server_7.1p2-1_ar71xx.ipk](https://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/generic/packages/packages/openssh-sftp-server_7.1p2-1_ar71xx.ipk)
 - [libopenssl_1.0.2g-1_ar71xx.ipk](https://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/generic/packages/base/libopenssl_1.0.2g-1_ar71xx.ipk)
 - [zlib_1.2.8-1_ar71xx.ipk](https://downloads.openwrt.org/chaos_calmer/15.05/ar71xx/generic/packages/base/zlib_1.2.8-1_ar71xx.ipk)              
1. Save them into a folder of their own on the SD card, e.g. `opkg_packages` through your machine's own card reader.
1. Insert the card reader back into the ZSun, boot it & connect. 
1. `cd /mnt/sda1/opkg_packages`
1. Install all the new packages with: `opkg install *`. You should see:       
 
```
root@OpenWrt:/mnt/sda1/packages# opkg install *
Installing libopenssl (1.0.2g-1) to root...
Installing zlib (1.2.8-1) to root...
Installing openssh-sftp-server (7.1p2-1) to root...
Package zlib (1.2.8-1) installed in root is up to date.
Configuring openssh-sftp-server.
Configuring zlib.
Configuring libopenssl.
root@OpenWrt:/mnt/sda1/packages#
```
 
Now, you can use [FileZilla](https://filezilla-project.org/) (for example) to SFTP into our OpenWRT instance.
 
 1. Create an SFTP connection to: 192.168.1.1 with your root username/password and you should see the folder listing for `/root`.
 1. Go to the parent directory and you can browse the rest of OpenWRT's file system, including the SD card under `/mnt/sda1/`. 
   
## Set up OpenWRT as a repeater (bridge) of an existing WiFi network.
 
Installing everything via SD card will get old super quick, so let's get an internet connection bridged onto our OpenWRT.

The guys at the Piratebox forum posted [a wireless configuration](https://forum.piratebox.cc/read.php?22,16780,16819#msg-16819) that does this. It:
- Requires an existing WiFi network that exposes an internet connection.
- Gives OpenWRT access to the internet connection.
- Gives anyone connected to the OpenWRT WiFi network access to the internet connection.

| This worked with both an Android and iPhone hotspot and my home WiFi connection. For iPhone, don't forget to uncomment the 'psk2' encryption and enter the password your iphone gives you. Hotspot is the handiest, because you can take it with you so you can work from a cafe if you're mobile.

Once SFTP is set up, you can use FileZilla to edit config files. Find them in the `/etc` folder.

You'll need to edit these three files: 
 
### /etc/config/firewall

Replace only the entry for the `wan` config zone in `/etc/config/firewall` like so: 

```
config zone
  option name		wan
  option network		'wan wan6 wwan'
  option input		REJECT
  option output		ACCEPT
  option forward		REJECT
  option masq		1
  option mtu_fix		1
```
 
### /etc/config/wireless 
 
 The configuration below sets up two wireless interfaces:
 
  1. `wwan` - that connects to your local WiFi network (where we get our internet access from) and 
  2. `lan` - that broadcasts the OpenWRT network.
  
Replace the entire configuration file contents at `/etc/config/wireless` with this, make changes according to the notes below.
 
```
config wifi-device 'radio0'
 option type 'mac80211'
 option channel 'auto'
 option hwmode '11g'
 option path 'platform/ar933x_wmac'
 option htmode 'HT20'
 option disabled '0'
 option txpower '30'
 option country 'US'

config wifi-iface
 option ssid '<YourOwnWifi>'
 option device 'radio0'
 option mode 'sta'
 option network 'wwan'
 option encryption 'none'
 #option encryption 'psk2'
 #option key '<YourOwnWifiPassword>'

config wifi-iface
 option ssid 'OpenWrt'
 option device 'radio0'
 option mode 'ap'
 option network 'lan'
 option encryption 'none'
 #option encryption 'psk2'
 #option key 'yourSuperSecurePass'
```

The `config wifi-iface` entry with `option network 'wwan'` is where we set the Wifi network that we want to bridge. Make sure you:
 
  1. Replace `<YourOwnWifi>` with your own WiFi network's SSID. 
 
If your Wifi network is protected (I hope it is):
 
  1. Uncomment the lines `#option encryption ...` and `#option key ...` for this entry and replace `<YourOwnWifiPassword>` with your WiFi network's password.
  2. Comment `option encryption 'none'`
  
| Unfortunately, if OpenWRT can't connect to the WiFi network you configure here, OpenWRT won't initialize the other WiFi interface correctly. This will mean you can't connect to OpenWRT and you'll have to do a factory reset.
 
Later, if you want to disable the bridged WiFi just comment the whole `config wifi-iface` block for your WiFi network.
 
### /etc/config/network
 
 Allow the wwan to get it's IP via DHCP by adding the following lines to the end of `/etc/config/network`:
  
```
config interface 'wwan'
  option proto 'dhcp'
```
 
### Save, upload & reboot the device
 
 Save all files, upload them back to `/etc/config/` and reboot the device (just unplug/replug or select Reboot in the LuCi interface).
 
 Re-connect to OpenWRT and you should have internet access on the device.
 
 If you messed it up and the OpenWRT WiFi network doesn't re-appear, do a factory reset and check your wireless lan SSID & authentication settings in `/etc/config/wireless`.
 
#### Connected, but no internet access?
 
This is most likely due to an IP address conflict between the ZSun and the WiFi network you're connecting to. Usually, when connecting to an iPhone or Android hotspot, this isn't an issue - but most household routers also run on `192.168.1.1`, like OpenWRT, and this conflict prevents the internet connection from being bridged correctly.

You'll need to make sure that your source WiFi router doesn't have the same static IP configured as OpenWRT's `lan` interface, otherwise DNS will fail and you won't get internet access on your device.

To do this, set up OpenWRT to use an alternative IP for the `lan` interface by changing the following line in your `/etc/config/network` file, under the `config interface 'lan'` entry:
 
```
option ipaddr '192.168.10.1'
```
  
Set the IP address you want OpenWRT to operate from, e.g. 192.168.10.1 and save/upload.  
 
| Note: You will now have to access the OpenWRT through this IP, including SSH, SFTP and the LuCi interface.
 
### So...

Here we end part 1. The end result should be a ZSun device with OpenWRT flashed onto it, SSH and SFTP enabled to access the terminal and file system respectively and a successfully bridged internet connection so that `opkg` can work without having to mess around with SD cards. This should be a pretty handy milestone for any custom tinkering with the OpenWRT you might want to do.

In part 2, we'll dive deep and set up the PHP web server & captive portal on top of what we've built.

Enjoy!
 