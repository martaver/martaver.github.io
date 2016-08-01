---
title: "Hacking Zsun for OpenWRT and Node"
modified: 2016-07-26T16:20:02-05:00
categories:  
  - projects  
tags:  
  - hacking
  - IoT
  
---

OpenWRT : To act as HTTP server/router.

Another good guide here: http://www.hackup.net/2016/04/openwrt-and-the-zsun-card-reader/gi

"Fortunately, I learned another fact from a helpful comment on my previous post: By inserting or removing the SD card during boot, OpenWRT can be put into fail-safe mode that will reset the configuration. I kept inserting and removing it in quick succession for about 10 seconds in order to get the timing right. Now I’m able to connect to the WiFi again!"


after http://10.168.168.1:8080/goform/Setcardworkmode?workmode=0

No SD card: Will boot up with WiFi

When SD card is empty:
 - From cold boot: No Wifi
 - While booted: Wifi continues, can connect.
 
After workmode=0, Power Down, Re-insert without SD, E: appeared as an SD card reader, clicking on it asking to insert disk.
I inserted disk, KOULUONFIRE appeared.
Browseable, empty.
copied tarball to /.update.
Ran http://10.168.168.1:8080/goform/upFirmWare to get -2

After http://10.168.168.1:8080/goform/Setcardworkmode?workmode=0 the card reader disappears - can't access E: anymore.

Connecting to SMB Windows Share is: \\10.168.168.1 in explorer then u:admin p:admin
Public folder appears.
I created .update. in that, and it created a wierd looking folder UPDAT~WY.__
Copied tarball into that
Now http://10.168.168.1:8080/goform/upFirmWare
Status: 2  Update Successful!

Now what? Waiting for something to happen...
After a while of (normal slow) flashing, one long flash. Then a whole bunch of v. fast flashes.
Wifi network disappeared, re-appeared as OpenWRT.
Connected to OpenWRT.

Now, when I browse to http://10.168.168.1/ - refused to connect
With ipconfig, can see gateway is https://192.168.1.1/ (SSL warning, bypass this)
Can now see LUCI interface!!

Can't see the SD card device though....

Can login with root and no password. First thing, set up a root password.



Now, telnetting into 192.168.1.1 doesn't work - since we've set up the root password, telnet is disabled and we have to connect through SSH.

So let's install Putty and log into 192.168.1.1, root:[ourpassword] you get the OpenWRT Chaos Calmer build splash.





Now, how do we set up NodeJS on OpenWRT?
What's MQTT and Node-Red? (from: https://www.youtube.com/watch?v=1KwiZq-zc-E)

Node-Red is some IoT connectivity designer running on top of Node.js : http://nodered.org/
MQTT is a message queuing server for IOT



Step './scripts/feeds update -a' opens the file in Notepad++... think something else should have happened here...

Looks like we'll have to do this from a Linux box, duh.
What now, cygwin?



Questions remaining:

## How do we create a hotspot airport-style wifi-connect 'splash' page?

What you're describing is a captive portal system (hotspot, walled garden, etc). Two SO articles helped:
http://stackoverflow.com/questions/4454655/can-i-trigger-a-mobile-client-to-automatically-launch-a-web-browser-when-connect
http://stackoverflow.com/questions/10162913/configure-openwrt-to-provide-http-authentication/10669533#10669533

After a little research, there seem to be two OpenWRT packages that can provide 'splash' pages for a public wifi network:
https://wiki.openwrt.org/doc/howto/wireless.hotspot.wifidog
https://wiki.openwrt.org/doc/howto/wireless.hotspot.nodogsplash

