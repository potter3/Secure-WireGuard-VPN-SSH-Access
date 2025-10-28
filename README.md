


<img width="1024" height="1024" alt="jj" src="https://github.com/user-attachments/assets/1231a9f1-0422-47f6-bb3a-0bb595d90f3b" />




# WireGuard VPN + SSH Access Gate (Ubuntu Server, Debian, Parrot, os Kali + Windows Client) 
A complete, secure setup for creating your own VPN server using WireGuard on Ubuntu or any Debian-based systems, designed so that:

- Only users connected to your VPN can SSH into the server.

- SSH supports password-based login (for simplicity).

- You can connect easily from Windows, Linux, or mobile clients.

- All keys are stored securely under */etc/wireguard*

This guide is meant to be both practical (works out of the box) and educational (explains why each step matters).

---

## 🔒Concept Overview

Goal:
Make your Ubuntu server accessible only through your VPN.
That means:

- Public SSH (port 22) → 🔒 Blocked.

- VPN access (port 51820 UDP) → ✅ Open.

- Once connected to the VPN, you can SSH via the private IP (10.8.0.1).

Why this setup?

- Adds an invisible security layer (your SSH port is never exposed).

- Encrypts all traffic between your client and server.

- Prevents brute-force attacks, port scans, and credential leaks.


## 🧱 Server Setup (Parrot OS, Kali, or any Debian-based systems)

### 1️⃣ Install dependencies
``` bash
sudo apt update
sudo apt install wireguard -y
```


This installs:

- Kernel module → runs the VPN in the kernel (fast, efficient).

- Command tools → wg and wg-quick to configure and control the VPN.

### 2️⃣ STEP 2 — GENERATE SERVER KEYS🗝️
Each peer (server or client) needs a private + public key pair.
Think of it like SSH keys, one is secret, the other is shared.
```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

This generates:

-server_private.key → your server’s identity (keep private)

-server_public.key → shared with clients


### 3️⃣ Secure the Keys
Best practice is to move keys into */etc/wireguard* and restrict access. 
- If you were working from the **/etc/wireguard* directory from the start, skip the first command
  
```bash
sudo mv ~/server_private.key ~/server_public.key /etc/wireguard/
sudo chmod 600 /etc/wireguard/server_private.key
sudo chmod 644 /etc/wireguard/server_public.key
sudo chown root:root /etc/wireguard/server_private.key /etc/wireguard/server_public.key
sudo chmod 700 /etc/wireguard
```
Why:

- /etc/wireguard is root-only by default.

- Restrictive permissions prevent accidental exposure.

- The private key never leaves this directory


### 4️⃣ Configure the WireGuard Server

Check your Network interface name with :
```bash
ip a
```


If your network adapter isn’t called *<eth0>*, change it to the one shown with the command above, "it is usually the second interface." 

Open/create the configuration file:
```bash
sudo nano /etc/wireguard/wg0.conf
```

Paste this (replace placeholders):
```ini
[Interface]
# Server (VPN) interface configuration
Address = 10.8.0.1/24        # VPN network address range
ListenPort = 51820           # UDP port WireGuard listens on
PrivateKey = <server_private_key_here>

# NAT and forwarding rules (allow VPN traffic to go out)
PostUp   = iptables -A FORWARD -i %i -j ACCEPT
PostUp   = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE #If your network adapter isn’t called eth0, check with: ip a
PostDown = iptables -D FORWARD -i %i -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE #If your network adapter isn’t called eth0, check with: ip a
```

Explanation:

- Address creates a private subnet 10.8.0.0/24

- ListenPort opens UDP 51820

- PrivateKey identifies your server

- PostUp and PostDown add/remove firewall rules automatically when VPN starts/stops



### 5️⃣ ENABLE PACKET FORWARDING
This allows the server to route traffic between VPN clients and the system:
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 6️⃣ ENABLE AND START WIREGUARD
Activate the VPN:
```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Check it’s running:
```bash
sudo wg show
ip addr show wg0
```

