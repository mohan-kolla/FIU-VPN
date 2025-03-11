# Personal VPN Creation

## Create a Skyhole (PiHole + Cloud + Unbound) VPN (WireGuard/OpenVPN) in the AWS Cloud using Lightsail.

### Prerequisites
To get started, you will need:
- **An AWS Account**
- **Time** (setup and troubleshooting)
- **A small budget** (AWS Lightsail costs start at $5/month)
- **Basic understanding of Linux commands**

---

## Step 1: Create an AWS Lightsail Instance

1. **Log into AWS** and navigate to **Lightsail** using the search bar.
2. Click on **Create Instance**.
3. Choose your **Region** and **Availability Zone**. (Example: `us-east-1/Zone A`)
4. Under **Pick your instance image**:
   - Select **OS Only** → Choose **Debian** or **Ubuntu**.
5. **SSH Key Pair**:
   - Use an existing key or generate a new one.
   - Store it securely (e.g., private S3 bucket or a USB drive).
6. **Enable automatic snapshots** *(Optional: Recommended after setup completion).*
7. **Choose an instance plan**:
   - `$5/month` for personal use (3-4 devices).
   - `$7/month` for more RAM/storage.
   - Larger plans for 20+ concurrent devices.
8. **Name your instance**, assign tags *(optional)*, and click **Create Instance**.

---

## Step 2: Configure Networking

1. Navigate to the **Networking** tab in Lightsail.
2. Click **Create Static IP** and attach it to your instance.
   - *(Note: AWS allows up to 5 free static IPs while attached to an instance.)*
3. Disable **IPv6 Networking** (use IPv4 only).
4. Under **Firewall settings**:
   - Remove ports `80` and `443`.
   - Keep port `22` (SSH).
   - Add a port for your VPN (depends on protocol: WireGuard/OpenVPN).

---

## Step 3: SSH into Your Instance

Once the instance is ready:

1. Click **Connect** in AWS Lightsail to open an SSH session.
2. Run the following commands to update and reboot:
   ```bash
   sudo apt-get update
   sudo apt-get upgrade -y
   sudo reboot
   ```
3. Reconnect after the reboot.

---

## Step 4: Install Pi-hole

Run the following command to install Pi-hole:
```bash
sudo curl -sSL https://install.pi-hole.net | bash
```

During installation:
- Select an **Upstream DNS Provider** (e.g., Cloudflare).
- Ensure **IPv4** is selected.
- Note down the **Web Interface URL** and **Admin password** *(you can change it later)*.

---

## Step 5: Install PiVPN (OpenVPN/WireGuard)

1. Run the installation script:
   ```bash
   curl -L https://install.pivpn.io | bash
   ```
2. Follow the prompts:
   - Choose **WireGuard** or **OpenVPN**.
   - Select your local user.
   - Use your previously assigned **Static IP**.
3. Save the generated configuration files securely.

---

## Step 6: Configure Pi-hole as a Recursive DNS Server (Unbound)

1. Install Unbound:
   ```bash
   sudo apt install unbound -y
   ```
2. Edit the Unbound configuration file:
   ```bash
   sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
   ```
   Add the following configuration:
   ```
   server:
       interface: 127.0.0.1
       port: 5335
       do-ip4: yes
       do-udp: yes
       do-tcp: yes
       prefetch: yes
       num-threads: 1
       private-address: 192.168.0.0/16
       private-address: 10.0.0.0/8
   ```
3. Restart Unbound:
   ```bash
   sudo service unbound restart
   ```
4. Test Unbound:
   ```bash
   dig pi-hole.net @127.0.0.1 -p 5335
   ```
   *(The first query may be slow, but subsequent ones will be faster.)*

---

## Step 7: Configure Pi-hole to Use Unbound

1. Go to the **Pi-hole Web Interface**.
2. Navigate to **Settings** → **DNS**.
3. Set **Custom DNS (IPv4)** to `127.0.0.1#5335`.
4. Save changes.

---

## Step 8: Disable Unwanted Services (For Debian Bullseye+ Users)

1. Check if `unbound-resolvconf.service` is active:
   ```bash
   systemctl is-active unbound-resolvconf.service
   ```
2. If active, disable it:
   ```bash
   sudo systemctl disable --now unbound-resolvconf.service
   ```
3. Modify `resolvconf.conf` to prevent conflicts:
   ```bash
   sudo sed -Ei 's/^unbound_conf=/#unbound_conf=/' /etc/resolvconf.conf
   sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
   ```
4. Restart Unbound:
   ```bash
   sudo service unbound restart
   ```

---

## Step 9: Automate Updates and Maintenance

1. Create a **scripts** folder:
   ```bash
   mkdir ~/scripts
   touch ~/scripts/update.sh
   nano ~/scripts/update.sh
   ```
2. Add the following update script:
   ```bash
   #!/bin/bash
   sudo apt-get update
   sudo apt-get upgrade -y
   sudo pihole updatePihole
   sudo pihole updateGravity
   sudo apt autoclean -y
   sudo apt autoremove -y
   ```
3. Grant execution permission:
   ```bash
   chmod +x ~/scripts/update.sh
   ```
4. Automate updates with a **cron job**:
   ```bash
   crontab -e
   ```
   Add the following lines:
   ```
   0 6 * * * ~/scripts/update.sh
   30 6 * * * /sbin/shutdown -r
   ```
   *(This will run updates daily at 6 AM and reboot at 6:30 AM.)*

---

## 🎉 Congratulations!
Your Skyhole VPN is now set up with Pi-hole and Unbound, providing a secure, private, and ad-free (mostly, still recommend using another adblocker for even better security) browsing experience! 🚀

