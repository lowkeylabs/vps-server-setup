# Securing a new VPS server
Securing your new Ubuntu 22.04.5 VPS is crucial to protect your server from unauthorized access, vulnerabilities, and potential attacks. Below is a comprehensive step-by-step guide to help you secure your server and prepare it for deploying applications:

---

### 1. **Initial System Update**

Before making any changes, ensure your system is up-to-date.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
```

---

### 2. **Create a New User with Sudo Privileges**

Using the root account for daily operations is risky. Create a new user and grant it sudo access.

```bash
sudo adduser john
sudo usermod -aG sudo john
```

---

### 3. **Configure SSH for Enhanced Security**

#### a. **Set Up SSH Key-Based Authentication**

1. **Generate SSH Keys on Your Local Machine:**

   ```bash
   ssh-keygen -t ed25519 -C "jdleonard@vcu.edu"
   ```

   *Follow the prompts to save the key, optionally with a passphrase.*

2. **Create .ssh folder and authorized_keys file:**

   ```bash
   mkdir -p ~/.ssh
   touch ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   chmod 600 ~/.ssh/authorized_keys
   ```

3. **Append local machine public key to authorized keys file:**

   Clip the contents of the local machine's id_25519.pub file into your clipboard:

   ```bash
   type "~\.ssh\id_ed25519.pub" | clip
   ```

   then,

   ```bash
   echo <paste> >> ~/.ssh/authorized_keys
   ```

#### b. **Disable Password Authentication**

1. **Edit SSH Configuration:**

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. **Modify/Add the Following Lines:**

   ```bash
   PermitRootLogin no
   PasswordAuthentication yes
   ```

   Change PasswordAuthentication to no if/when you're done adding new users.  Otherwise, you'll need to add the ~/.ssh/authorized_keys file to each user manually.

3. **Restart SSH Service:**

   ```bash
   sudo systemctl restart ssh
   ```

#### c. **Optional: Change the Default SSH Port**

Changing the default SSH port (22) can reduce automated attack attempts.

1. **Edit SSH Configuration:**

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. **Change the Port Number:**

   ```bash
   Port 22
   Port 7822
   ```

   *Choose a port number above 1024.*

3. **Update UFW Rules (see Firewall Setup below) and Restart SSH:**

   ```bash
   sudo systemctl restart ssh
   ```

---

### 4. **Set Up a Firewall (UFW)**

Ubuntu's Uncomplicated Firewall (UFW) is a user-friendly interface to manage iptables firewall rules.

1. **Install ufw**

   ```bash
   sudo apt install ufw -y
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   ```

2. **Allow SSH Connections:**

   ```bash
   sudo ufw allow OpenSSH
   sudo ufw allow 7822/tcp
   ```

3. **Enable UFW:**

   ```bash
   sudo ufw enable
   sudo ufw status
   ```

4. **Allow Other Necessary Ports:**

   Depending on your applications, allow necessary ports (e.g., HTTP, HTTPS).

   ```bash
   sudo ufw allow 'Apache Full'
   sudo ufw allow 'MySQL'
   ```

---

### 5. **Install and Configure Fail2Ban**

Fail2Ban helps protect against brute-force attacks by banning suspicious IPs.

1. **Install Fail2Ban:**

   ```bash
   sudo apt install fail2ban -y
   ```

2. **Create a Local Configuration File:**

   ```bash
   sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
   ```

3. **Edit `jail.local` to Customize Settings:**

   ```bash
   sudo nano /etc/fail2ban/jail.local
   ```

   - Ensure `[sshd]` section is enabled.
   - Adjust `bantime`, `findtime`, and `maxretry` as needed.

4. **Restart Fail2Ban:**

   ```bash
   sudo systemctl restart fail2ban
   sudo systemctl enable fail2ban
   ```

5. **Check Fail2Ban Status:**

   ```bash
   sudo fail2ban-client status
   sudo fail2ban-client status sshd
   ```

---

### 6. **Disable Unnecessary Services and Remove Unused Software**

Minimize potential attack vectors by disabling services you don’t need.

1. **List Active Services:**

   ```bash
   sudo systemctl list-unit-files --type=service | grep enabled
   ```

2. **Disable Unneeded Services:**

   ```bash
   sudo systemctl disable service_name
   sudo systemctl stop service_name
   ```

   *Replace `service_name` with the actual service name.*

3. **Remove Unnecessary Packages:**

   ```bash
   sudo apt purge package_name -y
   sudo apt autoremove -y
   ```

---

### 7. **Enable Automatic Security Updates**

Ensure your server receives security patches promptly.

1. **Install Unattended Upgrades:**

   ```bash
   sudo apt install unattended-upgrades -y
   ```

2. **Enable Unattended Upgrades:**

   ```bash
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```

3. **Configure Unattended Upgrades (Optional):**

   ```bash
   sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
   ```

   *Ensure the following line is enabled:*

   ```bash
   "${distro_id}:${distro_codename}-security";
   ```

---

### 8. **Set Up SSH Rate Limiting with UFW (Optional)**

Further protect SSH by limiting connection rates.

```bash
sudo ufw limit OpenSSH
sudo ufw limit 7822/tcp
```

*Adjust if you changed the SSH port.*

---

### 9. **Configure Swap Space (If Needed)**

Depending on your server's RAM, configuring swap can prevent out-of-memory issues.

1. **Check Existing Swap:**

   ```bash
   sudo swapon --show
   ```

2. **Create a Swap File (e.g., 2G):**

   ```bash
   sudo fallocate -l 2G /swapfile
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile
   ```

3. **Make Swap Permanent:**

   ```bash
   echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
   ```

4. **Adjust Swappiness (Optional):**

   ```bash
   sudo sysctl vm.swappiness=10
   echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
   ```

---

### 10. **Secure Shared Memory**

Protect shared memory from being exploited.

1. **Edit `fstab`:**

   ```bash
   sudo nano /etc/fstab
   ```

2. **Add the Following Line:**

   ```bash
   tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0
   ```

3. **Remount `/run/shm`:**

   ```bash
   sudo mount -o remount /run/shm
   ```

---

### 11. **Configure AppArmor**

AppArmor provides Mandatory Access Control (MAC) to confine programs.

1. **Ensure AppArmor is Installed and Enabled:**

   ```bash
   sudo apt install apparmor apparmor-utils -y
   sudo systemctl enable apparmor
   sudo systemctl start apparmor
   ```

2. **Check AppArmor Status:**

   ```bash
   sudo aa-status
   ```

3. **Enforce Profiles for Installed Applications as Needed.**

---

### 12. **Harden Network Settings**

Improve network security by tweaking system parameters.

1. **Edit Sysctl Configuration:**

   ```bash
   sudo nano /etc/sysctl.conf
   ```

2. **Add the Following Lines:**

   ```bash
   net.ipv4.ip_forward = 0
   net.ipv4.conf.all.accept_source_route = 0
   net.ipv6.conf.all.accept_source_route = 0
   net.ipv4.conf.all.accept_redirects = 0
   net.ipv6.conf.all.accept_redirects = 0
   net.ipv4.conf.all.secure_redirects = 0
   net.ipv4.icmp_echo_ignore_broadcasts = 1
   net.ipv4.icmp_ignore_bogus_error_responses = 1
   net.ipv4.tcp_syncookies = 1
   net.ipv4.conf.all.rp_filter = 1
   ```

3. **Apply the Changes:**

   ```bash
   sudo sysctl -p
   ```

---

### 13. **Set Up Regular Backups**

Ensure you have regular backups to recover from data loss or breaches.

- **Use Tools Like `rsync`, `tar`, or Backup Services:**
  
  Example using `rsync` to backup `/var/www` to `/backup`:

  ```bash
  sudo rsync -av --delete /var/www /backup
  ```

- **Automate Backups with Cron Jobs:**

  ```bash
  crontab -e
  ```

  *Add a cron job, e.g., daily at 2 AM:*

  ```bash
  0 2 * * * rsync -av --delete /var/www /backup
  ```

---

### 14. **Implement Monitoring and Logging**

Stay informed about your server's health and potential security issues.

1. **Install Monitoring Tools:**

   - **`htop` for system monitoring:**

     ```bash
     sudo apt install htop -y
     ```

   - **`vnstat` for network monitoring:**

     ```bash
     sudo apt install vnstat -y
     ```

2. **Set Up Log Management:**

   - **Use `logwatch` for log analysis:**

     ```bash
     sudo apt install logwatch -y
     ```

     *Configure it to send daily summaries via email.*

3. **Consider Advanced Monitoring Solutions:**

   - **Tools like Prometheus, Grafana, or Cloud-Based Services (e.g., Datadog).**

---

### 15. **Set Up Time Synchronization**

Ensure your server's time is accurate, which is vital for security protocols and logging.

1. **Install `chrony`:**

   ```bash
   sudo apt install chrony -y
   ```

2. **Enable and Start Chrony:**

   ```bash
   sudo systemctl enable chrony
   sudo systemctl start chrony
   ```

3. **Verify Time Sync Status:**

   ```bash
   chronyc tracking
   ```

---

### 16. **Limit User Access and Privileges**

Ensure users have the minimum necessary permissions.

1. **Review Sudo Users:**

   ```bash
   getent group sudo
   ```

2. **Remove Unnecessary Users from Sudo Group:**

   ```bash
   sudo deluser username sudo
   ```

3. **Set Strong Password Policies (if using password auth):**

   - **Install `libpam-pwquality`:**

     ```bash
     sudo apt install libpam-pwquality -y
     ```

   - **Configure Password Requirements:**

     ```bash
     sudo nano /etc/pam.d/common-password
     ```

     *Adjust parameters like `minlen`, `dcredit`, `ucredit`, etc.*

---

### 17. **Implement Two-Factor Authentication (Optional but Recommended)**

Adding an extra layer of security for SSH access.

1. **Install Google Authenticator:**

   ```bash
   sudo apt install libpam-google-authenticator -y
   ```

2. **Configure Google Authenticator for Your User:**

   ```bash
   google-authenticator
   ```

   *Follow the prompts to set up.*

3. **Edit SSH PAM Configuration:**

   ```bash
   sudo nano /etc/pam.d/sshd
   ```

   *Add the following line at the end:*

   ```bash
   auth required pam_google_authenticator.so
   ```

4. **Edit SSH Configuration to Enable Challenge-Response:**

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

   *Ensure the following lines are set:*

   ```bash
   ChallengeResponseAuthentication yes
   ```

5. **Restart SSH Service:**

   ```bash
   sudo systemctl restart ssh
   ```

---

### 18. **Finalize and Review Security Settings**

1. **Reboot the Server to Ensure All Changes Take Effect:**

   ```bash
   sudo reboot
   ```

2. **Verify SSH Access with the New User:**

   ```bash
   ssh your_username@your_server_ip
   ```

3. **Ensure Root Login is Disabled:**

   ```bash
   ssh root@your_server_ip
   ```

   *It should be denied.*

4. **Check Firewall Rules:**

   ```bash
   sudo ufw status verbose
   ```

5. **Review Fail2Ban Bans and Logs:**

   ```bash
   sudo fail2ban-client status
   sudo tail /var/log/fail2ban.log
   ```

---

### 19. **Prepare for Application Deployment**

With the server secured, you can now install necessary software and frameworks based on your application requirements. Common preparatory steps include:

- **Install a Web Server (e.g., Nginx or Apache):**

  ```bash
  sudo apt install nginx -y
  ```

- **Install a Database Server (e.g., MySQL, PostgreSQL):**

  ```bash
  sudo apt install mysql-server -y
  ```

- **Install Programming Languages and Runtimes (e.g., Python, Node.js):**

  ```bash
  sudo apt install python3-pip -y
  curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
  sudo apt install -y nodejs
  ```

- **Set Up Environment Variables and Dependencies as Needed.**

---

### 20. **Regular Maintenance and Auditing**

Security is an ongoing process. Regularly perform the following:

- **Update the System:**

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

- **Monitor Logs for Suspicious Activity.**

- **Review User Accounts and Access Permissions.**

- **Conduct Security Audits and Vulnerability Scans (e.g., using `Lynis`).**

  ```bash
  sudo apt install lynis -y
  sudo lynis audit system
  ```

---

By following these steps, you will establish a robust security foundation for your Ubuntu VPS, minimizing vulnerabilities and preparing the server for deploying your applications securely. Always stay informed about security best practices and regularly update your knowledge to handle new threats effectively.