You should see:

- Interface: wg0

- Address: 10.8.0.1

- Listening port: 51820

### 7️⃣ CREATE WINDOWS CLIENT KEYS
Each client also needs its own key pair:
```bash
wg genkey | tee win_private.key | wg pubkey > win_public.key
```

Now you have:

- win_private.key → for the Windows client config

- win_public.key → add to server config

### 8️⃣ ADD CLIENT TO SERVER CONFIG

Append to the bottom of */etc/wireguard/wg0.conf*:
```ini
[Peer]
# Windows Client
PublicKey = <win_public_key_here>
AllowedIPs = 10.8.0.2/32
```

Explanation:

- PublicKey identifies this client

- AllowedIPs assigns the client’s VPN IP (10.8.0.2)

Restart WireGuard to apply changes:
```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```


Check:
```bash
sudo wg show
```
You’ll see your Windows peer listed. 

### 9️⃣ CREATE WINDOWS CLIENT CONFIG FILE

Use this command to know your *Router's Public IP*
```bash
sudo apt update
sudo apt install curl -y
curl ifconfig.me
```

On Ubuntu, make a file client.conf with:
```ini
[Interface]
PrivateKey = <win_private_key_here>
# Your client’s VPN IP (unique per client)
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
# Server’s public key (from /etc/wireguard/server_public.key on Ubuntu)
PublicKey = <server_public_key_here>
Endpoint = <your_server_public_ip>:51820 #use this if you want SSH only to run locally "only when you are on your network
# Use ONE of these for this if you want to SSH from any network, remove the # for the endpoint you need
# 1) Router Public IP (from curl)
#Endpoint = 203.0.113.45:51820
# 2) Dynamic DNS (DuckDNS or similar)
# Send only the VPN subnet through the tunnel
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

Explanation:

- Address assigns the client’s VPN IP (10.8.0.2)

- DNS is optional, gives you name resolution

- Endpoint = your server’s public IP or domain + port 51820

- AllowedIPs = routes that go through the VPN
(Here, only the private VPN subnet 10.8.0.0/24)

- PersistentKeepalive keeps NAT holes open on the client side

### 🔟 TRANSFER CLIENT CONFIG TO WINDOWS 
Safely copy client.conf to your Windows machine:

- Via USB

- Or scp while SSH is  still open


### 11 CONNECT WINDOWS TO VPN 🌐

1. Download WireGuard for Windows → https://www.wireguard.com/install/

2. Open it → click Add Tunnel → Import from File

3. Choose client.conf

4. Click Activate

You’ll see:
```java
Handshake for peer (server_public_key) established
```
Now your Windows PC has a virtual IP *10.8.0.2*.

Test:
```powershell
ping 10.8.0.1
```
You should get replies from your Ubuntu server.

### 12 CONFIGURE FIREWALL (UFW)
We’ll restrict SSH to VPN users only.

```bash 
sudo ufw allow 51820/udp
sudo ufw allow from 10.8.0.0/24 to any port 22 proto tcp
sudo ufw deny 22/tcp
sudo ufw enable
sudo ufw status verbose
```

Explanation:

- Port 51820/UDP open to everyone (VPN entry)

- Port 22/TCP (SSH) denied publicly

- Port 22 is allowed only for the VPN subnet

Now SSH is accessible only through the VPN.

### 13 CONFIGURE SSH FOR PASSWORD LOGIN 🗝️
Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Make sure these lines exist and are not commented out:

```nginx
PasswordAuthentication yes
PermitRootLogin no
```

Restart SSH:
```bash
sudo systemctl restart ssh
```

✅ Now SSH accepts password logins but only over the VPN.



---




🏁 PERFORMANCE NOTES

- WireGuard adds very low latency (~10–20 ms).

- Perfect for VNC, RDP, SSH, and file transfers.

- Much faster and lighter than OpenVPN.

- Uses strong, modern cryptography (ChaCha20, Curve25519).



---
