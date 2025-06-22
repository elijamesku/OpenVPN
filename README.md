## OpenVPN Access Server Deployment with MFA, Auto-login & Secure Configuration

This guide walks through a fully automated and secured OpenVPN Access Server setup using AWS EC2 (Ubuntu), including:

* OpenVPN installation
* Security group rules
* Public IP profile fix
* MFA (TOTP) setup
* Auto-login profile download
* User creation automation script
* (Optional) Elastic IP + DNS routing


## Prerequisites

* AWS account
* Git Bash
* EC2 Ubuntu 22.04+ instance
* Security group allowing:

  * TCP `22` from IP (SSH)
  * TCP `943` from IP (Admin portal)
  * TCP `443` from IP (TLS VPN)
  * UDP `1194` from IP (Default OpenVPN)
* Domain (optional, for persistent DNS)
  
![Screenshot](Photos/Screenshot%202025-06-21%20211132.png)


## Step-by-Step Installation

### 1. Launch EC2 Ubuntu Instance

Set the security group rules as follows:

| Port | Protocol | Source (IP) | Purpose                |
| ---- | -------- | ----------- | ---------------------- |
| 22   | TCP      | IP     | SSH                    |
| 943  | TCP      | IP     | Admin Web UI           |
| 443  | TCP      | IP     | TLS VPN                |
| 1194 | UDP      | IP     | OpenVPN default tunnel |

![Screenshot](Photos/Screenshot%202025-06-21%20211920.png)


### 2. Connect to EC2 & Install OpenVPN

```bash
ssh -i "key.pem" ubuntu@ec2-public-ip
```


Add OpenVPN's repo and install OpenVPN:

```bash
sudo mkdir -p /etc/apt/keyrings
sudo wget https://packages.openvpn.net/as-repo-public.asc -qO /etc/apt/keyrings/as-repository.asc

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/as-repository.asc] \
  http://packages.openvpn.net/as/debian jammy main" | sudo tee /etc/apt/sources.list.d/openvpn-as-repo.list

sudo apt update
sudo apt install -y openvpn-as
```

![Screenshot](Photos/Screenshot%202025-06-21%20200828.png)  


### 3. Access the Admin Web UI

Visit:

```
https://<ec2-public-ip>:943/admin
```

Login using:

* **Username:** `openvpn`
* **Password:** set during setup  

![Screenshot](Photos/Screenshot%202025-06-21%20212355.png)  


### 4. Create & Configure a User

From the Admin Portal:

```
User Management → Add User
```

* Enable:

  * `Allow Auto-login`
  * `Default (Local)` authentication

![Screenshot](Photos/Screenshot%202025-06-21%20202309.png)


### 5. Enable MFA (TOTP)

Under the same user:

* Set `Require MFA` to `Enabled`
* Save and Apply

The user can now visit:

```
https://<ec2-public-ip>:943/
```

 QR code to scan using:

* Google Authenticator
* Microsoft Authenticator  

![Screenshot](Photos/Screenshot%202025-06-21%20204621.png)  


### 6. Fix Public IP in the .ovpn Profile

**Problem:** The .ovpn profile uses the private EC2 IP.

**Solution:**

1. Download the `.ovpn` profile from:

   ```
   https://<ec2-public-ip>:943/
   ```

![Screenshot](Photos/Screenshot%202025-06-21%20213003.png)



3. Open the file and edit:

   ```
   remote 172.31.x.x 1194
   ```

   Change to:

   ```
   remote <ec2-public-ip> 443
   ```


### 7. Automate User Creation & MFA Setup

```bash
nano create-openvpn-user.sh
```

Paste this:

```bash
#!/bin/bash

USERNAME="$1"
PASSWORD="$2"

if [[ -z "$USERNAME" ]]; then
  echo "Usage: $0 <username> [password]"
  exit 1
fi

if [[ -z "$PASSWORD" ]]; then
  read -s -p "Enter password for $USERNAME: " PASSWORD
  echo
fi

echo "Adding user: $USERNAME"
sudo useradd "$USERNAME" -s /sbin/nologin || echo "User may already exist."
echo "$USERNAME:$PASSWORD" | sudo chpasswd

echo "Enabling TOTP for $USERNAME..."
sudo sacli --user "$USERNAME" --key "prop_totp_enabled" --value "true" UserPropPut

echo "Allowing auto-login..."
sudo sacli --user "$USERNAME" --key "prop_autologin" --value "true" UserPropPut

echo "Finalizing user setup..."
sudo sacli --user "$USERNAME" --key "type" --value "user_connect" UserPropPut
sudo sacli --user "$USERNAME" UserPropCommit

echo "User '$USERNAME' created and configured!"
```

Make it executable:

```bash
chmod +x create-openvpn-user.sh
```

Run the script:

```bash
./create-openvpn-user.sh elijames mysecurepassword
```


## Security Hardening

### Elastic IP + DNS

* Assign an Elastic IP in AWS.
* Set an A record in your DNS provider pointing to that IP (e.g., `vpn.domain.com`).


### Install fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban --now
```


### Configure UFW (Firewall)

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 443/tcp
sudo ufw allow 943/tcp
sudo ufw allow 1194/udp
sudo ufw enable
```


## Done

You now have:

* OpenVPN Access Server up & running
* Admin UI secured
* MFA enabled per user
* Custom users with auto-login profiles
* Hardened firewall and fail2ban
* Optional DNS routing


## License

MIT — use and modify freely.

