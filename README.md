# RCC-K Guardians Gauntlet CTF - Complete Deployment Guide

This comprehensive guide will walk you through setting up the complete RCC-K Guardians Gauntlet CTF environment from scratch, including hardware configuration, virtualization setup, and all components of the CTF.

## Table of Contents
1. [Hardware Setup](#hardware-setup)
2. [ESXi Installation and Configuration](#esxi-installation-and-configuration)
3. [Virtual Machine Setup](#virtual-machine-setup)
4. [Network Configuration](#network-configuration)
5. [Ubuntu Server Installation](#ubuntu-server-installation)
6. [Base System Configuration](#base-system-configuration)
7. [Docker Installation](#docker-installation)
8. [CTF Environment Deployment](#ctf-environment-deployment)
9. [Challenge Containers Setup](#challenge-containers-setup)
10. [CTFd Platform Configuration](#ctfd-platform-configuration)
11. [Participant Environment](#participant-environment)
12. [Testing the Environment](#testing-the-environment)
13. [Running the Event](#running-the-event)
14. [Troubleshooting Guide](#troubleshooting-guide)
15. [Administrator Quick Reference](#administrator-quick-reference)

## Hardware Setup

### Required Hardware
- Dell Server with sufficient resources (recommended specs):
  - CPU: 8+ cores
  - RAM: 32GB+ 
  - Storage: 500GB+
  - Network: 2 NICs minimum
- Network switch (8+ port gigabit)
- Ethernet cables (Cat5e or better)
- USB flash drive (8GB+) for ESXi installation
- Laptop for accessing ESXi and managing the environment

### Physical Setup
1. **Rack or place the Dell server** in a secure location with proper power and cooling
2. **Connect the server to power** and to your network switch
3. **Connect network ports**:
   - NIC 1: Management network (will be used for ESXi management)
   - NIC 2: CTF network (will be isolated for the CTF environment)
4. **Connect the network switch** to power
5. **Prepare admin workstation** (laptop with network access to the server)

## ESXi Installation and Configuration

### Prepare ESXi Installation Media
1. **Download ESXi 8.0** from [VMware website](https://customerconnect.vmware.com/downloads/details?downloadGroup=ESXI80U1A&productId=1345&rPId=101390)
2. **Create bootable USB**:
   - Using [Rufus](https://rufus.ie/en/) (on Windows)
   - Or `dd` command (on Linux/macOS): `sudo dd if=VMware-VMvisor-Installer.iso of=/dev/sdX bs=1M status=progress`

### Install ESXi
1. **Insert USB drive** into the Dell server
2. **Boot from USB**:
   - Power on the server
   - Press F11 (or appropriate key) during startup to access boot menu
   - Select USB drive

3. **Follow ESXi installation prompts**:
   - Accept license agreement
   - Select destination disk (usually the primary drive)
   - Set root password (document this securely!)
   - Confirm installation (this will erase the destination disk)

4. **Complete installation**:
   - Remove USB drive when prompted
   - Server will reboot

### Initial ESXi Configuration
1. **Note the management IP** displayed on the ESXi DCUI (Direct Console User Interface)
2. **Access ESXi web interface** from your admin laptop browser: `https://<management-ip>`
3. **Login** with root and the password you set during installation
4. **Configure networking**:
   - Navigate to Networking > Virtual Switches
   - Ensure you have:
     - vSwitch0 (connected to your management NIC)
     - Create vSwitch1 (connected to NIC 2 for the CTF network)
   - Create port groups:
     - Management Network (on vSwitch0)
     - CTF Network (on vSwitch1)

5. **Configure time settings**:
   - Navigate to Host > Manage > System > Time & Date
   - Set correct time zone and NTP servers

## Virtual Machine Setup

### Create Ubuntu Server VM
1. **Download Ubuntu Server 22.04 LTS ISO** from [Ubuntu website](https://ubuntu.com/download/server)
2. **Upload ISO to ESXi**:
   - Navigate to Storage > Datastore Browser
   - Create a new folder named "ISOs"
   - Upload the Ubuntu ISO

3. **Create virtual machine**:
   - Click "Create/Register VM"
   - Select "Create a new virtual machine" > Next
   - Name: "RCC-K-CTF-Server"
   - Compatibility: ESXi 8.0
   - Guest OS Family: Linux
   - Guest OS Version: Ubuntu Linux (64-bit)
   - Click Next

4. **Select storage**:
   - Select the datastore with most free space
   - Click Next

5. **Customize settings**:
   - CPU: 4 vCPUs
   - Memory: 8GB
   - Hard disk: 100GB
   - CD/DVD Drive: Select the Ubuntu ISO
   - Network Adapter 1: Management Network
   - Network Adapter 2: CTF Network
   - Click Next

6. **Review settings** and click Finish

## Network Configuration

### Network Architecture Overview
┌────────────────────────────────────────────────┐
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

### IP Scheme
- **Management Network**:
  - Subnet: 10.0.0.0/24
  - ESXi Host: 10.0.0.10
  - Ubuntu VM (eth0): 10.0.0.20
  - Admin Laptop: DHCP (10.0.0.100-200)

- **CTF Network**:
  - Subnet: 192.168.1.0/24
  - Ubuntu VM (eth1): 192.168.1.134
  - Participant Laptops: DHCP (192.168.1.100-150)

- **Docker Containers**:
  - Subnet: 172.18.0.0/16
  - Container IPs: 172.18.0.2-15

## Ubuntu Server Installation

### Install Ubuntu Server
1. **Power on** the Ubuntu VM from ESXi interface
2. **Follow installation prompts**:
   - Select language: English
   - Keyboard configuration: US
   - Network connections:
     - eth0 (Management): DHCP
     - eth1 (CTF): Static IP (192.168.1.134/24)
   - Storage configuration: Use entire disk, set up as LVM
   - Profile setup:
     - Name: CTF Admin
     - Server name: ctf
     - Username: ctfadmin
     - Password: (create a secure password and document it)
   - SSH Setup: Install OpenSSH server
   - Featured Server Snaps: None

3. **Complete installation** and reboot

### Access Ubuntu Server
1. **Connect via SSH** from your admin laptop:
   ```bash
   ssh ctfadmin@10.0.0.20

2. Update the system:
   ```bash
   sudo apt update && sudo apt upgrade -y

Base System Configuration
Configure Network Interfaces

1. Edit netplan configuration:
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml

2. Update with the following configuration:
   ```bash
   network:
  version: 2
  ethernets:
    ens160:  # Management interface (actual name may differ)
      dhcp4: true
    ens192:  # CTF network interface (actual name may differ)
      dhcp4: no
      addresses: [192.168.1.134/24]

mkdir -p ~/rcc-k/challenges/crypto-data/html

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

      
