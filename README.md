# Guardians-Gaunlent-2025
Guardians Gauntlent 


RCC-K Guardians Gauntlet CTF - Complete Deployment Guide
This comprehensive guide will walk you through setting up the complete RCC-K Guardians Gauntlet CTF environment from scratch, including hardware configuration, virtualization setup, and all components of the CTF.
Table of Contents

Hardware Setup
ESXi Installation and Configuration
Virtual Machine Setup
Network Configuration
Ubuntu Server Installation
Base System Configuration
Docker Installation
CTF Environment Deployment
Challenge Containers Setup
CTFd Platform Configuration
Participant Environment
Testing the Environment
Running the Event
Troubleshooting Guide
Administrator Quick Reference

Hardware Setup
Required Hardware

Dell Server with sufficient resources (recommended specs):

CPU: 8+ cores
RAM: 32GB+
Storage: 500GB+
Network: 2 NICs minimum


Network switch (8+ port gigabit)
Ethernet cables (Cat5e or better)
USB flash drive (8GB+) for ESXi installation
Laptop for accessing ESXi and managing the environment

Physical Setup

Rack or place the Dell server in a secure location with proper power and cooling
Connect the server to power and to your network switch
Connect network ports:

NIC 1: Management network (will be used for ESXi management)
NIC 2: CTF network (will be isolated for the CTF environment)


Connect the network switch to power
Prepare admin workstation (laptop with network access to the server)

ESXi Installation and Configuration
Prepare ESXi Installation Media

Download ESXi 8.0 from VMware website
Create bootable USB:

Using Rufus (on Windows)
Or dd command (on Linux/macOS): sudo dd if=VMware-VMvisor-Installer.iso of=/dev/sdX bs=1M status=progress



Install ESXi

Insert USB drive into the Dell server
Boot from USB:

Power on the server
Press F11 (or appropriate key) during startup to access boot menu
Select USB drive


Follow ESXi installation prompts:

Accept license agreement
Select destination disk (usually the primary drive)
Set root password (document this securely!)
Confirm installation (this will erase the destination disk)


Complete installation:

Remove USB drive when prompted
Server will reboot



Initial ESXi Configuration

Note the management IP displayed on the ESXi DCUI (Direct Console User Interface)
Access ESXi web interface from your admin laptop browser: https://<management-ip>
Login with root and the password you set during installation
Configure networking:

Navigate to Networking > Virtual Switches
Ensure you have:

vSwitch0 (connected to your management NIC)
Create vSwitch1 (connected to NIC 2 for the CTF network)


Create port groups:

Management Network (on vSwitch0)
CTF Network (on vSwitch1)




Configure time settings:

Navigate to Host > Manage > System > Time & Date
Set correct time zone and NTP servers



Virtual Machine Setup
Create Ubuntu Server VM

Download Ubuntu Server 22.04 LTS ISO from Ubuntu website
Upload ISO to ESXi:

Navigate to Storage > Datastore Browser
Create a new folder named "ISOs"
Upload the Ubuntu ISO


Create virtual machine:

Click "Create/Register VM"
Select "Create a new virtual machine" > Next
Name: "RCC-K-CTF-Server"
Compatibility: ESXi 8.0
Guest OS Family: Linux
Guest OS Version: Ubuntu Linux (64-bit)
Click Next


Select storage:

Select the datastore with most free space
Click Next


Customize settings:

CPU: 4 vCPUs
Memory: 8GB
Hard disk: 100GB
CD/DVD Drive: Select the Ubuntu ISO
Network Adapter 1: Management Network
Network Adapter 2: CTF Network
Click Next


Review settings and click Finish

Network Configuration
Network Architecture Overview
Copy┌────────────────────────────────────────────────┐
│                 Dell Server                    │
│ ┌──────────────────────────────────────────┐   │
│ │               ESXi 8.0                   │   │
│ │                                          │   │
│ │  ┌────────────────┐                      │   │
│ │  │  CTF Server VM │                      │   │
│ │  │ (Ubuntu 22.04) │                      │   │
│ │  └────────────────┘                      │   │
│ │                                          │   │
│ └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
          │                      │
          │                      │
          ▼                      ▼
  ┌───────────────┐      ┌───────────────┐
  │  Management   │      │  CTF Network  │
  │    Network    │      │   Switch      │
  └───────────────┘      └───────────────┘
          │                      │
          ▼                      │
  ┌───────────────┐              │
  │    Admin      │              │
  │   Laptop      │              │
  └───────────────┘              ▼
                         ┌───────────────┐
                         │  Participant  │
                         │   Laptops     │
                         └───────────────┘
IP Scheme

Management Network:

Subnet: 10.0.0.0/24
ESXi Host: 10.0.0.10
Ubuntu VM (eth0): 10.0.0.20
Admin Laptop: DHCP (10.0.0.100-200)


CTF Network:

Subnet: 192.168.1.0/24
Ubuntu VM (eth1): 192.168.1.134
Participant Laptops: DHCP (192.168.1.100-150)


Docker Containers:

Subnet: 172.18.0.0/16
Container IPs: 172.18.0.2-15



Ubuntu Server Installation
Install Ubuntu Server

Power on the Ubuntu VM from ESXi interface
Follow installation prompts:

Select language: English
Keyboard configuration: US
Network connections:

eth0 (Management): DHCP
eth1 (CTF): Static IP (192.168.1.134/24)


Storage configuration: Use entire disk, set up as LVM
Profile setup:

Name: CTF Admin
Server name: ctf
Username: ctfadmin
Password: (create a secure password and document it)


SSH Setup: Install OpenSSH server
Featured Server Snaps: None


Complete installation and reboot

