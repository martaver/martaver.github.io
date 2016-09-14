---
title: "A ZSun Captive Portal - Part 2 (PHP, Lighttpd, SSL & Captive Redirects)"
modified: 2016-09-14T17:45:02-03:00
categories:  
  - projects  
tags:  
  - hacking
  - IoT
  
---

This is the second part in a series based on my tinkering with a [ZSun WiFi SD card reader](https://www.amazon.com/zsun-Memory-Reader-MicroSD-Flash/dp/B00U70J1BM) for [Koulu on Fire](http://kouluonfire.com/)'s 2016 Burning Man project. We powered it from a USB power bank, flashed it with [OpenWRT](https://openwrt.org/) and hosted a custom captive portal which allowed burners to take & share selfies. The whole idea is to have a closed world portal that simply 'appears' when users connect to the Wifi network. Turned out to be quite fiddly, so I made this write up based on my lab notes.

In the [previous post](https://martaver.github.io/projects/hack-zsun-for-openwrt-and-node/), we flashed a ZSun WiFi SD Card Reader devices with a custom build of OpenWRT and set it up for use with SSH, SFTP and a bridged internet connection.

Follow this guide to get our portal dev-ready with:
- PHP running on Lighttpd as a web server 
- hosted from SD card, so that the application can be hot-swapped between devices, and 
- configured network redirects & rewrites to capture users to our portal. 
  
# Setting up PHP on Lighttpd

Ideally, we want our PHP app to be the front face of our WiFi network, but still get admin access to OpenWRT. We can do this if we run LuCi on an alternative port. The easy way is to install Lighttpd & PHP to host our portal app and move uHTTP+LuCi to port 8080. Lighttpd also has the added advantage of being able to handle SSL certs & URL rewriting so we can set up our portal to be captive. 

Here's what we have to do:
  
## Install PHP 
 
Assuming we have everything ready to go from Part 1, this should be pretty easy. According to: https://wiki.openwrt.org/doc/uci/uhttpd, just:
 
1. SSH into OpenWRT and log in as root.
1. Install packages with `opkg install php5` and `opkg install php5-cgi`.
 
You can do a simple 'Hello World' sanity check with uhttpd to ensure PHP is working correctly by:
1. In the uHTTPd config file, add or uncomment: `list interpreter ".php=/usr/bin/php-cgi"`
1. Set up a basic hello world PHP file under `/www/yourphp/`
1. Browse to http://192.168.1.1/yourphp/file.php

| Remember: If you changed OpenWRT's gateway IP in [part 1](https://martaver.github.io/projects/hack-zsun-for-openwrt-and-node/), substitute `192.168.1.1` for it instead.
 
## Isolate Luci and get a lighttpd site running from an SD card
  
If we wanted to, we could replace uhttpd with lighttpd entirely, but for simplicity, we'll leave Luci running on uhttpd and just change it's port to 8080.
Then we'll set up lighttpd on 80 to handle our app for us.

1. To switch uhttpd over to port 8080, in `/etc/config/uhttpd` under `config uhttpd 'main'`, update the following line:

```
list listen_http '0.0.0.0:8080'
```

1. Reboot... LuCi will now be hosted out of 192.168.x.x:8080
1. opkg update
1. opkg install lighttpd
 
If you get this message, don't worry:

```
cat: can't open '/etc/lighttpd/conf.d/*.conf': No such file or directory
```

It's just trying to load configurations for plugins to lighttpd, but none exist yet.

Next, set up lighttpd to run on port 80, from the SD card:

1. Do `opkg install lighttpd-mod-cgi`
1. Do `mkdir /mnt/sda1/www` to create a 'www' folder on the SD card.
1. Stick a Hello World index.html in `/mnt/sda1/www` so you can check lighttpd is running.
1. Do `mkdir /mnt/sda1/logs` to create a folder for server logs on the SD card.
1. mkdir /mnt/sda1/logs/lighttpd
 
Next, configure lighttpd to listen on port 80 with directory-root at '/mnt/sda1/www'. Do this by editing `/etc/lighttpd/lighttpd.conf` to set:
```
 server.document-root        = "/mnt/sda1/www"
 server.errorlog             = "/mnt/sda1/logs/lighttpd/error.log"
 server.port                 = 80
```

Now, 
1. Set lighttpd to run on startup with `/etc/init.d/lighttpd enable`
1. Reboot the Zsun.
1. Browse to http://192.168.1.1 (or whatever your gateway IP is). You should get a 'Hello World'.


## Configure PHP to run from Lighttpd

Next, set up PHP (according to: [https://wiki.openwrt.org/doc/howto/http.lamp#lighttpd1](https://wiki.openwrt.org/doc/howto/http.lamp#lighttpd1)):

 1. In `/etc/lighttpd/lighttpd.conf` set the following to activate PHP for files with .php extension.
```
    cgi.assign = ( ".php"  => "/usr/bin/php-cgi" )
```    

Add index.php to the default file names, if not already set:
```
    index-file.names = ( "index.html", "default.html", "index.htm", "default.htm", "index.php" )
```
Restart lighthttpd: /etc/init.d/lighttpd restart

#### Error: 'Duplicate config variable'
If you get the error 'Duplicate config variable in conditional 0 global: cgi-assign', then it means there's already a cgi-assign variable in another config file.
Look in `/etc/lighttpd/conf.d/` - a config file here probably contains it already, and you can instead add `".php"  => "/usr/bin/php-cgi"`, to it there.
Don't forget to remove the conflicting `cgi-assign` entry in `lighttpd.conf`.

Finally, to get PHP working from the SD card you'll have to comment out the 'doc_root' in `/etc/php.ini`, like so: 
```
;doc_root = "/www"
```

Otherwise php will still try to only run scripts found in the old /www folder.

## Configure a secure HTTPS captive portal
 
The goal is to have a secure captive portal and, as a twist, we're going to be setting up a SSL certificate for a domain name that everyone gets redirected to.  

This sounds odd, but can be useful for a number of reasons. It gets rid of the SSL security warning from most browsers. If the ZSun is offline and there's no internet access, it allows your portal to 'impersonate' a real-world website, but display a different version of it. Another useful thing is that it lets users bookmark pages in the portal, which may correspond to real-world pages once they get internet access. It also allows your portal to operate under HTTPS which enables WebRTC (webcams) to work in certain browsers, such as Chrome.

The following steps will show you how to:
  
  1. re-route any DNS requests to our web server's IP.
  2. re-write all HTTP requests to HTTPS
  3. redirect any 404s to our App's root page

It assumes that you have an SSL certificate for a particular domain that you want to run on the WiFi, 
  
The end result will be a 'captive portal', where anybody who connects to the Wifi network will always be redirected to our app, which is hosted under the HTTPS secured domain name of your cert.
  
### How does a captive portal work?

When we connect to an Airport WiFi network we've all seen the 'log in' page that pops up. It's this page we want to hijack and replace with our own portal, but how does it work? From a useful [StackOverflow article](http://stackoverflow.com/questions/4454655/can-i-trigger-a-mobile-client-to-automatically-launch-a-web-browser-when-connect):
   
   > Both iOS and android devices will detect for captive portals by simply checking for a standard URI resource (eg: http://www.apple.com/library/test/success.html) and if that resource is blocked then you're offline, if that resource gets 302 or 307 redirected then it assumes there is a captive portal in place and they will open a browser.  
   
So, on iPhone this is the Captive Network Assistant (CNA), and on Android devices this is usually a prompt to 'log in', which then opens in a limited internet browser.

| It's important to recognise that these are not your first-class standards compliant web browsers and will have quirks. Different versions of iOS and Android also test their connections with different URLs and indicate a successful login differently.

### Install SSL cert in lighttpd and confgure HTTPS

From: https://wiki.openwrt.org/doc/howto/owncloud#get_ssl_optional

Upload certificate and set permissions:
chmod 0600 /etc/lighttpd/ssl/my.domain
chmod 0600 /etc/lighttpd/ssl/my.domain/certificate.pem

add to etc//lighttpd/lighttpd.conf:

#Enable ssl on port 443 for https

You'll need to upload the certificate & any certificates from the certificate authority to a folder on your ZSun. 
For easy management, create the folders in the path and upload them to `/etc/lighttpd/ssl/[my.domain]/`. 
Replace `[my.domain]` with the domain name your SSL certificate is signed for.

Next, add the following to your `/etc/lighttpd/lighttpd.conf`:

```
$SERVER["socket"] == ":443" {
    ssl.engine = "enable"
	ssl.pemfile = "/etc/lighttpd/ssl/[my.domain]/[my.certificate].pem"
	#ssl.ca-file = "/etc/lighttpd/ssl/[my.domain]/[my.fullchain].pem"
 }
```

Replace `[my.certificate].pem` with the name of the certificate (pem) file that contain both the public key and private key. 
If you have a bunch of pem files, it's probably the one with two entries.
If you have separate private and public key files, you can cut and paste the two together into one file.

If you need to include the full chain to the Certificate Authority (CA) just uncomment the second line `ssl.ca-file` and replace `[my.fullchain]` with the name of your CA's certificate.

Now, `/etc/init.d/lighttpd restart` to restart lighttpd.

If you get an error `can't resolve symbol 'EC_KEY_new_by_curve_name'` this is caused by a missing function, likely caused by an outdated version of `libopenssl`.
Do a `opkg update libopenssl` to update libopenssl and it should go away.

### Re-route any DNS requests to our web server's IP.   
_Based on information found [here](http://serverfault.com/questions/351108/using-dnsmasq-to-resolve-all-hosts-to-the-same-address)._

This will result in any server request being re-routed to our own web server. 

Configure `dnsmasq` by adding these entrys to `/etc/dnsmasq.conf`:
  
```
address=/#/[YourOpenWRTIpAddress]
local-ttl=0
```

Be sure to replace `[YourOpenWRTIpAddress]` with your OpenWRT gateway ip address, e.g. 192.168.1.1.
   
| Important: The line `local-ttl=0` ensures that the DNS rewrite's time-to-live is zero, which will ensure none of the redirected records are retained. This will prevent DNS poisoning, which is extremely annoying.  
 
Restart lighthttpd for the changes to take effect: `/etc/init.d/lighttpd restart`
 
### Redirect all HTTP to HTTPS requests 
_Based on information found [here](https://redmine.lighttpd.net/projects/lighttpd/wiki/HowToRedirectHttpToHttps)._
 
First, do 'opkg install lighttpd-mod-redirect' to install mod-redirect.

Next, add the following section to `/etc/lighttpd/lighttpd.conf`:

```
#Redirect all HTTP traffic to HTTPS 
$HTTP["scheme"] == "http" {
   # capture vhost name with regex conditiona -> %0 in redirect pattern
   # must be the most inner block to the redirect rule
   $HTTP["host"] =~ ".*" {
       url.redirect = ("(.*)" => "https://my.domain$1")
   }
}
``` 

Restart lighthttpd for the changes to take effect: `/etc/init.d/lighttpd restart`

### Redirect all domains that aren't ours to our own
_Based on information found [here](http://superuser.com/questions/442101/redirect-all-urls-except-one-in-lighttpd)._

Add the following section to '/etc/lighttpd/lighttpd.conf':
 
```
#Redirect all HTTPS traffic to my.domain
$HTTP["scheme"] == "https" {
   # capture vhost name with regex conditiona -> %0 in redirect pattern
   # must be the most inner block to the redirect rule
   $HTTP["host"] !~ "^(www\.)?my\.domain.*$" {
       url.redirect = ("^/(.*)" => "https://my.domain$1")
   }
}
```

Be sure to replace `my.domain` and its constituents in the regex conditions accordingly.

Restart lighthttpd for the changes to take effect: `/etc/init.d/lighttpd restart`
 
### Add a 404 handler to redirect to our portal's root.
 _Based on information found [here](https://redmine.lighttpd.net/projects/1/wiki/Server_error-handler-404Details)._
 
 This will ensure that any url with a subpath that hits our web server from a redirect will arrive at our portal's homepage. 
 
 Set this property in `/etc/lighttpd/lighthttp.conf`:
``` 
server.error-handler-404 = "/" 
```

Restart lighthttpd for the changes to take effect: `/etc/init.d/lighttpd restart`
 
### Testing the captive portal (Enabling/disabling it) 
 
According to [MAX_HOPPER's advice](https://forum.openwrt.org/viewtopic.php?id=63835) in the OpenWRT forums, the dnsmasq re-routing will only work when the router has no WWAN connection - this is because requests get routed through WWAN to an external DNS. You can set up a bypass using iptables - but actually, the easiest way is to just disable the 'wwan' wireless interface.

So basically, you can turn the captive portal on by disabling the `wwan` wireless interface in `/etc/config/wireless` by commenting out the entire `config wifi-iface` block that contains `option network wwan`.  You can turn it off by uncommenting.

Restart lighthttpd for the changes to take effect: `/etc/init.d/lighttpd restart`

## Adding custom Rewrite rules to lighttpd

As you set up your portal & API in PHP, you'll probably need to create a few custom rewrite rules to get everything working smoothly.

To do this, `opkg install lighttpd-mod-rewrite`. You can find docs on it [here](https://redmine.lighttpd.net/projects/lighttpd/wiki/Docs_ModRewrite).

As an example, for a basic configuration that will re-write requests to a php api in a sub-directory:

Add to `/etc/lighttpd/lighttpd.conf`:

```
url.rewrite-once = (
  "^/api/(.*)php(.*)" => "/api/$1php$2",
    "^/api(.*)"  => "/api/index.php$1"        
)
```

Restart lighthttpd for the changes to take effect: `/etc/init.d/lighttpd restart` 

## Making PHP play nice with large file uploads and slow scripts.

In our portal, users could upload pictures of themselves, which sometimes could get quite large. There are options in PHP that let it play nice with large & lengthy requests.
  
In `/etc/php.ini` increase the following options:
```
   default_socket_timeout = 180
   upload_max_filesize = 20M
   max_execution_time = 180
   post_max_size = 0 (disabled)
   max_input_time = -1 (Default disabled)
   memory_limit = 32M
```   
   
The last line allows PHP to take up more mem on the ZSun.

## Other stacks for hosting the portal
This is just one example of how to get a captive portal app working using PHP, although you could set up any platform you want. When I was choosing which platform to use for the web server, I did a bit of digging on the options. Here are some quick answers: 

### Can we get NodeJS on OpenWRT?
 
[In theory, yes](https://wiki.openwrt.org/doc/howto/nodejs) - but it seems that we'd have to do a [custom build](https://github.com/netbeast/docs/wiki/Cross-Compile-Nodejs-for-OpenWrt) of Node to run on our Hackerspace build of OpenWRT.

Remember, we have very tight memory restrictions on the ZSun and basic requirement for Node ask for 256mb. If we were to run Node on the ZSun, we could probably get around the RAM & space restrictions by:
- Setting up a swap file on the SD card to give us more RAM.
- Installing node on the SD card and run it from that.

If you're interested in trying this, you can follow the steps [to cross compile Node](https://forum.openwrt.org/viewtopic.php?pid=263650#p263650) on the OpenWRT forum. 

To do it, you'll probably need a Linux environment, and Cygwin isn't supported by the OpenWRT build environment since its file system is case insensitive.

For me, I lacked the bother, so I opted instead to simply run PHP on Lighttpd which is already available as a package for OpenWRT.
 
### What about LUA?
OpenWRT comes with LUA pre-rolled and LuCi is basically a big LUA app. One option is to write the API as a set of LUA scripts invoked by FastCGI. Webstorm has a LUA plugin and Eclipse also has an officially supported LUA plugin. So yes, you can use LUA but why, when we can run PHP and take advantage of its mass of mods and frameworks?

In the end, I went with a minimal installation of PHP.

## In the end...

So now we have a ZSun devices with a flashed with a functioning build of OpenWRT with the following capabilities:
- SSH & SFTP for terminal & file system access. 
- Bridging to an existing WiFi internet connection.
- LuCi admin console hosted on an alternative port, e.g. 8080.
- Lighttpd running PHP to host our custom portal.
- HTTPS support, hosting our custom portal at a custom domain name.
- Dnsmasq & redirects configured to capture users and send them to our portal automatically when connecting.

If you're interested to see our implementation for Koulu on Fire, you can it at the [koulu.space](https://github.com/Martaver/koulu.space) repo. As an experiment, we used an angular2 front-end and PHP API back-end to create a web-mobile app. 

The only limitation of this configuration, is that the captive portal which automatically opens is usually in a restricted browser. It's surprising how much you can actually achieve in this browser, however it's not a seamless experience for the user. In order to perfect this approach, we should figure out a way to indicate a successful login to iOS or Android and then open up a window in the phone's default web browser.

Any takers?