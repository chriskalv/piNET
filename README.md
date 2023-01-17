piNet Setup
========================
This is a guideline consisting of personal notes for the setup of a Raspberry Pi within my home network as 
  + an automatic adblocker (via <b>piHole</b>),
  + an own Wireguard VPN server for remote access to my home network (via <b>piVPN</b>),
  + and a device to encrypt DNS requests via HTTPS (via <b>Cloudflared</b>) for all clients on my network.
  
Also, the device is equipped with a TFT display to show the current status of piHole, as well as CPU/RAM load etc. (via <b>PADD</b>).

## Hardware
+ [Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
+ [MicroSD Card](https://www.westerndigital.com/products/memory-cards/sandisk-max-endurance-uhs-i-microsd#SDSQQVR-032G-GN6IA)
+ [Case](https://geekworm.com/collections/raspberry-pi/products/raspberry-pi-4-model-b-armor-aluminum-alloy-case-protective-shell)
+ [Adafruit PiTFT Display](https://www.adafruit.com/product/2441)
+ [GPIO Header with a 90° angle](https://www.voc-electronics.com/a-37420733/gpio-extensions/26-pin-gpio-connector-header-extender-90-degree-angle/#description)

Of course, you can also use a Raspberry Pi 3 or even a Pi Zero with some (minor) disadvantages in performance. Regarding the TFT display, you can also attach it directly onto the Raspberry Pi board without the need of the 90° GPIO header, if you like. Be aware, however, that there are Pins on the back of the display, which you would need to remove if you plan on using the case I've linked above.

<br>

## Installation Process
Install [Raspberry Pi OS](https://www.raspberrypi.com/software/) by using the <b>Raspberry Pi Imager</b>. With it, you can directly set up your hostname (`pinet`), your user/password credentials, your SSH access and your timezone/keyboard settings before flashing the OS to the microSD card.<br>
Once the Raspberry Pi has booted, check for its IP within your router UI. There, you should directly assign a static DHCP lease for the device, so it always gets assigned the same IP from your router. Once that's done, reboot the Pi.<br>
Then, establish a connection via SSH and update/upgrade your system with `sudo apt-get update && sudo apt-get upgrade -y` and reboot with `sudo reboot`.

<br>

+ ### Install piHole
--> A more detailed guide can be found [here](https://www.smarthomebeginner.com/pi-hole-setup-guide/).
1. Start the automatic install script for piHole with `sudo curl -sSL https://install.pi-hole.net | bash` and go through the installation process.
2. Change the automatically generated password to access the web interface of the piHole with `pihole -a -p`.
3. I like to limit the size of DNS packets to match the MTU size of my router. This is done by creating a file like `/etc/dnsmasq.d/99-edns.conf` and pasting the line `edns-packet-max=1492` (if the MTU size of your router is 1492) into it. After saving the file, restart DNS services with `pihole restartdns`.
5. Set up your router to use the piHole's IP as its DNS server.
6. Access the piHole's web interface within your browser by entering `<IP of your piHole>/admin` and configure the available settings to your liking.
7. I also like to edit `/etc/pihole/pihole-FTL.conf` and include a 5 second delay to the startup process by adding `DELAY_STARTUP=5` to avoid occasional errors in the piHole dashboard saying that there is no eth0 device present. 
8. Check if ads are being blocked in your pihole dashboard.
  
<br>
  
+ ### Install PiVPN
--> A more detailed guide can be found [here](https://www.crosstalksolutions.com/pivpn-wireguard-complete-setup-2022).
1. Open port 51820 UDP on your router.
2. Execute the automatic install script for piVPN with `sudo curl -L https://install.pivpn.io | bash` and go through the installation process.
3. In order for a VPN client to properly forward IPv4 packets, IPv4 forwarding needs to be enabled. This can be done by executing `sudo nano /etc/sysctl.conf` and uncommenting the line `net.ipv4.ip_forward=1`.
4. Since I've experienced cases where my VPN connection via WireGuard was a little unstable, I like to specify the MTU value that piVPN uses to match the MTU value of my router. Also, the PostUp and PostDown configurations of iptables need to be defined. Both of these steps can be done by editing `/etc/wireguard/wg0.conf`. In order to do that, log in as root (`sudo su`) and execute `nano /etc/wireguard/wg0.conf` to edit the [Interface] section of the file:
```python
[Interface]
PrivateKey = <PeerPrivateKey>
Address = <PeerAddress>
ListenPort = <YourListeningPort>
MTU = 1492
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables ->
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables>
``` 
5. Add users for the piVPN service with `pivpn -a`.
6. Execute a quick debug, as this solves potential issues with not being able to connect to local devices `sudo pivpn -d`.
7. Import the config files for said users either by either
  + QR code with `pivpn -qr` for devices with integrated cameras, like cellphones or
  + Config file, which you can retrieve via SCP (with <b>WinSCP</b> or <b>FileZilla</b>, for example) from `/configs` on your Raspberry Pi.
8. Connect to your piVPN from an external network and check if your IP changes with help of pages like [whatismyipaddress.com](https://whatismyipaddress.com/).

<br>
  
+ ### Install Cloudflared (DNS over HTTPS)
--> A more detailed guide can be found [here](https://nathancatania.com/posts/pihole-dns-doh/).
1. Follow the guide above
2. Check if DNS over HTTPS is active with https://1.1.1.1/help
  
<br>

+ ### Install piTFT display
--> A more detailed guide can be found [here](https://learn.adafruit.com/pi-hole-ad-pitft-tft-detection-display/pitft-configuration).
1. Execute the automatic install script with 
```
cd ~
sudo apt-get install -y git python3-pip
sudo pip3 install --upgrade adafruit-python-shell click
git clone https://github.com/adafruit/Raspberry-Pi-Installer-Scripts.git
cd Raspberry-Pi-Installer-Scripts
```
2. Execute initialization of the 480x320 display with
```
sudo python3 adafruit-pitft.py --display=35r --rotation=270 --install-type=console
```

<br>
  
+ ### Establish recurring processes for backups, updates etc.
1. Open crontab with `sudo crontab -e`.
2. Add the following lines to the file (you can get some help with crontab time syntax with [crontab.guru](https://crontab.guru/)).
```yaml
# BACKUPS
# Back up piVPN at 04:05h every first day of the month
5 4 1 * * pivpn -bk
# Back up piHole at 05:05h every first day of the month
5 5 1 * * pihole -a -t

# UPDATES
# Update piHole at 04:05h every second day of the month
5 4 2 * * pihole -up
# Update PADD at 05:05 every second day of the month
5 5 2 * * cd ~ && rm padd.sh && wget -N https://raw.githubusercontent.com/pi-hole/PADD/master/padd.sh && sudo chmod +x padd.sh

# REBOOT
# Reboot Raspberry Pi at 04:25h every third day of the month
5 4 2 * * /sbin/shutdown -r now

# POWER SAVERS
# Turn off PiTFT screen at 03:00h
0 3 * * * sh -c 'echo "0" > /sys/class/backlight/soc\:backlight/brightness'
# Turn on PiTFT screen at 06:00h
0 6 * * * sh -c 'echo "1" > /sys/class/backlight/soc\:backlight/brightness'
```
  
<b></br>
