# Remotely Controlled Darts Scoring

<p align="center">
  <img src="https://miklos-szel.github.io/rpi-darts-kiosk/imgs/final.png" />
</p>

# Hardware requirements
* Raspberry Pi model 3 or newer recommended
* 4GB or bigger SD card 
* HDMI to HDMI or HDMI to DVI cable (Raspberry 3)
* TV or monitor
* Scorekeeping:
  * Bluetooth Numpad (Microsoft Wireless Number Pad) or
  * 2.4ghz wireless keyboard(works out of the box) or
  * Physical USB keyboard
* Navigation (navigating menus,stats)
  * Wireless bluetooth mouse or
  * 2.4ghz wireless mouse (works out of the box) or
  * Physical USB mouse
* OPTIONAL: vnc (remote control software, can act as a mouse or keyboard)

The final system boots withing a minute with all the optional settings enabled. 

# Install Raspberry Pi OS (bullseye)

Download and install the [Raspberry Imager.](https://github.com/raspberrypi/rpi-imager/releases/tag/v1.6.2)

From the Operating System menu, choose: Raspberry OS Other -> **Raspberry Pi OS lite(32-BIT)**.

Before writing the image use the Advanced options in the Raspberry Pi Imager to configure the network and enable ssh.
Press [CTRL+SHIFT+X](https://youtu.be/om8gGB3gyT0?t=17) then make sure that 'enable ssh' and the 'Configure Wifi' sections are filled as seen below:

<p align="center">
  <img src="https://miklos-szel.github.io/rpi-darts-kiosk/imgs/rpi_imager_advanced_menu.png" height="800" />
</p>

# Setup the system

Determine the ip address of the raspberry machine nmap:
```
nmap -sn 192.168.1.0/24
```

Log in with user pi:

`ssh pi@ip_of_the_rpi`

### Upgrade the system to the latest:
``` 
sudo apt-get update && sudo apt-get -y upgrade 
```
### Install the browser and its dependencies
```
sudo apt-get install -y  --no-install-recommends xserver-xorg x11-xserver-utils xinit openbox chromium 
```

### Enable auto login with user pi
```
sudo raspi-config
```

`1. System Options ->  S5 Boot / Auto Login ->  B2 Console Autologin`

(you can defer the reboot for now)

### Configure the browser to start in kiosk mode:

change the `/etc/xdg/openbox/autostart` to the following:
```
URL1="https://app.kingofdarts.com/lets-play-darts"
URL2="https://lidarts.org/game/create"
URL3="http://www.pro-darter.com/dashboard.php"

xset -dpms     # disable DPMS (Energy Star) features.
xset s off     # disable screen saver
xset s noblank # don't blank the video device

unclutter &    # hide X mouse cursor unless mouse activated
setxkbmap -option terminate:ctrl_alt_bksp


/usr/bin/sed -i 's/"exited_cleanly":false/"exited_cleanly":true/' /home/pi/.config/chromium/'Local State'
/usr/bin/sed -i 's/"exited_cleanly":false/"exited_cleanly":true/; s/"exit_type":"[^"]\+"/"exit_type":"Normal"/' /home/pi/.config/chromium/Default/Preferences



chromium --display=:0 --kiosk --noerrdialogs --disable-translate --no-first-run --disable-infobars --disk-cache-dir=/dev/null  --password-store=basic --disable-pinch --overscroll-history-navigation=disabled  --check-for-update-interval=31536000 --force-device-scale-factor=1.9 --fast-start --fast --window-position=0,0 $URL1

```

`--force-device-scale-factor=1.9`  is based on my personal preferences and monitor size(4x3) you can change it here  or with CTRL+SHIFT + or - in the browser (or in vnc)

`--force-device-scale-factor=1.2` is recommended for 4x3 monitors when using Pro Darter.


Append the following to the `/home/pi/.bashrc` 
```
[[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && startx --
```

The system is ready at this point! 

### OPTIONAL: Install bluetooth keypad or mouse
```
sudo bluetoothctl
agent on
default-agent
scan on

scan off
pair xx:xx:xx:xx:xx
(enter the Passkey on the keyboard and press enter)
trust xx:xx:xx:xx:xx
quit
```
The Microsoft Wireless Number Pad works perfectly for scorekeeping. 
<p align="center">
  <img src="https://miklos-szel.github.io/rpi-darts-kiosk/imgs/keypad.png" />
</p>

Warning: The keyboard won't send any output to your ssh terminal. 


### OPTIONAL: Enable vnc server (remote control keyboard/mouse)

```
sudo apt-get install -y --no-install-recommends realvnc-vnc-server
sudo systemctl enable vncserver-x11-serviced.service
sudo systemctl start vncserver-x11-serviced.service 
```

Install `VNC viewer - Remote Desktop` to your phone, you can connect with user: pi. 
<p align="center">
  <img src="https://miklos-szel.github.io/rpi-darts-kiosk/imgs/vnc.png" height="400" />
</p>



### OPTIONAL: Make the boot screen less verbose
`/boot/cmdline.txt`: add to the end of the first line `logo.nologo loglevel=0 consoleblank=1 silent quiet nosplash  vt.global_cursor_default=0`

```
touch /home/pi/.hushlogin
```

### OPTIONAL Add a nice splash screen
```
sudo apt-get install -y fbi
```

Download the logo for 16x9 screens:
```
sudo wget  https://github.com/miklos-szel/rpi-darts-kiosk/raw/gh-pages/imgs/king_of_darts.png -O /opt/splash.png
```
or for 4x3 screens:
```
sudo wget  https://github.com/miklos-szel/rpi-darts-kiosk/raw/gh-pages/imgs/king_of_darts_4_3.png -O /opt/splash.png
```


Create the following file
```
/etc/systemd/system/splashscreen.service
```

Add: 
```
[Unit]
Description=Splash screen
DefaultDependencies=no
After=local-fs.target

[Service]
ExecStart=/usr/bin/fbi -d /dev/fb0 --noverbose -a /opt/splash.png
StandardInput=tty
StandardOutput=tty

[Install]
WantedBy=sysinit.target
```

Enable the service
```
sudo systemctl enable splashscreen.service
```


### OPTIONAL: Make the system read only
I highly recommend to make all the changes you want before doing this. You might want to log in to homepage with your user first and optionally save the credentials in the chromium-browser so only a mouse will be needed later to start a match (slightly unsecure)
Scoring can be completely done with the numberic keyboard.

#### Disable the swap
```
sudo apt-get purge -y dphys-swapfile
sudo apt-get autoremove -y 
```


```
sudo raspi-config
```
`4 Performance Options - >  P3 Overlay File System (enable)`

Would you like the boot partition to be write-protected? **yes**

If you want to make changes, log in, disable the overlay file system, reboot, make the changes and enable the overlay file system again.


#### OPTIONAL: smart socket to remotely turn the system on/off :)
This could be done in different ways but using an [ikea smart socket](https://www.ikea.com/gb/en/p/tradfri-wireless-control-outlet-00364477/) is super convenient for me. Only turn the Raspberry Pi off this way if the read only file system is enabled.




Useful urls: 

- https://itnext.io/raspberry-pi-read-only-kiosk-mode-the-complete-tutorial-for-2021-58a860474215
- https://kingofdarts.com/
- http://www.pro-darter.com/
- https://lidarts.org/

