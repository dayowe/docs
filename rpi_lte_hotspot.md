### Turning Your Raspberry Pi into a Mobile Hotspot: A Step-by-Step Guide

#### Prerequisites

- **Raspberry Pi 3b or 4b** and the following hardware components:
  - Waveshare SIM7600 3G/4G/LTE Hat
  - Two Antennas - Necessary for cellular connectivity.
  - Two RF Cables - To connect the antennas to the HAT board.
  - USB Cable - For power or data connection between the Raspberry Pi and the HAT board.
  - WD My Passport Ultra 5TB HDD - For network attaches storage (optional)

- **Operating System**: Raspberry Pi OS - Buster

### Step-by-Step Instructions

#### Step 1: Update Your System

Start by updating your Raspberry Pi packages to ensure you have the latest software:

```sh
sudo apt update && sudo apt upgrade -y
```

#### Step 2: Install Required Packages

Install the necessary packages for bridging, DHCP, DNS, and modem utilities:

```sh
sudo apt install bridge-utils dnsmasq hostapd libqmi-utils resolvconf samba udhcpc -y
```

#### Step 3: Enable Modem Initialization

Create a script to initialize the SIM7600 modem. This script sets the modem to online mode and starts the network connection:

```sh
sudo nano /bin/qmistart
```

Add the following lines to the script:

[replace YOUR_PROVIDER_APN with the actual value]

```sh
#!/bin/sh

qmicli -d /dev/cdc-wdm0 --dms-set-operating-mode=online
qmicli -d /dev/cdc-wdm0 -E raw-ip
qmicli -d /dev/cdc-wdm0 -p --wds-start-network="ip-type=4,apn=YOUR_PROVIDER_APN" --client-no-release-cid
udhcpc -i wwan0
```

Make the script executable:

```sh
sudo chmod 755 /bin/qmistart
```

Enable `resolvconf` to handle DNS resolution:

```sh
sudo systemctl enable resolvconf
```

#### Step 4: Create a Systemd Service for Modem Initialization

Create a systemd service to run the script at boot:

```sh
sudo nano /etc/systemd/system/init_modem.service
```

Add the following content:

```ini
[Unit]
Description=Initialize SIM7600 Modem
After=network.target

[Service]
ExecStart=/bin/qmistart
RemainAfterExit=yes
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```sh
sudo systemctl enable init_modem.service
sudo systemctl start init_modem.service
```

This ensures the modem initialization script runs automatically at boot, making it ready for use every time your Raspberry Pi starts.

#### Step 5: Configure the Wi-Fi Access Point

Unmask and enable the hostapd service:

```sh
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```

Edit the `hostapd` configuration file:

```sh
sudo nano /etc/hostapd/hostapd.conf
```

Add the following configuration:

[replace `YOUR_COUNTRY_CODE`, `YOUR_NETWORK_NAME` and `YOUR_PASSWORD` with the actual values]

```plaintext
interface=wlan0
country_code=YOUR_COUNTRY_CODE
ieee80211ac=1
wmm_enabled=1
hw_mode=a
channel=36
ht_capab=[HT40+]
ssid=YOUR_NETWORK_NAME
wpa_passphrase=YOUR_PASSWORD
wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
bridge=br0
```

This step sets up the Raspberry Pi as a Wi-Fi access point, allowing other devices to connect to it.

#### Step 6: Set Up Network Interfaces

Configure the network interfaces to create a bridge between the Wi-Fi and Ethernet interfaces:

```sh
sudo nano /etc/network/interfaces
```

Add the following configuration:

```plaintext
allow-hotplug br0
iface br0 inet static
address 192.168.0.1
bridge_ports wlan0 eth0
```

#### Step 7: Configure DHCP and DNS

Edit the `dnsmasq` configuration file to set up DHCP for the bridged interface:

```sh
sudo nano /etc/dnsmasq.conf
```

Add the following configuration:

```plaintext
interface=br0
dhcp-range=192.168.0.2,192.168.0.10,24h
```

This step configures the DHCP server to assign IP addresses to devices connecting to the Wi-Fi network.

#### Step 8: Enable IP Forwarding

Enable IP forwarding to allow traffic to pass between interfaces:

```sh
sudo nano /etc/sysctl.conf
```

Uncomment or add the following line:

```plaintext
net.ipv4.ip_forward=1
```

IP forwarding is crucial for routing traffic between the Wi-Fi and LTE interfaces.

#### Step 9: Configure iptables for NAT and Security

Set up iptables rules for NAT and secure the network:

```sh
sudo iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -i wlan0 -o wwan0 -j ACCEPT
```

Save the iptables rules to persist them across reboots:

```sh
sudo sh -c 'iptables-save > /etc/iptables.rules'
sudo nano /etc/rc.local
```

Add the following line before `exit 0`:

```sh
iptables-restore < /etc/iptables.rules
```

These rules ensure that only established connections and SSH from the local subnet are allowed, securing your network.

#### Step 10: Network Attached Storage (Optional)

Configure Samba to share external storage over the network:

```sh
sudo nano /etc/samba/smb.conf
```

Add the following configuration:

```plaintext
[External Storage]
path = /media/pi
public = yes
writeable = yes
```

Create the shared directory and set permissions:

```sh
sudo mkdir /media/pi
sudo chmod -R 777 /media/pi
```

This allows you to share files over the network using Samba.

#### Step 11: Reboot and Verify

Reboot the Raspberry Pi to apply all configurations:

```sh
sudo reboot
```

After the reboot, verify the modem state and IP address:

```sh
ifconfig wwan0
```

Test internet connectivity by connecting a client device to the Wi-Fi network and pinging a public IP address or domain name:

```sh
ping 8.8.8.8
ping google.com
```