Access Ubuntu Server

Connect via SSH from your admin laptop:
bashCopyssh ctfadmin@10.0.0.20

Update the system:
bashCopysudo apt update && sudo apt upgrade -y


Base System Configuration
Configure Network Interfaces

Edit netplan configuration:
bashCopysudo nano /etc/netplan/00-installer-config.yaml

Update with the following configuration:
yamlCopynetwork:
  version: 2
  ethernets:
    ens160:  # Management interface (actual name may differ)
      dhcp4: true
    ens192:  # CTF network interface (actual name may differ)
      dhcp4: no
      addresses: [192.168.1.134/24]

Apply the configuration:
bashCopysudo netplan apply

Verify the configuration:
bashCopyip addr show


Install Essential Packages
bashCopysudo apt install -y net-tools curl wget git vim ufw python3-pip openssh-server
Configure Firewall
bashCopy# Allow SSH on management interface
sudo ufw allow in on ens160 to any port 22

# Allow all traffic on CTF network
sudo ufw allow in on ens192

# Enable firewall
sudo ufw enable

# Verify status
sudo ufw status
Configure SSH
bashCopysudo nano /etc/ssh/sshd_config

Ensure PasswordAuthentication yes is set
Restart SSH: sudo systemctl restart sshd

Docker Installation
Install Docker Engine
bashCopy# Add Docker's official GPG key
sudo apt install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Install Docker Compose
sudo apt install -y docker-compose

# Add user to docker group
sudo usermod -aG docker ctfadmin

# Start and enable Docker
sudo systemctl enable docker
sudo systemctl start docker
Verify Docker Installation
bashCopy# Log out and log back in for group membership to take effect
exit
ssh ctfadmin@10.0.0.20

# Test Docker installation
docker --version
docker-compose --version
docker run hello-world
CTF Environment Deployment
Create Directory Structure
bashCopy# Create main directory
mkdir -p ~/rcc-k

# Create subdirectories
mkdir -p ~/rcc-k/ctfd
mkdir -p ~/rcc-k/story-portal
mkdir -p ~/rcc-k/challenges
Create Docker Network
bashCopy# Create the main network for challenges
docker network create --subnet=172.18.0.0/16 rcc-k-net

# Create additional network for internal services
docker network create --subnet=172.19.0.0/16 --internal secret-network
Deploy CTFd Platform

Create docker-compose.yml:

bashCopycat > ~/rcc-k/ctfd/docker-compose.yml << 'EOF'
version: '3'

services:
  ctfd:
    image: ctfd/ctfd:latest
    restart: always
    ports:
      - "8000:8000"
    volumes:
      - ./data:/var/lib/CTFd
    environment:
      - REVERSE_PROXY=false
      - SECRET_KEY=rcc_k_guardians_secret_key_2024
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.2

networks:
  rcc-k-net:
    external: true
EOF

Start CTFd platform:

bashCopycd ~/rcc-k/ctfd
mkdir -p data
docker-compose up -d
Deploy Story Portal

Create directory structure:

bashCopymkdir -p ~/rcc-k/story-portal/content

Create Dockerfile:

bashCopycat > ~/rcc-k/story-portal/Dockerfile << 'EOF'
FROM nginx:alpine
COPY content /usr/share/nginx/html
EOF

Create docker-compose.yml:

bashCopycat > ~/rcc-k/story-portal/docker-compose.yml << 'EOF'
version: '3'

services:
  story-portal:
    build: .
    container_name: rcc-k-story-portal
    ports:
      - "80:80"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.10
    restart: always

networks:
  rcc-k-net:
    external: true
EOF

Create index.html:

