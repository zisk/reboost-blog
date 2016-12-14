Title: Kiosk Display on Raspbian Jessie
Date: 2015-12-07 10:00

I was recently testing the idea of using a Raspberry Pi to run a [Dashing](http://dashing.io/) dashboard on a TV. I found several tutorials for kiosk style displays using Raspbian Wheezy but had some trouble getting everything to work on Jessie. I was eventually able to get everything operational with only a few tweaks.

### Setup
Follow the standard procedure to get Raspbian on the Pi's SD card. I used the full [Raspbian image](https://www.raspberrypi.org/downloads/raspbian/) since I would need a windows manager in order to launch Chrome but a lighter setup is probably possible with the "lite" distribution. 

After your Raspberry Pi is booted, SSH in. First, make sure everything is up to date:

`sudo apt-get update && sudo apt-get -y upgrade`

Then run `raspi-config`. Do some of the standard system configuration:

1. Expand the file system from the main menu.
2. Change the pi user's password. 
3. Disable overscan from the advanced menu. This will help ensure the display fills the entire screen on the TV. 
4. If not already enabled change the default boot option to launch to GUI and login as the pi user. 

Restart.

Install unclutter which hides the mouse cursor. 

`sudo apt-get install unclutter`

### Chromium

One roadblock I ran into is the fact that Chromium (the open source project Google Chrome is based on) is no longer in the official repos for Raspberry Pi. It seems as though the project stopped auto-generating ARM builds at some point. I saw some suggestions to use `iceweasel`, the Debian fork of Firefox, but it lacks the ability to specify launch options. All of the suggested methods of getting it to run full screen, typically using plugins, seemed hacky. I wanted to be able to configure everything from SSH if possible.

Luckily I found [this](http://conoroneill.net/running-the-latest-chromium-45-on-debian-jessie-on-your-raspberry-pi-2/) blog post that points out that the Ubunutu folks have been generating compatible builds. You will need to grab three packages from the Launchpad repos: [libgcrypt11](https://launchpad.net/ubuntu/trusty/armhf/libgcrypt11), [Chromium](https://launchpad.net/ubuntu/trusty/armhf/chromium-browser), and [Chromium codecs](https://launchpad.net/ubuntu/trusty/armhf/chromium-codecs-ffmpeg-extra). Click on the latest build number and find the link for the deb file then `wget` them down to the Pi and install. For example:

```language-bash
wget http://launchpadlibrarian.net/201290259/libgcrypt11_1.5.3-2ubuntu4.2_armhf.deb
wget http://launchpadlibrarian.net/219267135/chromium-codecs-ffmpeg-extra_45.0.2454.101-0ubuntu0.14.04.1.1099_armhf.deb
wget http://launchpadlibrarian.net/219267133/chromium-browser_45.0.2454.101-0ubuntu0.14.04.1.1099_armhf.deb

sudo dpkg -i libgcrypt11_1.5.3-2ubuntu4.2_armhf.deb
sudo dpkg -i chromium-codecs-ffmpeg-extra_45.0.2454.101-0ubuntu0.14.04.1.1099_armhf.deb
sudo dpkg -i chromium-browser_45.0.2454.101-0ubuntu0.14.04.1.1099_armhf.deb
```

### Autostart Settings

Raspbian uses [LXDE](http://lxde.org/lxde/) for its desktop environment. You can set application to automatically launch right from the autostart config file. Many of the Wheezy tutorials I found suggested modifying the global file but that does not appear to work in Jessie. Instead modify the file found in the pi user's home directory:

```language-bash
vi /home/pi/.config/lxsession/LXDE-pi/autostart
```

Comment out the following line to disable the screensaver by adding a `#` to the beginning:

`@xscreensaver -no-splash`

Under that line add the following [xset](http://www.x.org/archive/X11R7.5/doc/man/man1/xset.1.html) options to disable some of the power saving settings and ensure the screensaver is truly off:

```
@xset s off
@xset s noblank
@xset -dpms
```

Now, add a line to launch Chromium in kiosk mode. Obviously you'll want to fill in your own URL at the end:

`@chromium-browser --noerrdialogs --kiosk --incognito https://news.google.com`

Now reboot and you should see the desktop auto-launch followed shortly by a full screen Chromium window.

---
#### Additional Sources
https://www.danpurdy.co.uk/web-development/raspberry-pi-kiosk-screen-tutorial/
https://www.raspberrypi.org/forums/viewtopic.php?f=36&t=122444
http://toxaq.com/index.php/2012/12/running-shopifys-dashing-on-a-raspberry-pi/
