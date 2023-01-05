# Raspberry Pi Wifi Access Point with VPN and Pi-Hole

This guide will demonstrate how to configure a Raspberry Pi with a usb wifi adapter to act as a wireless access point with VPN and Pi-Hole available to the clients for free.

Essentially when a client connects to this wifi "hotspot" their connection will be routed though a vpn and they will have ad blocking right out of the box. 

This type of network is handy for "smart" devices like smart tvs which don't have any way to install an ad blocker.

This type of network is also handy if your vpn provider only limits you to 5 devices. By connecting to this network the vpn company will think you are using only 1 device no matter how many clients are connected to this wifi. Plus you get ad blocking for free.

![diagram](https://github.com/ArtiomSu/Raspberry-Pi-Wifi-Access-Point-with-VPN-and-Pi-Hole/blob/main/pi-custom-wifi-network.png)

## Equipment used in this guide.
1. Raspberry Pi 2b running Raspbian GNU/Linux 11 (bullseye)
2. Wifi card that supports AP mode and that has linux drivers (ALFA AWUS036NHA) basically anything on Kali Linux recommended forums should work.
3. Ethernet cable and ethernet network with access to the internet.
4. OpenVpn config files (ipvanish)

## Demo and Walkthrough on Youtube
Skip to 05:18 for a demo.

[![Create a Raspberry Pi Wireless Router (Wifi Access Point) with VPN and Pi-Hole | DEMO and GUIDE](https://img.youtube.com/vi/gQ6598jxWJM/0.jpg)](https://www.youtube.com/watch?v=gQ6598jxWJM)

## Guide

1. Connect the wifi card to any of the usb ports and setup your static ip addresses.
You can see your devices with `ifconfig` or `ip link show` you will probably not have easy names like eth0 and wlan0. This is fine I don't either. the eth0 device can show up as something like this `enx111111111` and the wifi device can show up as something like this `wlx011111111`. We will use those for the remainder of the guide.

2. To setup the static ip edit /etc/dhcpcd.conf as root. My ethernet ip will be 10.0.0.12 as my network with internet access is 10.0.0.0/24. The new wifi network can be whatever you want I will go with 69.69.69.0/24 as it is nice.
```
interface enx111111111 
static ip_address=10.0.0.12/24
static routers=10.0.0.1
static domain_name_servers=1.1.1.1

interface wlx011111111 
static ip_address=69.69.69.1/24
nohook wpa_supplicant
```
3. At this point you can reboot your pi so it starts using the static ips.

### Setting up the VPN and Access Point
We will set up the vpn and wifi AP first and then when this is all working we will add on the pi hole support. Just to make testing easier.

1. Update the pi and install openvpn `sudo apt update -y && sudo apt upgrade -y && sudo apt-get install openvpn -y`

2. Save the openvpn configs you want into /etc/openvpn/openvpn_files. Run `sudo mkdir /etc/openvpn/openvpn_files` to create this folder.

#### Auto connect to vpn on startup
This is the easiest way to set this up otherwise you will need to connect manually each time you reboot the pi with `sudo openvpn --config /etc/openvpn/openvpn_files/some_server.ovpn --daemon`

1. edit `/etc/default/openvpn` as root. find `#AUTOSTART="all"` and delete the `#` at the start so its `AUTOSTART="all"`. 

2. copy one of your configs you would like to use by default `sudo cp /etc/openvpn/openvpn_files/some_server.ovpn /etc/openvpn/`

3. edit `/etc/openvpn/openvpn_files/some_server.ovpn` as root and change the line `auth-user-pass` to `auth-user-pass pass`

4. create `/etc/openvpn/pass` as root and on the first line put in your vpn account email and on the second line put in your vpn account password.

5. change permissions to protect password `sudo chmod 400 /etc/openvpn/pass`

6. rename the default config `sudo mv /etc/openvpn/some_server.ovpn /etc/openvpn/client.conf`

7. enable the vpn service and reboot `sudo systemctl enable openvpn@client.service && sudo reboot`

8. you should be connected to the vpn shortly after the pi starts. You can check your ip with `curl ifconfig.me` make sure its the vpn ip.

#### Setup Access Point.
With the vpn auto connecting you should see a new device in `ip link show` called `tun0` this is the vpn connection.

1. make sure you have iptables installed `sudo apt-get install iptables`

2. next few commands will route the wifi connection through the vpn tunnel.
```sh
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
sudo iptables -A FORWARD -i tun0 -o wlx011111111 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlx011111111 -o tun0 -j ACCEPT
```

3. to persist these rules `sudo apt-get install -y iptables-persistent` during the installation, the program will ask you to save the current iptables rules (IPv4 and IPv6). You should acknowledge both prompts.

4. to make the access point `sudo apt install hostapd`

5. to make sure it works on startup `sudo systemctl unmask hostapd && sudo systemctl enable hostapd`

6. to provide dns and dhcp `sudo apt install dnsmasq` (pi-hole will be used later for dns and dhcp but we will use a basic test configuration for now)

7. create `/etc/sysctl.d/routed-ap.conf` as root and put this in 
```
# Enable IPv4 routing
net.ipv4.ip_forward=1
```

8. add another rule and save it `sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE && sudo netfilter-persistent save` 

9. edit `/etc/dnsmasq.conf` as root and temporarily add the following. This will be removed in the pi-hole installation step.
```
conf-dir=/etc/dnsmasq.d

interface=wlx011111111 # Listening interface
dhcp-range=69.69.69.120,69.69.69.250,255.255.255.0,24h
                # Pool of IP addresses served via DHCP
domain=wlan     # Local wireless DNS domain
address=/gw.wlan/69.69.69.1
                # Alias for this router
```

10. edit `/etc/hostapd/hostapd.conf` as root. This is the configuration for the access point. Mine looks like this. Make sure to change the interface, ssid, wpa_passphrase
```
country_code=GB
interface=wlx011111111
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
ssid=FreeInternet
wpa_passphrase=secure_password_123
```

11. reboot the pi. And you should be able to connect to the new access point. make sure that your public ip is the vpn servers ip, and make sure you can actually browse the internet.

### Setting up Pi-Hole
Now that the vpn is successfully working with with the access point, we can add Pi-hole support for ad blocking for all clients that connect to this network.

1. edit `/etc/dnsmasq.conf` as root and uncomment/delete the stuff we did as we will use the pihole for this.
```
conf-dir=/etc/dnsmasq.d

#interface=wlx011111111 # Listening interface
#dhcp-range=69.69.69.120,69.69.69.250,255.255.255.0,24h
                # Pool of IP addresses served via DHCP
#domain=wlan     # Local wireless DNS domain
#address=/gw.wlan/69.69.69.1
                # Alias for this router
```

2. reboot the pi so the new settings apply.

3. install pihole `curl -sSL https://install.pi-hole.net | bash` provide your sudo password.

4. during installation choose the wifi interface

5. during installation select the custom ip option and set 69.69.69.1/24 for the first field and 69.69.69.1 for the second.

6. go through the rest of the installation as normal, enable the default list and you can enable logging if you like.

7. after the installer exists, note your web ui password if you selected to have a webui during install.

8. enable dhcp using pi-hole `sudo pihole -a enabledhcp "69.69.69.120" "69.69.69.250" "69.69.69.1" "24" "local"`

9. change the webui password with `pihole -a -p`

10. reboot the pi and you should be all set.

11. to test if the pihole is working you can either go into the webui or go to this website here `https://d3ward.github.io/toolz/adblock.html` without pi-hole on chrome you should see around 10% blocking. and if pi-hole is working you should see atleast over 50%. In my case it was 70%.

12. also make the vpn is working by checking your public ip address.

## References
1. [Setting up VPN Hotspot](https://medium.com/swlh/make-a-hotspot-of-raspberry-pi-while-using-a-vpn-e8f6620c1ab9)
2. [Auto connecting to vpn on startup](https://raspberrypi.stackexchange.com/questions/136051/connect-to-vpn-network-on-startup)
3. [Setting up Pi-Hole](https://www.crosstalksolutions.com/the-worlds-greatest-pi-hole-and-unbound-tutorial-2023/#Install_Pi-hole)
