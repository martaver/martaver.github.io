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

It looks like we might have to Cross compile NodeJs for OpenWRT: https://github.com/netbeast/docs/wiki/Cross-Compile-Nodejs-for-OpenWrt
But this should work according to: https://wiki.openwrt.org/doc/howto/nodejs

Started to follow the steps found in this guide to compile my own OpenWRT version of NodeJS:
https://forum.openwrt.org/viewtopic.php?pid=263650#p263650

But, since ZSun doesn't have any internet of it's own, I'll try to do the actual compilation locally, then copy the package to OpenWRT via SFTP or Web interface(?)

Step './scripts/feeds update -a' opens the file in Notepad++... think something else should have happened here...

Looks like we'll have to do this from a Linux box, duh.
What now, cygwin?



Questions remaining:

How do we create a hotspot airport-style wifi-connect 'splash' page?
How do we mount the SD card in OpenWRT, so we can serve from it?
How do we get NodeJS on OpenWRT?