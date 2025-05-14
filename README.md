# Raspberry Pi as Mini Firewall + NAT + WoL Gateway for Tailscale RDP Access  
This project demonstrates how to use a **Raspberry Pi** as a **firewall and NAT gateway**, allowing a **MacBook to remotely wake and access a Windows PC** via **Tailscale**, while routing all PC internet traffic through the Pi. The setup is designed to work even on captive portal Wi-Fi networks via a travel router.

---

## 🎯 Objective  
To configure a **remote-accessible NAT gateway firewall** using a Raspberry Pi that routes a Windows PC’s traffic through it, and supports RDP access via Tailscale — even in restricted network environments like apartments or hotels.

---

## 🔧 Hardware Needed  

| Component | Details |
|----------|---------|
| Raspberry Pi 3B+ or 4 | With Pi OS Lite |
| 16GB+ microSD card | Used for OS image |
| USB-to-Ethernet Adapter | For second NIC |
| Ethernet Cables (x2) | Router → Pi, Pi → PC |
| Windows PC | Target for RDP |
| MacBook | Remote control device |
| GL.iNet Travel Router | Provides internet bypassing captive portal |

---

## Step 1: Install Raspberry Pi OS Lite  

1. Download Raspberry Pi Imager from [RaspberryPi.com](https://www.raspberrypi.com/software/)
2. Select OS: `Raspberry Pi OS Lite (64-bit)`
3. Set custom options:
   - Hostname: `Raspberrypi`
   - User: `User` 
   - Enable SSH and set Wi-Fi (if wanted, I will connect later)
4. Flash SD card
5. Eject and insert into Pi. Power it on with monitor + keyboard or SSH from mac.

---

## Step 2: Set Up Tailscale on Raspberry Pi  

> ⚠️ Use unrestricted Wi-Fi or a **travel router** if on captive portal.
> If you have unrestrictive wifi then no need for a travel router.

Configure Travel Router (GL.iNet) on your Phone or Mac:
1. Connect to router’s Wi-Fi: `GL-XXXX`
2. Visit: [http://192.168.8.1](http://192.168.8.1)
3. Change default pass to a strong pass (default pass is goodlife for mine)
4. Go to: `Internet > Repeater` make sure to enable repeater
5. Connect to apartment SSID and log in through captive portal

I personally connected my main Wifi router Ethernet to the WAN port on my Travel router, then connect the Pi to a travel router LAN port via another ethernet, then use the USB -> Ethernet Adapter to connect the Pi to PC (USB to Pi then Eth to PC). REMEMBER you still need to do the steps above even if using Ethernet to your Travel router.

On Pi (via mac SSH or monitor+keyboard+terminal):
```bash
sudo raspi-config  # Connect to travel routers SSID (System Options > Wireless LAN)
```
No need to do ⬆︎ step if using Ethernet from Travel router to Pi.
```bash
sudo apt update && sudo apt upgrade -y
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscaled &
sudo tailscale up
tailscale ip -4  # Confirm 100.x.x.x IP
sudo apt install net-tools iptables iproute2 curl nano -y
```

### Add ACLs in Tailscale Admin Panel:
```bash
{
  "acls": [
    {
      "action": "accept",
      "src": ["Mario-CyberS@github"],
      "dst": ["tag:home:3389"]
    },
    {
      "action": "accept",
      "src": ["Mario-CyberS@github"],
      "dst": ["tag:relay-node:*"]
    }
  ],
  "tagOwners": {
    "tag:home": ["Mario-CyberS@github"],
    "tag:relay-node": ["Mario-CyberS@github"]
  }
}
```
Then go to the machines section and add the relay-node tag to your pi device. 
- Click the 3 dots ⋮ next to your Pi
- Click Edit tags
- Set to: tag: relay-node

Note: the home tag access is to allow my mac access to my Windows PC.

---

## Step 3: Install & Configure DHCP + NAT  
Run on: Raspberry Pi

Install dhcpcd:
```bash
which dhcpcd  # If missing:
sudo apt install dhcpcd5 -y
sudo systemctl enable dhcpcd
sudo systemctl start dhcpcd
```
Configure Static IP on `eth1`:
```bash
sudo nano /etc/dhcpcd.conf
```
Append:
```bash
interface eth1
static ip_address=192.168.50.1/24
nohook wpa_supplicant
```
Restart Networking:
```bash
sudo systemctl restart dhcpcd
sudo systemctl status dhcpcd 
ip a  # Verify eth1 shows 192.168.50.1/24
```
If you dont see the proper static IP at eth1, run this on Pi:
```bash
sudo ip addr flush dev eth1
sudo ip addr add 192.168.50.1/24 dev eth1
```
Then confirm:
```bash
ip a
```
You should now see:
eth1: inet 192.168.50.1/24

---

## Step 4: Enable NAT Routing and IP Forwarding  
Run on: Raspberry Pi
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```
```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

## Step 5: Install DHCP Server for PC  

```bash
sudo apt install dnsmasq -y
sudo nano /etc/dnsmasq.conf
```
Append:
```conf
interface=eth1
dhcp-range=192.168.50.10,192.168.50.100,12h
```
Restart:
```bash
sudo systemctl restart dnsmasq
```
Now your Windows PC is connected to wifi (connected to Pi via Ethernet)

---

## Step 6: Configure Windows PC  

Make all the following changes to your PC below (Windows 11 Pro)

### Make the Firewall Rule Persistent and Force it ON at Boot
You can create a Startup Task that automatically runs Enable-NetFirewallRule each time your PC boots up.
1. Open Task Scheduler
2. Click Create Task (not "Basic Task")
3. Under General tab:
Name: Enable RDP Firewall
Run whether user is logged on or not
Run with highest privileges
4. Under Triggers:
New → At Startup
5. Under Actions:
Start a program →
Program/script: 
```bash
powershell.exe 
```
```bash
-Command "Enable-NetFirewallRule -DisplayGroup 'Remote Desktop'"
```
Save and Exit

Security Impact:
This is safe because you’re only re-enabling the Windows built-in RDP rules — you're not creating new loose ports.
However, always ensure you're connecting through Tailscale IP only, not allowing public internet access.

### Set Up Wake-on-LAN on PC
Enable Wake-on-LAN in BIOS:
- Reboot your Windows PC
- Enter BIOS Setup
Restart PC and hold F2 (need wired keyboard)
- Find Power Management section
- Enable:
Wake on LAN
Wake on PCI-E/PCI device
Save and exit.

### Configure Windows to Allow Wake
In Windows:
- Go to Device Manager > Network Adapters
- Find your Ethernet or WiFi adapter
- Right-click → Properties → Power Management tab
- Enable:
Allow this device to wake the computer
Only allow a magic packet to wake the computer

### Disable Windows "Fast Startup" (Critical)
Fast Startup breaks Wake-on-LAN on a lot of machines by default.
Steps:
- Go to Control Panel → Hardware and Sound → Power Options → Choose what the power buttons do 
- Click "Change settings that are currently unavailable"
- Under Shutdown settings, UNCHECK: Turn on fast startup (recommended)

- Next do these in the Advanced Tab:

Wake on Magic Packet: Enabled

Wake on Pattern Match: Enabled

Shutdown Wake-On-LAN: Enabled

Low Power Idle: Disabled (if present)

---

## Step 7: Finish WoL Set-up on Pi & Test Wake + RDP from Mac  
ssh into your Pi from your mac and run these commands:

1. Install wakeonlan on the Pi
Run this:
```bash
sudo apt update
sudo apt install wakeonlan -y
```
2. Get the PC’s MAC Address
On your Windows PC (before it sleeps), open PowerShell and run:
```powershell
getmac /v
```
Find the MAC address of the Ethernet adapter connected to the Pi.
It will look like: C8-A3-62-C4-81-2A
Convert it to Linux format (colons instead of dashes):
```bash
C8:A3:62:C4:81:2A
```
3. On the Pi: Send the Wake Packet
From your Pi:
```bash
wakeonlan C8:A3:62:C4:81:2A
```
(Replace with your actual MAC address.)

If everything is configured right on the PC:
It will wake up immediately
Within seconds, Tailscale will reconnect
You can RDP from your Mac again

---

## OPTIONAL: Lock Down PC Traffic Using Firewall (iptables)

Run on: Raspberry Pi

Apply the Pi Firewall Rules (to only allow RDP, web, and DNS):
```bash
# Allow RDP from the PC to the internet
sudo iptables -A FORWARD -i eth1 -p tcp --dport 3389 -j ACCEPT

# Allow DNS (UDP and TCP, for compatibility)
sudo iptables -A FORWARD -i eth1 -p udp --dport 53 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -p tcp --dport 53 -j ACCEPT

# Allow HTTP and HTTPS (web browsing + Windows/app updates)
sudo iptables -A FORWARD -i eth1 -p tcp --dport 80 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -p tcp --dport 443 -j ACCEPT

# Allow responses (stateful traffic handling)
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT

# Drop everything else from the PC
sudo iptables -A FORWARD -i eth1 -j DROP

# Save the rules to survive reboot
sudo netfilter-persistent save
```
Or block known telemetry IPs/ranges from PC.

---

## Final Network Topology

```bash
                      +------------------------------+
                      |         Your MacBook         |
                      | (Tailscale, RDP into PC & Pi)|
                      +-------------+----------------+
                                    |
                             Tailscale Encrypted Mesh
                                    |
         +--------------------------+-------------------------+
         |                                                    |
   +-----v-----+ eth0                                  +------v--------+
   |           |<---------LAN----------+               |                |
   |  Pi NAT/  |                       |               |  GL.iNet Travel|
   | Firewall + Tailscale              |               |  Router (WISP) |
   +-----+-----+                       |               |  (Captive Wi-Fi|
         | eth1                        +---------------+  Auth'd router)|
         v                                             +----------------+
+--------|-------------------------------------------------------------+
|        |       Your Windows PC (Ethernet to Pi eth1)                 |
|        +--> IP: 192.***.**.10                                        |
|        +--> Gateway: 192.***.**.1 (Raspberry Pi)                     |
|        +--> Tailscale Enabled ✅                                     |
|        +--> RDP & Wake-on-LAN Enabled ✅                             |
+----------------------------------------------------------------------+
```

---

## 👨‍💻 Author  
Mario Tagaras | Florida State University Alum  

