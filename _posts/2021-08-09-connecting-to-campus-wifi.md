---
layout: post
title: " "
description: 
author: Cedar Warman
date: 2020-08-09
hero_image: /img/blog/2021-08-09-raspberry-pi-zero.jpg
image: /img/blog/2021-08-09-raspberry-pi-zero.jpg
tags: raspberry-pi raspberry-pi-os linux wifi university-of-arizona
canonical_url: https://www.cedarwarman.com/2020/08/09/connecting-to-campus-wifi.html
---

<p class="title is-2">Connecting a headless Raspberry Pi to University of Arizona wifi (and fixing time)</p>

<hr>


I have a lot of <a href="https://www.raspberrypi.org/products/raspberry-pi-zero-w/">Raspberry Pi Zeros</a>. Most of them are spread across various basement growth chambers at the University of Arizona. They run sensors that monitor heat and humidity inside the chambers and upload the data to the cloud for visualization in a <a href="https://shiny.rstudio.com/">Shiny</a> app. To keep costs down, I run the Pi’s headless, meaning no monitor, mouse, or keyboard. This works fine at home, where I can easily add the network name and password to the `/etc/wpa_supplicant/wpa_supplicant.conf` file by directly editing it on the SD card before I plug it into the Pi, as described in the <a href="https://www.raspberrypi.org/documentation/configuration/wireless/headless.md">Raspberry Pi docs</a>:

```bash
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
country=US
update_config=1

network={
 ssid="<Name of your wireless LAN>"
 psk="<Password for your wireless LAN>"
}
```

However, when I try to do the same thing on campus, I quickly run into problems. When you connect to the University of Arizona wifi you have to click through a series of screens where you enter your username and password. This doesn’t work on a headless Pi. Fortunately, with the right configuration files you can still get through. I got the following network configurations from campus IT: “Phase II authentication: MCSCHAPV2, EAP method: PEAP, Certificate: do not validate, Anonymous Identity: leave blank.” Combining this info with directions from <a href="https://www.miskatonic.org/2019/04/24/networkingpi/">this blog post</a> allowed me to complete the configuration.

First, edit `/etc/wpa_supplicant/wpa_supplicant.conf`file as follows:

```bash
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US

network={
        ssid="UAWiFi"
        priority=1
        proto=RSN
        key_mgmt=WPA-EAP
        pairwise=CCMP
        auth_alg=OPEN
        eap=PEAP
        identity="insert_username_here"
        password=hash:insert_hashed_password_here
        phase1="peaplabel=0"
        phase2="auth=MSCHAPV2"
}
```

Note that this time we’ll be responsible and not store our password in plain text. To hash your password, use this command:

```bash
echo -n 'password_in_plaintext' | iconv -t utf16le | openssl md4
```

Next, edit one more file, `/etc/network/interfaces`. If you don’t have this file, you’ll need to create one. The file should read:

```bash
# interfaces(5) file used by ifup(8) and ifdown(8)

# Please note that this file is written to be used with dhcpcd
# For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'

# Include files from /etc/network/interfaces.d:
source-directory /etc/network/interfaces.d

auto lo

iface lo inet loopback
iface eth0 inet dhcp

allow-hotplug wlan0

iface wlan0 inet dhcp
        pre-up wpa_supplicant -B -Dwext -i wlan0 -c/etc/wpa_supplicant/wpa_supplicant.conf
        post-down killall -q wpa_supplicant
```

And that’s it! Except for one small thing. It looks like the University of Arizona has decided to block whichever port Network Time Protocol (NTP) uses (I think port 123), causing the Raspberry Pi to not know what time it is. You can check this after you connect your Pi to the network by printing the current time with `date` and the checking status of NTP with `sudo systemctl status --no-pager systemd-timesyncd`. This can be a big problem if you are, for example, recording the system time along with your sensor readings. To get around this, I installed `htpdate` with the following command:

```bash
sudo apt-get install htpdate
```

`htpdate` will <a href="https://github.com/angeloc/htpdate">“synchronize your computer's time by extracting timestamps from HTTP headers found in web server responses,”</a> eliminating the need to access special ports. The downside is that it’s not as accurate as NTP, but the <1 second accuracy is good enough for my projects.

In writing this post I came across <a href="https://uarizona.service-now.com/sp?id=sc_cat_item&sys_id=d902391ddb3728109627d90d689619d8">this page</a> from the University of Arizona. They describe a `wpa_supplicant` configuration for Raspberry Pi/Linux that might also work. Full disclosure: I’m not even close to being an expert at these things. I have tested my method using ​​Raspberry Pi OS Lite kernel version 5.10. Good luck!
