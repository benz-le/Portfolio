# Pi-Hole with PiVPN (WireGuard) and Unbound on Raspberry Pi
These are my installation notes for creating a DNS server using Pi-Hole, PiVPN (WireGuard), and Unbound on a Raspberry Pi Zero 2 W as a homelab project. PiVPN and WireGuard create a VPN connection on port `51820` to access the Pi-Hole admin interface, using a public DNS hostname I set up through `noip.com` and a DDNS key for my home router (to enable dynamic DNS). This setup allows connection to Pi-Hole from any device, anywhere.

Split-tunneling is used to route DNS traffic to and from Pi-Hole, enabling ad blocking and DNS-based denylisting. Unbound is configured as a recursive DNS server to perform root DNS resolution and cache the results, eliminating reliance on third-party DNS services like Google or Cloudflare.

# Set up Raspberry Pi
Materials required:
- Raspberry Pi – I used a Raspberry Pi Zero 2 W for its compact form factor, though it lacks a wired Ethernet port.
- Micro USB to RJ45 Ethernet Adapter (optional) – A wired connection is more reliable than Wi-Fi, especially on a Pi Zero.
- microSD Card – Storage size is flexible; I used a 128GB card (more than necessary).
- [Raspberry Pi Imager](https://www.raspberrypi.com/software/) - Used to format the SD card and install the OS.
## Step 1: Install Raspberry Pi OS
- [Download](https://www.raspberrypi.com/software/) and run Raspberry Pi Imager.
- Insert the microSD card into a card reader.
- Choose an operating system like `Raspberry Pi OS Lite (64-bit)`.
- Select the microSD card (note the drive letter).
- Click the settings icon and:
  - Set the hostname to your preferred name.
  - Enable SSH and leave the option as `Use password authentication`.
  - Set the username and password for future logins.
  - Set locale settings for your time zone and keyboard layout.
- Click “Write” and insert the microSD card into your Raspberry Pi.
- Power on the Raspberry Pi to boot it up.
![Raspberry Pi Imager](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/imager.png)
## Step 2: Use Terminal or PuTTY to SSH into the Pi 
- Open Command Prompt on Windows and run:
```
ssh pi@192.168.1.195
```
- Or use PuTTY:
  - Enter the IP address in the "Host Name" field.
  - Click "Open".
![PuTTY SSH](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/putty.png)
## Step 3: Update Raspberry Pi OS and Packages
- Run the following command to update all installed packages:
```
sudo apt update && sudo apt upgrade -y
```
# Set a Static IP Address
## Step 4a: Configure DHCP Reservation
- Access your router's DHCP lease table or network map (e.g., on a Linksys router).
- Look for the Pi’s hostname (e.g., `pihole`) to see its assigned IP address.
- Reserve this IP address for the Raspberry Pi’s MAC address (e.g., `192.168.1.195` for `pihole`).
![DHCP Reservation on LinkSys Router](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/dhcp_reservation.png)
## Step 4b: Set a Static IP Directly on the Pi
- If you can’t determine the Pi’s IP via the router:
  - Connect a monitor and keyboard directly to the Pi.
  - Run:
```
ip a
```
- Manually configure a static IP on the Pi itself. (I used both DHCP reservation and manual config to ensure consistency.) Open the Network Manager text user interface:
```
sudo nmtui
```
- Select Edit a Connection, then choose Wired Connection 1.
- Set IPv4 Configuration to Manual and enter:
  - Address: `192.168.1.195`
  - Gateway: `192.168.1.1`
  - DNS Servers: `192.168.1.1`, `1.1.1.1`
- Save the configuration and reboot the Pi:
```
sudo reboot
```
![Setting Static IP on Pi using nmtui](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/staticip.png)
# Install PiHole
## Step 5: Run the Installation Script
```
curl -sSL https://install.pi-hole.net | bash
```
## Step 6: Configure Using the Installation Wizard
- Review each screen carefully.
- On the “Static IP Address” screen, verify that your IP matches the one you configured earlier. Press ENTER (select Yes - Set static IP using current values).
- For “Select Upstream DNS Provider,” choose any option temporarily; you'll replace this with Unbound later.
- Continue through the remaining screens by selecting the recommended defaults ENTER (Yes).
- When prompted to “Select a privacy mode for FTL,” choose Show Everything and press ENTER (Continue).
- You will see a final screen confirming that installation is complete.
## Step 7: Access the Pi-Hole Admin Interface
- Set a new password for the Pi-Hole admin interface:
```
pihole setpassword `yourpassword`
```
- Open a browser and navigate to:
```
HTTP://192.168.1.195/admin
```
- Log in using the password you just created.
![Pi-Hole Admin Interface](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/pihole_interface.png)
# Install Unbound
## Step 8: Install Unbound
```
sudo apt install unbound -y
```
## Step 9: Configure Unbound
- Create a new configuration file for Unbound:
```
sudo nano -w /etc/unbound/unbound.conf.d/pi-hole.conf
```
<details>
<summary>Click to expand: Paste the following configuration into the file</summary>

```
server:
    # If no logfile is specified, syslog is used
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes

    # May be set to no if you don't have IPv6 connectivity
    do-ip6: yes

    # You want to leave this to no unless you have *native* IPv6. With 6to4 and
    # Terredo tunnels your web browser should favor IPv4 for the same reasons
    prefer-ip6: no

    # Use this only when you downloaded the list of primary root servers!
    # If you use the default dns-root-data package, unbound will find it automatically
    #root-hints: "/var/lib/unbound/root.hints"

    # Trust glue only if it is within the server's authority
    harden-glue: yes

    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size.
    # IP fragmentation is unreliable on the Internet today, and can cause
    # transmission failures when large DNS messages are sent via UDP. Even
    # when fragmentation does work, it may not be secure; it is theoretically
    # possible to spoof parts of a fragmented DNS message, without easy
    # detection at the receiving end. Recently, there was an excellent study
    # >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
    # by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
    # in collaboration with NLnet Labs explored DNS using real world data from the
    # the RIPE Atlas probes and the researchers suggested different values for
    # IPv4 and IPv6 and in different scenarios. They advise that servers should
    # be configured to limit DNS messages sent over UDP to a size that will not
    # trigger fragmentation on typical network links. DNS servers can switch
    # from UDP to TCP when a DNS response is too big to fit in this limited
    # buffer size. This value has also been suggested in DNS Flag Day 2020.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes

    # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
    num-threads: 1

    # Ensure kernel buffer is large enough to not lose messages in traffic spikes
    so-rcvbuf: 1m

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10

    # Ensure no reverse queries to non-public IP ranges (RFC6303 4.2)
    private-address: 192.0.2.0/24
    private-address: 198.51.100.0/24
    private-address: 203.0.113.0/24
    private-address: 255.255.255.255/32
    private-address: 2001:db8::/32
```
</details>

## Step 10: Start and Test Unbound
- Restart Unbound and test that it’s functioning:
```
sudo service unbound restart
dig pi-hole.net @127.0.0.1 -p 5335
```
- Log into the Pi-hole web interface and configure the upstream DNS server:
  - Go to Settings > DNS.
  - Uncheck all preset upstream DNS providers.
  - In the “Custom 1 (IPv4)” field, enter:
```
127.0.0.1#5335
```
![Set Unbound DNS Server as Upstream](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/unbound.png)
# Install PiVPN with WireGuard
## Step 11: Run the PiVPN Installation Script
```
curl -L https://install.pivpn.io | bash
```
## Step 12: Configure via Installation Wizard
- Proceed carefully through the wizard:
- On the “DHCP Reservation” screen, press → then ENTER (No: Setup static IP address).
![DHCP Reservation Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/dhcpsetting.png)
- On the “Static IP Address” screen, confirm the settings are correct, then press ENTER (Yes).
- On the “Choose a user” screen, select the default user (typically pi).
![Choose a user Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/user.png)
- On the “Installation mode” screen, choose “WireGuard” as our VPN, and press ENTER (WireGuard).
![Installation Mode Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/vpnoptions.png)
- The default WireGuard port should be `51820`.
![Port Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/port.png)
- If Pi-hole is detected, that’s expected and correct.
![Pi-Hole Detected Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/piholedetected.png)
- On the “Public IP or DNS” screen:
  - If you plan to use Dynamic DNS, select that option by pressing ↓, then SPACE, and ENTER (DNS Entry - Use a public DNS).
    - Enter your public DNS domain. I will include instructions for creating a DNS hostname and DDNS key for your Dynamic DNS/home network using `noip.com` [here](https://github.com/Nova281/pihole-pivpn-unbound-raspberrypi-server?tab=readme-ov-file#set-up-a-domain-name-for-router-optional).
  - Otherwise, choose to use your public IP address.
![Public IP or DNS Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/publicdns.png)
![DDNS Hostname Screen](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/enterddns.png)
- For “Unattended Upgrades,” press ENTER (Yes).
## Step 13: Set Port Forwarding for PiVPN
- Log in to your router and access the Port Forwarding settings.
- Add a new rule with:
  - Application Name: `vpn` 
  - External Port: `51820`
  - Internal Port: `51820`
  - Protocol: `UDP`
  - Internal IP: Your Pi’s static IP (e.g., `192.168.1.195`)
![Port Forwarding Setting on Router](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/portforwarding.png)
## Step 14: Add a VPN Client
- Run the following command to create a new client profile:
```
pivpn -a
```
- Name the client (e.g., `{username-device}` like `ble-pc`).
![Add Client to PiVPN](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/addclient.png)
- Transfer the config file to your client machine using Secure Copy (SCP). On Windows, you can use `pscp`:
```
pscp -P 22 pi@192.168.1.195:/home/pi/configs/{username-device}.conf {username-device}.conf
```
- Replace placeholders accordingly and enter your Pi password when prompted.
![Secure Copy to Windows](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/pscp.png)
![Conf file on Windows](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/conffile.png)
- To add a mobile device, you can use a QR code instead:
```
pivpn -qr
```
![QR Code Example](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/qrcode.png)
## Step 15: Set Up WireGuard Tunnel
- On Windows: Open WireGuard, click “Add Tunnel,” and select the `.conf` file you transferred.
- On Mobile: Tap “+” and choose “Create from QR code,” then scan the generated QR code.
- Once the tunnel is activated, your device will route DNS traffic securely through Pi-hole, even outside your home network.
![WireGuard Client on Windows](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/wireguard.png)
![Pi-Hole Detects VPN Client outside of Network](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/wireguardconnection.png)
# Set Up a Domain Name for Router (Optional)
- Register with a Dynamic DNS provider like `noip.com`.
- Create a free hostname. Note: Free hostnames must be confirmed every 30 days to avoid deletion.
![Hostname Homepage](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/ddnshostname.png)
- Generate a DDNS key (includes username, password, and domain).
![DDNS Screen on noip.com](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/ddns.png)
- Log in to your router and navigate to DDNS settings:
  - Select provider: No-IP
  - Enter the credentials provided:
```
DDNS Provider: No-IP
Username: Your DDNS Key Username / Email
Password: Your DDNS Key Password
Hostname/Domain: all.ddnskey.com
Server/Server Address: dynupdate.no-ip.com
```
  - Not every device will ask you for a server or server address. The service still works without issue if the device does not request it.
- Click “Update” and verify that the status shows as “Success.”
- Test the new domain in a browser; your router’s login page should appear.
![Router DDNS Setting](https://github.com/benz-le/Portfolio/blob/main/Pi-Hole%2C%20PiVPN%20(WireGuard)%2C%20Unbound%20on%20Raspberry%20Pi/images/routerddns.png)
# Notes
- The Pi-hole dashboard is accessible from both LAN and VPN.
- Be mindful of periodic No-IP hostname verification if using their free service.
- For added security, consider implementing firewall rules, fail2ban, and Ansible (configuration management).