bashCopycat > ~/rcc-k/story-portal/content/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>RCC-K Guardians Gauntlet</title>
    <style>
        body { font-family: 'Courier New', monospace; max-width: 800px; margin: 0 auto; padding: 20px; background-color: #000; color: #0f0; }
        h1 { color: #ff0; text-align: center; border-bottom: 2px solid #0f0; padding-bottom: 10px; }
        .mission { background-color: #001100; padding: 15px; border: 1px solid #0f0; margin: 20px 0; }
        .alert { color: #f00; }
        a { color: #0ff; }
        pre { background-color: #001100; padding: 10px; border-left: 3px solid #0f0; }
    </style>
</head>
<body>
    <h1>RCC-K GUARDIANS GAUNTLET</h1>
    
    <div class="mission">
        <h2>MISSION BRIEFING - CLASSIFIED</h2>
        <p>Welcome, Guardians. A sophisticated Advanced Persistent Threat (APT) has infiltrated the RCC-K network. 
        Intelligence indicates they are after our classified research data.</p>
        
        <p>The attackers have established multiple footholds across our infrastructure.
        Your mission is to identify all compromised systems, collect evidence of intrusion,
        and secure our network before critical data is exfiltrated.</p>
        
        <p class="alert">WARNING: The attackers appear to be monitoring our security response.
        Proceed with caution.</p>
    </div>
    
    <h2>INTELLIGENCE REPORT</h2>
    <ul>
        <li>Network scan detected unusual traffic at multiple points in our infrastructure</li>
        <li>Authentication logs show suspicious login attempts</li>
        <li>Several encrypted messages have been intercepted</li>
        <li>File integrity checks have identified potentially compromised servers</li>
        <li>A compromised employee workstation was discovered</li>
    </ul>
    
    <h2>AVAILABLE RESOURCES</h2>
    <pre>
    NETWORK MAP (PARTIAL):
    ├── Admin Portal: 192.168.1.134:8080
    ├── File Server: 192.168.1.134:445 (SMB)
    ├── Operations Server: 192.168.1.134:2201 (SSH)
    ├── Employee Workstation: 192.168.1.134:2202 (SSH)
    ├── Crypto Challenges: 192.168.1.134:8001
    ├── Hidden Service: ??? (find it!)
    └── Web Vulnerabilities: ??? (find them!)
    </pre>
    
    <p>Submit your findings to headquarters: <a href="http://192.168.1.134:8000" target="_blank">CTF Platform</a></p>
    
    <div class="mission">
        <h2>FIELD NOTES</h2>
        <p>From previous incident response, we know the attackers favor default credentials,
        weak file permissions, and unpatched vulnerabilities. They are also known to leave
        encrypted messages as they move through networks.</p>
    </div>
</body>
</html>
EOF

Start the story portal:

bashCopycd ~/rcc-k/story-portal
docker-compose up -d
Challenge Containers Setup
Prepare Challenge Containers

Create the docker-compose.yml for challenges:

bashCopycat > ~/rcc-k/challenges/docker-compose.yml << 'EOF'
version: '3'

services:
  #
  # LINUX FUNDAMENTALS CHALLENGES
  #
  linux-basics:
    image: ubuntu:20.04
    container_name: linux-basics
    hostname: operations-server
    command: >
      /bin/bash -c "
      apt-get update && apt-get install -y openssh-server sudo vim cron &&
      useradd -m -s /bin/bash guardian &&
      echo 'guardian:guardianpassword' | chpasswd &&
      echo 'guardian ALL=(ALL) NOPASSWD: /usr/bin/find' >> /etc/sudoers &&
      mkdir -p /var/run/sshd &&
      mkdir -p /secret &&
      echo 'FLAG{basic_reconnaissance_complete}' > /secret/level1_flag.txt &&
      chmod 644 /secret/level1_flag.txt &&
      echo 'FLAG{sudo_privileges_exploited}' > /root/level2_flag.txt &&
      echo 'Found something interesting? The next clue is hidden in a dot file.' > /home/guardian/.hidden_note &&
      echo 'FLAG{hidden_files_discovered}' > /home/guardian/.secret_flag.txt &&
      mkdir -p /var/backups/ &&
      echo 'FLAG{scheduled_tasks_examined}' > /var/backups/cron_flag.txt &&
      (crontab -l 2>/dev/null; echo '*/5 * * * * cat /var/backups/cron_flag.txt > /dev/null') | crontab - &&
      service ssh start &&
      tail -f /dev/null
      "
    ports:
      - "2201:22"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.5
    restart: always

  #
  # FILE SERVER (SMB)
  #
  file-server:
    image: dperson/samba
    container_name: file-server
    hostname: file-server
    ports:
      - "445:445"
    environment:
      - USERID=1000
      - GROUPID=1000
      - TZ=UTC
      - USER=user;password
      - ADMIN=administrator;adminpass
      - SMB="security = user"
      - SHARE=public;/share/public;yes;no;yes;all
      - SHARE2=private;/share/private;no;no;no;user,administrator
      - SHARE3=restricted;/share/restricted;no;no;no;administrator
    volumes:
      - ./samba-data:/share
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.3
    restart: always

  #
  # CRYPTOGRAPHY CHALLENGES
  #
  crypto-server:
    image: nginx:alpine
    container_name: crypto-server
    volumes:
      - ./crypto-data/html:/usr/share/nginx/html
    ports:
      - "8001:80"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.7
    restart: always

  #
  # ADMIN PORTAL
  #
  admin-portal:
    image: nginx:alpine
    container_name: admin-portal
    volumes:
      - ./admin-portal-data:/usr/share/nginx/html
    ports:
      - "8080:80"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.12
    restart: always

  #
  # EMPLOYEE WORKSTATION
  #
  employee-workstation:
    image: ubuntu:20.04
    container_name: employee-workstation
    hostname: employee-workstation
    command: >
      /bin/bash -c "
      apt-get update && apt-get install -y openssh-server sudo vim curl wget netcat &&
      useradd -m -s /bin/bash employee &&
      echo 'employee:employeepass' | chpasswd &&
      mkdir -p /var/run/sshd &&
      mkdir -p /home/employee/Documents &&
      mkdir -p /home/employee/Downloads &&
      echo 'To: IT Support\nFrom: John\nSubject: Password Reset\n\nI forgot my password again for the database. Can you reset it to the usual format: RCC-K_spring_2024?' > /home/employee/Documents/email.txt &&
      echo 'FLAG{employee_workstation_accessed}' > /home/employee/Documents/personal_notes.txt &&
      echo '#!/bin/bash\nnc -l -p 9999 -e /bin/bash' > /home/employee/.backdoor.sh &&
      chmod +x /home/employee/.backdoor.sh &&
      echo 'FLAG{hidden_backdoor_discovered}' > /home/employee/.backdoor_flag.txt &&
      echo 'admin:21232f297a57a5a743894a0e4a801fc3' > /home/employee/Downloads/hash.txt &&
      echo 'FLAG{network_listening_service_found}' > /home/employee/listener_flag.txt &&
      service ssh start &&
      /home/employee/.backdoor.sh &
      tail -f /dev/null
      "
    ports:
      - "2202:22"
      - "9999:9999"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.6
    restart: always

  #
  # NETWORK RECON - HIDDEN SERVICE
  #
  hidden-service:
    image: nginx:alpine
    container_name: hidden-service
    volumes:
      - ./hidden-data:/usr/share/nginx/html
    ports:
      - "8888:80"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.9
    restart: always
    
  #
  # VULNERABLE WEBAPP
  #
  vulnerable-webapp:
    image: nginx:alpine
    container_name: vulnerable-webapp
    volumes:
      - ./webapp-data:/usr/share/nginx/html
    ports:
      - "8081:80"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.8
    restart: always
    
  #
  # API SERVER
  #
  internal-api:
    image: python:3.9-alpine
    container_name: internal-api
    command: /bin/sh -c "pip install flask && python /app/api.py"
    volumes:
      - ./api-data:/app
    ports:
      - "5000:5000"
    networks:
      rcc-k-net:
        ipv4_address: 172.18.0.11
    restart: always

networks:
  rcc-k-net:
    external: true
EOF
Create Challenge Content Files

Create SMB file directories:

bashCopymkdir -p ~/rcc-k/challenges/samba-data/public
mkdir -p ~/rcc-k/challenges/samba-data/private
mkdir -p ~/rcc-k/challenges/samba-data/restricted

Create SMB public share files:

bashCopycat > ~/rcc-k/challenges/samba-data/public/welcome.txt << 'EOF'
Welcome to the RCC-K File Server

This server contains operational data for day-to-day activities.
If you need access to the private share, please contact IT support.

- Admin Team
EOF

cat > ~/rcc-k/challenges/samba-data/public/memo.txt << 'EOF'
MEMO: Security Reminder

Please remember to follow proper security protocols:
1. Do not share passwords
2. Lock your workstation when away
3. Report suspicious activities

Also, reminder that the weekly password for the operations database
follows the format: rcc-k_[current_month]_[current_year]
EOF

cat > ~/rcc-k/challenges/samba-data/public/smb_flag.txt << 'EOF'
FLAG{public_share_accessed}
EOF

Create SMB private share files:

bashCopycat > ~/rcc-k/challenges/samba-data/private/backup.sql << 'EOF'
-- Database backup 2023-11-15
-- WARNING: This file contains sensitive information

CREATE TABLE users (
  id INT PRIMARY KEY,
  username VARCHAR(50),
  password VARCHAR(100),
  role VARCHAR(20)
);

INSERT INTO users VALUES (1, 'admin', 'sha256:1000:K8fsjbz5fFjJrcue:9b948d2e8d834cc31a3b94618339ea3723a344971c14d08bc0c16de8cd3c963a', 'administrator');
INSERT INTO users VALUES (2, 'jsmith', 'sha256:1000:mPrtD438jsdfs:a232d5fef93e7fac5be1e9bba16dc755ca661c0a9a861ee5c07d55a6283de4d2', 'user');
INSERT INTO users VALUES (3, 'analyst', 'sha256:1000:lse45DpsFBD:8d4488c2ac52abb6c3a423709a8f6e7f9cf8e94ead457d9a4bce2f4a92cb3aca', 'analyst');

-- FLAG{database_backup_compromised}
EOF

cat > ~/rcc-k/challenges/samba-data/private/access_codes.txt << 'EOF'
RESTRICTED - LEVEL 3 CLEARANCE REQUIRED

Server Room: 7863
Operations Center: 5291
Research Lab: 3478

Emergency Override: FLAG{private_share_accessed}
EOF

Create SMB restricted share files:

bashCopycat > ~/rcc-k/challenges/samba-data/restricted/admin_notes.txt << 'EOF'
ADMIN ONLY - DO NOT SHARE

Administrator backdoor access:
URL: http://192.168.1.134:8080/admin.html
Username: superadmin
Password: RCC-K_adm1n_2024!

FLAG{restricted_share_accessed}
EOF

Create cryptography challenge files:

bashCopymkdir -p ~/rcc-k/challenges/crypto-data/html

cat > ~/rcc-k/challenges/crypto-data/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>RCC-K Communications Center</title>
    <style>
        body { font-family: 'Courier New', monospace; background-color: #000; color: #0f0; padding: 20px; }
        .terminal { background-color: #001100; padding: 15px; border: 1px solid #0f0; margin-bottom: 20px; }
        h1, h2 { color: #0f0; }
        .challenge { margin-bottom: 30px; }
    </style>
</head>
<body>
    <h1>RCC-K Communications Center</h1>
    
    <div class="terminal">
        <p>Welcome to the communications center. Several encrypted messages have been intercepted. Decrypt them to reveal critical intelligence.</p>
    </div>
    
    <div class="challenge">
        <h2>Challenge 1: Basic Encoding</h2>
        <div class="terminal">
            <p>Intercepted message:</p>
            <p>RkxBR3tiYXNlNjRfZGVjb2RlZF9zdWNjZXNzZnVsbHl9</p>
            <p><em>Hint: This is a common encoding method used for transmitting binary data over text-based systems.</em></p>
        </div>
    </div>
    
    <div class="challenge">
        <h2>Challenge 2: Caesar's Legacy</h2>
        <div class="terminal">
            <p>Intercepted message:</p>
            <p>IODJ{fdhvdu_flskhu_ghfubswhg}</p>
            <p><em>Hint: Julius would be proud. The shift is 3.</em></p>
        </div>
    </div>
    
    <div class="challenge">
        <h2>Challenge 3: Substitution Cipher</h2>
        <div class="terminal">
            <p>Intercepted message:</p>
            <p>NTGW{lgfmqbqlgqbhf_zbekdi_zvfqogzqdo}</p>
            <p><em>Hint: A=Z, B=Y, C=X...</em></p>
        </div>
    </div>
    
    <div class="challenge">
        <h2>Challenge 4: Vigenère Cipher</h2>
        <div class="terminal">
            <p>Intercepted message:</p>
            <p>LFSI{GSQTICFSR_VM_ZMKIRSTV_GMTLVV}</p>
            <p><em>Hint: The key is "RCCKG"</em></p>
        </div>
    </div>
    
    <div class="challenge">
        <h2>Challenge 5: Binary Code</h2>
        <div class="terminal">
            <p>Intercepted message:</p>
            <p>01000110 01001100 01000001 01000111 01111011 01100010 01101001 01101110 01100001 01110010 01111001 01011111 01100100 01100101 01100011 01101111 01100100 01100101 01100100 01111101</p>
            <p><em>Hint: The foundation of all digital communications.</em></p>
        </div>
    </div>
</body>
</html>
EOF

Create hidden service files:

bashCopymkdir -p ~/rcc-k/challenges/hidden-data

cat > ~/rcc-k/challenges/hidden-data/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>RCC-K Internal System</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; background-color: #f0f0f0; }
        .secret { color: white; background-color: white; }
        .secret:hover { color: red; }
    </style>
</head>
<body>
    <h1>RCC-K Hidden Service</h1>
    <p>Congratulations on discovering this hidden service!</p>
    <p>FLAG{hidden_service_discovered}</p>
    
    <h2>Internal Notes</h2>
    <p>The secure communication channel is at 192.168.1.134:5000/api/status</p>
    
    <div class="secret">
        Emergency access code: 7355608
    </div>
</body>
</html>
EOF

Create admin portal files:

bashCopymkdir -p ~/rcc-k/challenges/admin-portal-data

cat > ~/rcc-k/challenges/admin-portal-data/index.html << 'EOF'
<!DOCTYPE html>
<html>RetryClaude hit the max length for a message and has paused its response. You can write Continue to keep the chat going. Claude does not have internet access. Links provided may not be accurate or up to date.KcontinueEdit
Create admin portal files (continued):

bashCopycat > ~/rcc-k/challenges/admin-portal-data/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Admin Portal</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f4f4f4; margin: 0; padding: 20px; }
        .login-form { background: white; max-width: 400px; margin: 100px auto; padding: 20px; border-radius: 5px; box-shadow: 0 0 10px rgba(0,0,0,0.1); }
        h1 { color: #333; text-align: center; }
        input[type="text"], input[type="password"] { width: 100%; padding: 10px; margin: 10px 0; box-sizing: border-box; }
        button { background: #4CAF50; color: white; border: none; padding: 10px; width: 100%; cursor: pointer; }
        .error { color: red; text-align: center; }
    </style>
</head>
<body>
    <div class="login-form">
        <h1>Admin Portal</h1>
        <form id="loginForm">
            <div>
                <label for="username">Username:</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div>
                <label for="password">Password:</label>
                <input type="password" id="password" name="password" required>
            </div>
            <div>
                <button type="submit">Login</button>
            </div>
        </form>
        <p id="message" class="error"></p>
    </div>

    <script>
        document.getElementById('loginForm').addEventListener('submit', function(e) {
            e.preventDefault();
            
            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;
            
            if (username === "superadmin" && password === "RCC-K_adm1n_2024!") {
                window.location.href = "admin.html";
            } else {
                document.getElementById('message').textContent = "Invalid username or password";
            }
        });
    </script>
</body>
</html>
EOF

cat > ~/rcc-k/challenges/admin-portal-data/admin.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Admin Dashboard</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; max-width: 800px; margin: 0 auto; }
        .panel { border: 1px solid #ccc; padding: 20px; border-radius: 5px; margin-top: 20px; }
    </style>
</head>
<body>
    <h1>Admin Dashboard</h1>
    <p>Welcome, Administrator!</p>
    
    <div class="panel">
        <h2>System Status</h2>
        <p>All systems operational.</p>
        <p>FLAG{admin_portal_accessed}</p>
    </div>
</body>
</html>
EOF

Create vulnerable webapp files:

bashCopymkdir -p ~/rcc-k/challenges/webapp-data

cat > ~/rcc-k/challenges/webapp-data/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>RCC-K Internal Portal</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; max-width: 800px; margin: 0 auto; }
        .login-form { border: 1px solid #ccc; padding: 20px; border-radius: 5px; margin-top: 20px; }
        input[type="text"], input[type="password"] { width: 100%; padding: 8px; margin: 8px 0; }
        button { background-color: #4CAF50; color: white; padding: 10px 15px; border: none; cursor: pointer; }
    </style>
</head>
<body>
    <h1>RCC-K Internal Portal</h1>
    <p>Please login to access the system.</p>
    
    <div class="login-form">
        <form id="loginForm">
            <div>
                <label for="username">Username:</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div>
                <label for="password">Password:</label>
                <input type="password" id="password" name="password" required>
            </div>
            <button type="submit">Login</button>
        </form>
    </div>
    
    <div id="message" style="margin-top: 20px; color: red;"></div>
    
    <script>
        document.getElementById('loginForm').addEventListener('submit', function(e) {
            e.preventDefault();
            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;
            
            // Very insecure validation - client-side only
            if (username === "superadmin" && password === "RCC-K_adm1n_2024!") {
                window.location.href = "admin.php";
            } else {
                document.getElementById('message').textContent = "Invalid username or password";
            }
        });
    </script>
    
    <!-- Hidden comment with hint -->
    <!-- Note: Admin access is at /admin.php - Default credentials in the restricted share -->
</body>
</html>
EOF

cat > ~/rcc-k/challenges/webapp-data/admin.php << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>RCC-K Admin Portal</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; max-width: 800px; margin: 0 auto; }
        .panel { border: 1px solid #ccc; padding: 20px; border-radius: 5px; margin-top: 20px; }
    </style>
</head>
<body>
    <h1>RCC-K Admin Portal</h1>
    <p>Welcome, Administrator!</p>
    
    <div class="panel">
        <h2>System Status</h2>
        <p>All systems operational.</p>
        <p>FLAG{admin_portal_accessed}</p>
    </div>
</body>
</html>
EOF

Create API server files:

bashCopymkdir -p ~/rcc-k/challenges/api-data

cat > ~/rcc-k/challenges/api-data/api.py << 'EOF'
from flask import Flask, jsonify, request
import json

app = Flask(__name__)

# Insecure API key validation
API_KEY = "RCC-K-API-KEY-2024"

@app.route('/')
def index():
    return jsonify({"message": "RCC-K API Server", "status": "running"})

@app.route('/api/status')
def status():
    return jsonify({"status": "online", "message": "API is operational", "hint": "Try /api/data with a valid API key"})

@app.route('/api/data')
def data():
    api_key = request.args.get('key', '')
    if api_key != API_KEY:
        return jsonify({"error": "Invalid API key", "hint": "The key format is RCC-K-API-KEY-YYYY"}), 403
    
    return jsonify({
        "message": "Access granted",
        "flag": "FLAG{api_key_discovered}",
        "hint": "For more information, try /api/secret"
    })

@app.route('/api/secret')
def secret():
    api_key = request.args.get('key', '')
    if api_key != API_KEY:
        return jsonify({"error": "Invalid API key"}), 403
    
    command = request.args.get('cmd', '')
    if command:
        # Intentionally vulnerable to command injection
        try:
            import os
            output = os.popen(command).read()
            return jsonify({"output": output, "flag": "FLAG{command_injection_successful}"})
        except Exception as e:
            return jsonify({"error": str(e)}), 500
    
    return jsonify({"message": "Use the cmd parameter to execute commands", "example": "?key=RCC-K-API-KEY-2024&cmd=ls"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

Start the challenge containers:

bashCopycd ~/rcc-k/challenges
docker-compose up -d
Configure DHCP for Participant Network

Install DHCP server:

bashCopysudo apt install -y isc-dhcp-server

Configure DHCP server:

bashCopysudo nano /etc/dhcp/dhcpd.conf

Add the following configuration:

Copydefault-lease-time 600;
max-lease-time 7200;
option subnet-mask 255.255.255.0;
option broadcast-address 192.168.1.255;
option routers 192.168.1.134;
option domain-name-servers 8.8.8.8, 8.8.4.4;

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.150;
}

Set the DHCP server interface:

bashCopysudo nano /etc/default/isc-dhcp-server

Edit the file to set the interface:

CopyINTERFACESv4="ens192"  # Use the actual interface name for the CTF network

Start the DHCP server:

bashCopysudo systemctl enable isc-dhcp-server
sudo systemctl restart isc-dhcp-server
CTFd Platform Configuration

Access CTFd web interface at http://192.168.1.134:8000
Complete the initial setup:

Create admin account:

Username: admin
Email: admin@rcc-k.local
Password: (use a secure password)


CTF Name: RCC-K Guardians Gauntlet
Theme selection: default


Configure CTF settings:

Navigate to Admin Panel > Config > General
Set CTF start time to current date
Set CTF end time to a future date
Save changes


Create challenge categories:

Navigate to Admin Panel > Challenges
Edit challenge categories (pencil icon)
Add the following categories:

Linux Fundamentals
File Access & Permissions
Cryptography
Password Cracking
Network Reconnaissance




Create challenges one by one (15 total challenges):

Navigate to Admin Panel > Challenges
Click the plus (+) button to create a new challenge
For each challenge, configure:

Name
Category
Value (points)
Description
Flag


Refer to the list of challenges with descriptions provided earlier


Make challenges visible to participants:

Navigate to Admin Panel > Config > Challenge Settings
Set Challenge Visibility to "Public"
Save changes



Create Environment Reset Script

Create the reset script:

bashCopycat > ~/rcc-k/reset.sh << 'EOF'
#!/bin/bash
echo "Resetting RCC-K Guardians Gauntlet environment..."

# Stop all containers
cd ~/rcc-k/challenges
docker-compose down

# Reset CTFd (if needed between iterations)
cd ~/rcc-k/ctfd
docker-compose down
rm -rf ~/rcc-k/ctfd/data/*
docker-compose up -d

# Wait for CTFd to start up
echo "Waiting for CTFd to initialize..."
sleep 10

# Reset SMB data
rm -rf ~/rcc-k/challenges/samba-data/public/*
rm -rf ~/rcc-k/challenges/samba-data/private/*
rm -rf ~/rcc-k/challenges/samba-data/restricted/*

# Recreate SMB data
mkdir -p ~/rcc-k/challenges/samba-data/public
mkdir -p ~/rcc-k/challenges/samba-data/private
mkdir -p ~/rcc-k/challenges/samba-data/restricted

# Public share files
cat > ~/rcc-k/challenges/samba-data/public/welcome.txt << 'EOFINNER'
Welcome to the RCC-K File Server

This server contains operational data for day-to-day activities.
If you need access to the private share, please contact IT support.

- Admin Team
EOFINNER

cat > ~/rcc-k/challenges/samba-data/public/memo.txt << 'EOFINNER'
MEMO: Security Reminder

Please remember to follow proper security protocols:
1. Do not share passwords
2. Lock your workstation when away
3. Report suspicious activities

Also, reminder that the weekly password for the operations database
follows the format: rcc-k_[current_month]_[current_year]
EOFINNER

cat > ~/rcc-k/challenges/samba-data/public/smb_flag.txt << 'EOFINNER'
FLAG{public_share_accessed}
EOFINNER

# Private share files
cat > ~/rcc-k/challenges/samba-data/private/backup.sql << 'EOFINNER'
-- Database backup 2023-11-15
-- WARNING: This file contains sensitive information

CREATE TABLE users (
  id INT PRIMARY KEY,
  username VARCHAR(50),
  password VARCHAR(100),
  role VARCHAR(20)
);

INSERT INTO users VALUES (1, 'admin', 'sha256:1000:K8fsjbz5fFjJrcue:9b948d2e8d834cc31a3b94618339ea3723a344971c14d08bc0c16de8cd3c963a', 'administrator');
INSERT INTO users VALUES (2, 'jsmith', 'sha256:1000:mPrtD438jsdfs:a232d5fef93e7fac5be1e9bba16dc755ca661c0a9a861ee5c07d55a6283de4d2', 'user');
INSERT INTO users VALUES (3, 'analyst', 'sha256:1000:lse45DpsFBD:8d4488c2ac52abb6c3a423709a8f6e7f9cf8e94ead457d9a4bce2f4a92cb3aca', 'analyst');

-- FLAG{database_backup_compromised}
EOFINNER

cat > ~/rcc-k/challenges/samba-data/private/access_codes.txt << 'EOFINNER'
RESTRICTED - LEVEL 3 CLEARANCE REQUIRED

Server Room: 7863
Operations Center: 5291
Research Lab: 3478

Emergency Override: FLAG{private_share_accessed}
EOFINNER

# Restricted share files
cat > ~/rcc-k/challenges/samba-data/restricted/admin_notes.txt << 'EOFINNER'
ADMIN ONLY - DO NOT SHARE

Administrator backdoor access:
URL: http://192.168.1.134:8080/admin.html
Username: superadmin
Password: RCC-K_adm1n_2024!

FLAG{restricted_share_accessed}
EOFINNER

# Start everything back up
cd ~/rcc-k/challenges
docker-compose up -d
cd ~/rcc-k/story-portal
docker-compose up -d

echo "Environment reset complete! Ready for next session."
EOF

chmod +x ~/rcc-k/reset.sh
Participant Environment
Preparing Participant Laptops

Hardware requirements:

Windows 10 laptops (5 or more)
Minimum 8GB RAM, 4 cores, 100GB free disk space
Network port (Ethernet)
USB ports for additional peripherals


Software requirements:

VirtualBox (or VMware Workstation)
Kali Linux VM image (2023.1 or newer)
Network cable for each laptop


Kali VM Configuration:

4GB RAM allocation
2 CPU cores
40GB disk space
Network adapter set to bridged mode


Verify tools installation:

nmap (network scanning)
smbclient (SMB file sharing)
hydra (password cracking)
john/hashcat (hash cracking)
Firefox (web browser)



Participant Guide

Create a participant guide document:

bashCopycat > ~/rcc-k/participant-guide.md << 'EOF'
# RCC-K Guardians Gauntlet - Participant Guide

## Overview
Welcome to the RCC-K Guardians Gauntlet! You have one hour to complete as many challenges as possible. Points are awarded based on difficulty, and the participant with the most points at the end wins.

## Getting Started
1. Connect your laptop to the provided Ethernet cable
2. Your laptop will receive an IP address automatically
3. Open a web browser and navigate to http://192.168.1.134
4. Follow the story portal instructions
5. Register at the CTF platform: http://192.168.1.134:8000

## Available Tools in Kali Linux
- Network scanning: nmap, netcat
- SMB tools: smbclient, enum4linux
- Cryptography: CyberChef (browser-based)
- Password cracking: hydra, john, hashcat
- Web tools: Burp Suite, curl, Firefox
- Remote access: SSH

## Challenge Categories
1. **Linux Fundamentals** - Basic system exploration and privilege escalation
2. **File Access & Permissions** - SMB shares with varying levels of access control
3. **Cryptography** - Decode various encrypted and encoded messages
4. **Password Cracking** - Find and crack passwords
5. **Network Reconnaissance** - Discover hidden services and backdoors

## Flag Format
All flags follow the format: FLAG{something_here}
Submit them exactly as found.

## Tips for Success
- Read all challenge descriptions carefully
- Take notes as you go
- If stuck, use the hint system (may cost points)
- Work methodically and don't panic - the challenges are designed to be solvable within one hour
- Remember that everything on the network is there for a reason

Good luck, Guardian!
EOF

Print copies of the participant guide for distribution

Testing the Environment
Verification Checklist

Network connectivity:

bashCopy# Test that Docker network is correctly set up
docker network inspect rcc-k-net

# Test that containers can communicate
docker exec linux-basics ping -c 3 172.18.0.3

# Test external access to services
curl http://192.168.1.134:8000  # CTFd
curl http://192.168.1.134:80    # Story portal

Container status:

bashCopy# Verify all containers are running
docker-compose -f ~/rcc-k/challenges/docker-compose.yml ps
docker-compose -f ~/rcc-k/ctfd/docker-compose.yml ps
docker-compose -f ~/rcc-k/story-portal/docker-compose.yml ps

Service accessibility:

bashCopy# SSH connections
ssh -p 2201 guardian@192.168.1.134    # Operations server
ssh -p 2202 employee@192.168.1.134    # Employee workstation

# SMB access
smbclient -L //192.168.1.134 -N
smbclient //192.168.1.134/public -N

# Web services
curl http://192.168.1.134:8001     # Cryptography challenges
curl http://192.168.1.134:8080     # Admin portal
curl http://192.168.1.134:8888     # Hidden service
curl http://192.168.1.134:5000     # API server

DHCP service:

bashCopy# Verify DHCP server is running
sudo systemctl status isc-dhcp-server

# Check DHCP configuration
cat /etc/dhcp/dhcpd.conf
Full CTF Walkthrough

Connect a test laptop to the network and verify:

It receives an IP address in the 192.168.1.100-150 range
It can access the CTFd platform and story portal
It can complete at least one challenge from each category


Test the reset script:

bashCopy# Run the reset script
~/rcc-k/reset.sh

# Verify that services are restarted and working
curl http://192.168.1.134:8000
Running the Event
Pre-Event Checklist

Environment preparation:

Run the reset script to ensure a clean environment
Verify all services are running
Test network connectivity


Physical setup:

Arrange participant workstations
Connect all laptops to the network switch
Ensure power is available for all devices


Administrative access:

Verify admin access to CTFd platform
Have administrative credentials ready for troubleshooting



During the Event

Introduction (5 minutes):

Welcome participants
Explain the CTF concept
Distribute participant guides
Point out the story portal and CTF platform URLs


Setup (5 minutes):

Help participants connect to the network
Assist with registration on the CTF platform
Ensure all participants can access the Kali VM


Competition (45 minutes):

Monitor progress on the CTFd scoreboard
Be available for questions (without giving away solutions)
Watch for any technical issues with the environment


Conclusion (5 minutes):

Announce winners
Provide a brief overview of solutions
Collect feedback for future events



Between Sessions

Reset the environment:

bashCopy~/rcc-k/reset.sh

Verify the reset was successful:

Check container status
Test a sample challenge
Ensure CTFd platform is ready for new registrations



Troubleshooting Guide
Common Issues and Solutions
Container Issues

Container not starting:

bashCopy# Check container logs
docker logs [container_name]

# Try restarting the container
docker restart [container_name]

# If needed, recreate the container
cd ~/rcc-k/challenges
docker-compose up -d [service_name]

Container networking issues:

bashCopy# Check network configuration
docker network inspect rcc-k-net

# Recreate network if necessary
docker network rm rcc-k-net
docker network create --subnet=172.18.0.0/16 rcc-k-net
cd ~/rcc-k/challenges
docker-compose down
docker-compose up -d
DHCP Issues

Participants not getting IP addresses:

bashCopy# Check DHCP server status
sudo systemctl status isc-dhcp-server

# Check DHCP logs
sudo journalctl -u isc-dhcp-server

# Restart DHCP server
sudo systemctl restart isc-dhcp-server

Wrong IP range assigned:

bashCopy# Verify DHCP configuration
cat /etc/dhcp/dhcpd.conf

# Ensure the interface is correct in configuration
cat /etc/default/isc-dhcp-server
CTFd Platform Issues

CTFd not accessible:

bashCopy# Check container status
docker ps | grep ctfd

# Check container logs
docker logs ctfd_ctfd_1

# Restart container
docker restart ctfd_ctfd_1

CTFd database issues:

bashCopy# Reset CTFd completely (last resort)
cd ~/rcc-k/ctfd
docker-compose down
rm -rf data/*
docker-compose up -d
# Then reconfigure the platform
Networking Issues

Can't connect to services:

bashCopy# Check if services are running
docker ps

# Check if services are listening on expected ports
sudo netstat -tulpn | grep [port]

# Verify firewall settings
sudo ufw status

Connection timeouts:

bashCopy# Check network interfaces
ip addr show

# Check network routes
ip route

# Test connectivity between networks
ping -c 3 192.168.1.134
Emergency Recovery Procedures

Full environment reset:

bashCopy# Stop all containers
cd ~/rcc-k/challenges
docker-compose down
cd ~/rcc-k/ctfd
docker-compose down
cd ~/rcc-k/story-portal
docker-compose down

# Remove all containers and networks
docker rm -f $(docker ps -aq)
docker network prune -f

# Recreate network
docker network create --subnet=172.18.0.0/16 rcc-k-net

# Restart all services
cd ~/rcc-k/ctfd
docker-compose up -d
cd ~/rcc-k/story-portal
docker-compose up -d
cd ~/rcc-k/challenges
docker-compose up -d

VM recovery (if the VM becomes corrupted):

Create a backup of the VM before the event
If needed, restore the VM from backup in ESXi:

Shut down the problematic VM
Deploy a new VM from the backup
Reconfigure networking if necessary





Administrator Quick Reference
Important Commands
Copy# Check system status
docker ps -a                  # List all containers
docker-compose ps             # Check service status
sudo systemctl status isc-dhcp-server  # Check DHCP server

# Network verification
ip addr show                  # Check IP addresses
ping -c 3 192.168.1.134       # Test connectivity
curl http://192.168.1.134:8000 # Test CTFd platform

# Service management
docker restart [container]     # Restart a specific container
~/rcc-k/reset.sh              # Reset the entire environment

# Troubleshooting
docker logs [container]       # View container logs
sudo journalctl -u isc-dhcp-server  # View DHCP logs
sudo netstat -tulpn           # Check listening ports
Key Information

IP Addresses:

CTF Server: 192.168.1.134
Docker Containers: 172.18.0.2-15
Participant Range: 192.168.1.100-150


Important Ports:

CTFd Platform: 8000
Story Portal: 80
Linux Server SSH: 2201
Employee Workstation SSH: 2202
SMB File Server: 445
Various web services: 8001, 8080, 8888, 5000


Critical Credentials:

CTFd Admin: (as configured)
Linux Server: guardian:guardianpassword
Employee Workstation: employee:employeepass
CTF Server: ctfadmin:(as configured)


File Locations:

CTF Environment: ~/rcc-k/
Reset Script: ~/rcc-k/reset.sh
Docker Compose Files:

~/rcc-k/challenges/docker-compose.yml
~/rcc-k/ctfd/docker-compose.yml
~/rcc-k/story-portal/docker-compose.yml
