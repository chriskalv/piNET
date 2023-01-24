piNet
========================
This is a guideline consisting of personal notes for the setup of a Raspberry Pi within a home network as 
  + an automatic adblocker (via <b>piHole</b>) that optionally shows its current status on a nifty TFT display attached to the Raspberry Pi (via <b>PADD</b>),
  + an own Wireguard VPN server for remote access to the home network (via <b>piVPN</b>),
  + an uptime and response tracker for websites and/or clients on the network (via <b>Uptime Kuma</b>),
  + a reverse proxy server to redirect specified domain requests to local IPs on specified ports (via <b>Nginx</b>),
  + a device to encrypt all DNS requests for all clients on the network via HTTPS (via <b>Cloudflared</b>),
  + and an automatic & recurring backup for your entire Raspberry Pi (via <b>raspiBackup</b>).

Needless to say, all of these applications/services can be installed on their own and none of them "requires" the other. However, the installation of the DoH service (<b>Cloudflared</b>) only makes sense in conjunction with the previous installation of <b>piHole</b> in this specific guide. <br>
Furthermore, <b>raspiBackup</b> can be used to back up the microSD of your Raspberry Pi to all kinds of hosts and/or cloud services. In my guide, however, I will only describe the backup configuration to a Synology NAS.

| Finished piNet Device   |
| :-------------: | 
| [![](https://i.imgur.com/Vlfj5FX.jpg?raw=true)](https://i.imgur.com/Vlfj5FX.jpg)   | 

| piNet (Front)   | piNet (Back)   |
| ------------- | -------------|
| [![](https://i.imgur.com/7YVQKyC.jpg?raw=true)](https://i.imgur.com/7YVQKyC.jpg)   |   [![](https://i.imgur.com/QLDGQQx.jpg?raw=true)](https://i.imgur.com/QLDGQQx.jpg)   |

<br></br>

## Hardware
+ [Raspberry Pi 4](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
+ [MicroSD Card](https://www.westerndigital.com/products/memory-cards/sandisk-max-endurance-uhs-i-microsd#SDSQQVR-032G-GN6IA)
+ [Case](https://geekworm.com/collections/raspberry-pi/products/raspberry-pi-4-model-b-armor-aluminum-alloy-case-protective-shell)
+ [Adafruit PiTFT Display](https://www.adafruit.com/product/2441)
+ [GPIO Header with a 90° angle](https://www.voc-electronics.com/a-37420733/gpio-extensions/26-pin-gpio-connector-header-extender-90-degree-angle/#description)

Of course, you can also use a Raspberry Pi 3 or even a Pi Zero with some (minor) disadvantages in performance. Furthermore, the TFT didsplay can alternatively be attached directly onto the Raspberry Pi board without the need of the 90° GPIO header. In that case, however, be aware that there are Pins on the back of the display, which you would need to remove if you plan on using the same case I've linked above. 

<br>

## Setup Process
+ ### Groundwork
1. Install [Raspberry Pi OS Lite](https://www.raspberrypi.com/software/) by using the <b>Raspberry Pi Imager</b>. Since I have a Raspberry Pi 4, I install the 64bit version. With the imager software, you can directly set up your hostname (`piNet`), your user/password credentials, your SSH access and your timezone/keyboard settings before flashing the OS to the microSD card.<br>
2. Once the Raspberry Pi has booted with the SD card inside, you check for the Pi’s IP address in your router’s web interface. There, you should also directly assign a static DHCP lease for it, so your new device always gets assigned the same IP from your router.
3. You can then access the Pi via SSH (with software like <b>Putty</b>). Once you've logged in, define your local WLAN country via `sudo raspi-config` (<i>5</i> --> <i>L4</i> --> <i>Country</i>) and update/upgrade your system with `sudo apt-get update && sudo apt-get upgrade -y`. Ultimately, reboot with `sudo reboot`.
  
<br>

+ ### Install piHole
--> [PiHole](https://github.com/pi-hole/pi-hole) is an ad/tracker blocker that acts as a DNS sinkhole, not responding to blacklisted domain requests.
1. Log in as root with `sudo su`. Then start the automatic install script for piHole with `curl -sSL https://install.pi-hole.net | bash` and go through the installation process.
2. Change the automatically generated password to access the web interface of the piHole with `pihole -a -p`. After that, `reboot`.
3. I like to limit the size of DNS packets to match/undercut the MTU size of my router (1500 in my case). This is done by creating a file with `sudo nano /etc/dnsmasq.d/99-edns.conf` and pasting the line `edns-packet-max=1232` into it. Somehow, anything above that defaults to a packet size of over 4000. Save the file.
4. I also like to include a 5 second delay to the startup process by executing `sudo nano /etc/pihole/pihole-FTL.conf` and adding `DELAY_STARTUP=5` to it in order to avoid occasional errors in the piHole dashboard saying that there is no eth0 device present, as it does not initialize quickly enough.
5. In order to not run into any IP conflicts with our reverse proxy via <b>Nginx</b> later on, change the port piHole is using from 80 to something else (in my case 8017) with `sudo sed -ie 's/= 80/= 8017/g' /etc/lighttpd/lighttpd.conf`. Since this change does not persist through piHole updates, we will later implement a script to have this changed back to our desired port after every update. Save the file and reboot the device with `sudo reboot`.
5. Set up your router to use the piHole's IP as its DNS server. In some cases you also need to restart the router.
6. Access the piHole's web interface within your browser by entering `<IP of Raspberry Pi>:<port of your piHole WebUI>/admin` and configure the available settings to your liking. There are several guides out there to help with configuring piHole properly.
8. Check if ads are being blocked.

<br>

+ ### Install PiTFT display & PADD
--> [PADD](https://github.com/pi-hole/PADD) is used to show the current status of piHole, IPs, CPU/RAM usage etc. on the PiTFT display. 
1. Execute the automatic install script for PiTFT display with:
```
cd ~
sudo apt-get install -y git python3-pip
sudo pip3 install --upgrade adafruit-python-shell click
git clone https://github.com/adafruit/Raspberry-Pi-Installer-Scripts.git
cd Raspberry-Pi-Installer-Scripts
```
2. Execute the initialization of the 480x320 display with:
```
sudo python3 adafruit-pitft.py --display=35r --rotation=270 --install-type=console
```
3. After having rebooted successfully, install PADD with:
```
cd ~
wget -N https://raw.githubusercontent.com/jpmck/PADD/master/padd.sh
```
4. Execute `sudo chmod +x padd.sh` to adjust permissions.
5. Make PADD run automatically on startup by editing `bashrc` with `sudo nano ~/.bashrc` and adding the following to the bottom of the file:
```python
# Run PADD
# If we're on the PiTFT screen (ssh is xterm)
if [ "$TERM" == "linux" ] ; then
  while :
  do
    ./padd.sh
    sleep 1
  done
fi
```
6. Reboot. PADD should now show on startup.

<br>

+ ### Install PiVPN
--> [PiVPN](https://github.com/pivpn/pivpn) lets us dial into our home network remotely via WireGuard.
1. Open port 51820 UDP on your router for the Raspberry Pi.
2. Execute the automatic install script for piVPN with `sudo curl -L https://install.pivpn.io | bash` and go through the installation process for the WireGuard VPN server. Reboot when finished.
3. In order for a VPN client to properly forward IPv4 packets, IPv4 forwarding needs to be enabled. This can be done by executing `sudo nano /etc/sysctl.conf` and uncommenting the line `net.ipv4.ip_forward=1`.
4. Add users for the piVPN service with `pivpn -a`. One user for each client you want to use the VPN with.
5. Since I've experienced cases where my VPN connection via WireGuard was a little unstable, I like to specify the MTU value that piVPN uses to match the MTU value of my router and enable a persistent KeepAlive for my clients. Also, the PostUp and PostDown configurations of iptables need to be defined. Both of these steps can be done by editing `/etc/wireguard/wg0.conf`. In order to do that, log in as root (`sudo su`) and execute `nano /etc/wireguard/wg0.conf` to edit the file like so:
```python
[Interface]
PrivateKey = <PrivateKey>
Address = <Address>
MTU = 1492
ListenPort = <ListeningPort>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
### begin ConfigOne ###
[Peer]
PublicKey = <publickey>
PresharedKey = <presharedkey>
AllowedIPs = <allowedIPs>
PersistentKeepalive = 25
### end ConfigOne ###
### begin ConfigTwo ###
[Peer]
PublicKey = <publickey>
PresharedKey = <presharedkey>
AllowedIPs = <allowedIPs>
PersistentKeepalive = 25
### end ConfigTwo ###
``` 
6. Execute a quick debug, as this solves potential issues with not being able to connect to local devices: `sudo pivpn -d`.
7. Import the config files for said users either by either
  + QR code with `pivpn -qr` for devices with integrated cameras or
  + Config file, which you can retrieve via SCP (with software like <b>WinSCP</b> or <b>FileZilla</b>, for example) from `/configs` on your Raspberry Pi.
8. Connect to your piVPN from an external network and check if your IP changes with help of websites like [whatismyipaddress.com](https://whatismyipaddress.com/).

<br>

+ ### Install Uptime Kuma
--> [Uptime Kuma](https://github.com/louislam/uptime-kuma) enables us to monitor various protocols (TCP, HTTP, PING etc.) and can notify us about changes.
1. First, we will need to install all prerequisites (Git and NodeJS) by executing:
- `sudo apt install git`,
- `curl -fsSL https://deb.nodesource.com/setup_current.x | sudo -E bash -`,
- and `sudo apt install nodejs`.
2. Next, Uptime Kuma and its process manager are going to be installed by executing:
- `git clone https://github.com/louislam/uptime-kuma.git`,
- `cd uptime-kuma && npm run setup`,
- `sudo npm install -g pm2`,
- and `pm2 install pm2-logrotate`.
3. After that, Uptime Kuma is started and configured to run after bootup with:
- `pm2 start server/server.js --name uptime-kuma`,
- `pm2 save`,
- and `pm2 startup`.<br>
The last command will give you a command to execute, which will look like `sudo env PATH=$PATH...`, but might be slightly different for each individual system. Copy the command and execute it.
4. You can now access Uptime Kuma at <IP of your Raspberry Pi>:3001 via your Browser. Add an account and configure Uptime Kuma to monitor the services you'd like to keep an eye on.

<br>

+ ### Install Nginx
--> We will use [Nginx](http://nginx.org/) as a reverse proxy, so we can enter a domain name in our browser and be redirected to a specific IP on a specific port.
1. Install Nginx by executing `sudo apt install nginx`. After the installation, verify that Nginx is running with `sudo systemctl status nginx`.
2. Remove the current default Nginx site with `sudo rm /etc/nginx/sites-available/default` and create a new one with `sudo nano /etc/nginx/sites-available/default`. Paste the following content to redirect <b>piHole</b> and <b>Uptime Kuma</b>, edit it to match your network configuration and save the file. 
```php
server{
        listen 80;
        server_name pihole.arpa;

        location / {
        proxy_pass http://<IP of your Raspberry Pi>:<piHole WebUI port number>/admin/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_hide_header X-Frame-Options;
        proxy_set_header X-Frame-Options "SAMEORIGIN";
        proxy_read_timeout 90;
        }
        location /admin {
        proxy_pass http://<IP of your Raspberry Pi>:<piHole WebUI port number>/admin/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_hide_header X-Frame-Options;
        proxy_set_header X-Frame-Options "SAMEORIGIN";
        proxy_read_timeout 90;
        }
}

server{
        listen 80;
        server_name uptimekuma.arpa;

        location / {
        proxy_pass http://<IP of your Raspberry Pi>:3001;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_hide_header X-Frame-Options;
        proxy_set_header X-Frame-Options "SAMEORIGIN";
        proxy_read_timeout 90;
        }
}


```
4. Restart Nginx with `sudo systemctl restart nginx`. Nginx is now running our new reverse proxy configuration.
5. Next, we need to tell piHole (as it is our default DNS server) to redirect domain requests we would like to reverse proxy (in our example `pihole.arpa` and `uptimekuma.arpa`) to Nginx, so Nginx can redirect these requests instead. In order to do that, head to piHole's WebUI, go to <i>Local DNS</i> --> <i>DNS Records</i> and add new domain/IP combinations consisting of the domain we want to redirect (`pihole.arpa`, for example) and the IP of Nginx (which is the same IP as the Raspberry Pi's IP in our case). 
7. Type in `pihole.arpa` or `uptimekuma.arpa` into your web browser, which should get you to each application's WebUI. You can create additional proxy redirects by adding them to `/etc/nginx/sites-available/default` and setting a Local DNS for them in piHole as well.

<br>
  
+ ### Install Cloudflared (DNS over HTTPS)
--> A more detailed about [Cloudflared](https://github.com/cloudflare/cloudflared) can be found [here](https://nathancatania.com/posts/pihole-dns-doh/).
1. Follow the guide above
2. Check if DNS over HTTPS is active with https://1.1.1.1/help
  
<br>
  
+ ### Install raspiBackup
--> [raspiBackup](https://github.com/framps/raspiBackup) will create a recurring backup of our miroSD to the Synology NAS on our network.
1. Configure your Synology NAS to allow users access to the NFS protocol. In order to do that, log into DSM, go to <i>Control Panel</i> --> <i>File Services</i> --> <i>NFS</i> and activate <i>Enable NFS service</i>. Choose <i>NFS v4.1</i> as the "maximum NFS protocol".
2. Create a new shared folder on your NAS. Right-click it and select <i>Edit</i>. In the <i>NFS Permissions</i> tab, click <i>Create</i> and enter the following:
```
Hostname or IP: <IP of your Raspberry Pi>
Privilige: Read/Write
Squash: No mapping
Security: sys
[v] Enable asynchronus
[v] Allow connections for non-priviliged ports
[ ] Allow users to access mounted subfolders
```
3. SSH into your Raspberry Pi and install NFS Common to read from a NFS shared folder by executing `sudo apt-get install nfs-common`.
4. Create a backup folder with `sudo mkdir /backup`.
5. Enter `sudo nano /etc/fstab` and add the following line to the file:
```
NAS_IP_ADDRESS:/PATH/TO/NAS /PATH/TO/MOUNT/POINT nfs auto 0 0
```
  In my specific case, this line is
```
192.168.2.8:/volume1/backups_LINUX /backup nfs auto 0 0
```
6. Edit permissions accordingly by logging in as root with `sudo -i` and executing `chmod 777 /backup`. Then, make the Raspberry Pi's user ("pi" in my case) the owner of the new folder with `chown pi:pi /backup` and logout of the root user with `exit`.
7. Install raspiBackup with `curl -L https://raspibackup.linux-tips-and-tricks.de/install | sudo bash` and configure everything according to your needs in the installer/configuration UI. Be sure to
- select `rsync` as backup type and
- stop recommended services before the backup process.
8. After that, you can initiate a first backup with `sudo raspiBackup.sh -m detailed`. You can always access the installer/configuration UI again with `sudo raspiBackupInstallUI`.

<br></br>

+ ### Create custom recurring processes
--> We will use crontab to establish routines for application updates, config backups, reboots, and power saving.
1. Open crontab with `sudo crontab -e`. Select `nano` to edit the file, if asked.
2. Add the following lines to the file (you can get some help with crontab time syntax with [crontab.guru](https://crontab.guru/)).
```yaml
# CONFIG BACKUPS
# Back up piHole config at 04:15h every first day of the month:
15 4 1 * * pihole -a -t
# Back up piVPN config at 04:30h every first day of the month:
30 4 1 * * pivpn -bk

# APPLICATION UPDATES
# Update piHole and change the standard port 80 of the WebUI back to custom port 8017 at 04:15h every second day of the month:
15 4 2 * * pihole -up && sed -ie 's/= 80/= 8017/g' /etc/lighttpd/lighttpd.conf
# Update PADD at 04:30 every second day of the month:
30 4 2 * * cd ~ && rm padd.sh && wget -N https://raw.githubusercontent.com/pi-hole/PADD/master/padd.sh && sudo chmod +x padd.sh

# REBOOTS
# Reboot Raspberry Pi at 01:00h every third day of the month
0 1 3 * * /sbin/shutdown -r now

# POWER SAVERS
# Turn off PiTFT screen every day at 03:00h:
0 3 * * * sh -c 'echo "0" > /sys/class/backlight/soc\:backlight/brightness'
# Turn on PiTFT screen every day at 06:00h:
0 6 * * * sh -c 'echo "1" > /sys/class/backlight/soc\:backlight/brightness'
```
  
<b></br>

Congratulations! Your <b>piNet</b> device should now be all set up and ready!
