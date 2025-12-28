---
title: Linux server hardening checklist
layout: home
parent: Hardening
nav_order: 1
tags: [hardening, linux, checklist, ]
---
# Linux server hardening checklist
---

### 1. Keep packages update

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Configure Firewall (ufw/iptables)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow ssh
sudo ufw enable
```

### 3. Disable ssh root login

```bash
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 4. Use ssh key authentication and disable password

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
ssh-copy-id user@server_ip
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?ChallengeResponseAuthentication.*/ChallengeResponseAuthentication  no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

### 5. Install and configure Fail2ban

```bash
sudo apt install fail2ban -y
sudo tee /etc/fail2ban/jail.local > /dev/null << 'EOF'
	[sshd]
	enabled = true
	port = ssh
	filter = sshd
	logpath = /var/log/auth.log
	maxretry = 5
	bandtime = 1h
	findtime = 10m
	EOF
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

#### Note
> For systemd-based distributions use

```bash
[sshd]
enabled = true
port = ssh
filter = sshd
maxretry = 5
bandtime = 1h
findtime = 10m
backend = systemd
```

Check funtional `fail2ban`

```bash
# Verify status
sudo systemctl status fail2ban
# Check active jails
sudo fail2ban-client status
# Check for blocked IPs
sudo fail2ban-client status sshd
# Check logs fail2ban
sudo tail -f /var/log/fail2ban.log
```

### 6. Remove unnecesary services

```bash
systemctl list-units --type=service --state=running
sudo systemctl disable --now service_name
```

### 7. Set proper permissions and least privilegie

```bash
sudo find / -type f -perm -o+w 2>/dev/null | head
sudo find / -type d -perm -o+w 2>/dev/null | head
```

### 8. Audit processes and open ports

```bash
ss -tulpn
ps aux --sort=-%cpu | head
```

### 9. Check logs

```bash
# In systemd
sudo journalctl -u ssh --since "today"
#No systemd
sudo tail -f /var/log/auto.log
```

### 10. Regular backups and tested restores

```bash
sudo tar -czf /directory/backups/etc-$(date +%F).tar.gz /etc
ls -lh /directory/backups/
sudo crontab -e 
```

##### Credits:

> `Cristopher Mejia`