WifiDog seems a little more mature with broader documentation, so we'll go with that.
 - It has the following dependencies:
  iptables-mod-extra
  iptables-mod-ipopt
  kmod-ipt-nat - x already installed
  iptables-mod-nat-extra
  libpthread - x
  
  Trying to install them revealed the following dependencies:
  kmod-ipt-extra
  kmod-ipt-ipopt
  kmod-ipt-nat-extra
  
  So back to download/SD them.
  
  installing kmod-ipt-extra... errors:
  Collected errors:
   * satisfy_dependencies_for: Cannot satisfy the following dependencies for kmod-ipt-extra:
   *      kernel (= 3.3.8-1-d6597ebf6203328d3519ea3c3371a493) *
   Installed version is: 3.18.20-1-59a2e66b6c0..b63
   
   We can try installing with '--force-depends'
   
   Realised you can do opkg install * from folder to install all local files.
   
   Same for wifidog package.
   Use FileZilla to 'edit' /etc/wifidog.conf - completely lost.
   Research on wifidog reveals it can't be used without an auth server - it's designed for htis purpose, and they recommend nocatsplash for a gateway/splash page only experience.
   
   Well, nocatsplash looks almost the same as nodogsplash, so let's try nodogsplash, since it was what we researched in the first place.
   
   Edit /etc/nodogsplash/nodogsplash.conf
   Set:
    RedirectURL http://192.168.1.1/
    AuthenticateImmediately yes
   
   /etc/init.d/nodogsplash start
   
   .. starts successfully.
   
   Connect with phone > no redirect.
   
   


## How do we mount the SD card in OpenWRT, so we can serve from it?
In OpenWRT you can go to Mounting points and configure the SD card there. By default it should be found in ./dev/sda1
Then uHttp can be configured to route traffic to a folder you set up in that card.



## How do we get NodeJS on OpenWRT?
It looks like we might have to Cross compile NodeJs for OpenWRT: https://github.com/netbeast/docs/wiki/Cross-Compile-Nodejs-for-OpenWrt
But this should work according to: https://wiki.openwrt.org/doc/howto/nodejs

Started to follow the steps found in this guide to compile my own OpenWRT version of NodeJS:
https://forum.openwrt.org/viewtopic.php?pid=263650#p263650

But, since ZSun doesn't have any internet of it's own, I'll try to do the actual compilation locally, then copy the package to OpenWRT via SFTP or Web interface(?)

To compile these locally, we need a Linux environment, and Cygwin isn't supported by the OpenWRT build environment since its file system is case insensitive.

Another option could be to simply write the API as a set of Lua scripts.
OpenWRT seems to also support a basic PHP installation - so this is a possibility.

We have very tight memory restrictions on the ZSun...
Basic reqs for Node asks for 256mb
Basic reqs for PHP asks for 64mb

So Node/PHP seem like difficult options. In the interest of safety, maybe LUA scripts are the best bet.

Webstorm has a LUA plugin but it looks kind of nasty.
Eclipse has an officially supported LUA plugin.

# A LUA Hello world, served from the SD card.

## How do we upload stuff to OpenWRT?
 - Windows Share/SMB?
    Windows Share SMB doesn't want to connect. Asks for a cert (just picked local cert) and then user/pass (entered root u/p) and then says can't find it.
 - SFTP?
     Status:	Connecting to 192.168.1.1...
     Response:	fzSftp started, protocol_version=3
     Command:	keyfile "C:\****\****.ppk"
     Command:	open "root@192.168.1.1" 22
     Command:	Pass: ********
     Status:	Connected to 192.168.1.1
     Error:	Received unexpected end-of-file from SFTP server
     Error:	Could not connect to server
     
     https://wiki.openwrt.org/doc/howto/sftp.server states that an extra module might need to be installed to be compatible with some clients.
     So I SSH in to run these commands:
     opkg update
     opkg install openssh-sftp-server
     
     The commands don't work because no internet access.
     
     A lot of things would be so much easier if I could get internet access on this OpenWRT.
     
     Maybe I can upload the items to the SD card and install them offline.
     
     Download openssh-sftp-server.ipk from https://downloads.openwrt.org/attitude_adjustment/12.09/ar71xx/generic/packages/
     Save to SD card through laptop's own card reader.
     Put the card reader back in ZSun, connect. OpenWRT will boot
     
     In SSH session, can find the SD card under /mnt/sda1. cd into switch to this folder and you can: 
     
     opkg install openssh-sftp-server_6.1p1-1_ar71xx.ipk
     
     install started but dependencies were missing:
      libopenssl
      zlib
     
     Go download them, put on SD and install them from /mtn/sda1 with:
     
     opkg install zlib_1.2.7-1_ar71xx.ipk
     opkg install libopenssl_1.0.1h-1_ar71xx.ipk

     Now we can install openssh-sftp-server.
     
     Now, I can use FileZilla to SFTP into our OpenWRT instance.
      
      
      