# OpenWRT Lucy HTTPS OpenSSL Guide on Windows
Assistance for establishing a ssl connection on openwrt lucy using openssl and windows 10. This is my way on how to create and use ssl certificates. No guarantee this will work for you, I'm not an expert either! I won't give any support, so if you run into trouble, take your time to figure out what has to be done.

## Prerequisite
* Windows 10 (I'm using Pro Build 1909) client
* Router with OpenWRT (I'm using OpenWrt 18.06.2 r7676-cddd7b4c77 / LuCI openwrt-18.06 branch (git-19.051.55698-76cf653) on TP-Link TL-WR1043N/ND v1
* libopenssl, luci-ssl (packages installed via opkg install) and uhttpd 
* PuTTY
* OpenSSL

## Getting started

__Router preparations__:

Make sure the required packages and software is installed (on your windows machine as well as on your router). If you didn't have already, define a hostname in your Lucy web interface (System > System Properties > General Settings > Hostname) and choose a local domain name for your server (Network > DHCP and DNS > Server Settings > General Settings > Local server + Local domain) (make sure your domain suffix matches your local server domain specification) (see images)

You also need to edit your /etc/config/uhttpd file and restart uhttpd afterwards to enable HTTPS. I have uploaded my uhttpd config file for comparison. Here you can also set the path to your server.key and server.crt (e.g. /etc/uhttpd.key and /etc/uhttpd.crt) which will be used to complete the chain.

__Client preparations__:

After installing OpenSSL on your windows machine, I highly recommend you to tweak your windows explorer context menu by adding the "open command prompt window here as administrator" (manual and registry file can be found here https://www.tenforums.com/tutorials/59686-open-command-window-here-administrator-add-windows-10-a.html). This is necessary because you need to operate in the openssl installation directory and openssl has the bad habit of getting a hiccup in the current command prompt window after issuing some commands so that you have to re-open the prompt in order to get the job done).



