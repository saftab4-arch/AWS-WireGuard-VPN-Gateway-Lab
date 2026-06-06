# AWS WireGuard VPN Gateway Lab

## Project Overview

This project demonstrates how to build a secure VPN-based AWS environment where a Windows laptop securely connects to private AWS resources through a WireGuard VPN tunnel.

The VPN terminates on a dedicated Gateway EC2 instance located in a public subnet. The gateway forwards traffic to private EC2 instances running Nginx and MariaDB inside isolated private subnets.

The goal was to provide secure administrative access to private workloads without exposing application or database servers directly to the internet.

---

# Skills Demonstrated

- AWS VPC Design
- Public & Private Subnets
- Route Tables
- Internet Gateway
- NAT Gateway
- Security Groups
- EC2 Administration
- Amazon Linux 2023
- WireGuard VPN
- Linux Networking
- IP Forwarding
- Private Infrastructure Management
- Nginx Deployment
- MariaDB Deployment
- Network Troubleshooting
- VPN Routing
- Secure Database Isolation

---

# WireGuard Peer Configuration

WireGuard uses a peer-to-peer model where each endpoint trusts the other's public key.

In this lab:

| Device | VPN IP |
|----------|----------|
| AWS WireGuard Gateway | 10.10.10.1 |
| Windows Laptop | 10.10.10.2 |

---

## AWS WireGuard Gateway Configuration

File:

```bash
/etc/wireguard/wg0.conf
```

Configuration:

```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = <AWS_PRIVATE_KEY>

[Peer]
PublicKey = <WINDOWS_PUBLIC_KEY>
AllowedIPs = 10.10.10.2/32
```

Explanation:

- Interface section defines the VPN server.
- Address assigns the VPN IP to AWS.
- ListenPort specifies the WireGuard listening port.
- Peer section defines the trusted Windows laptop.
- AllowedIPs tells AWS which traffic belongs to that peer.

---

## Windows WireGuard Client Configuration

Configuration:

```ini
[Interface]
PrivateKey = <WINDOWS_PRIVATE_KEY>
Address = 10.10.10.2/24

[Peer]
PublicKey = <AWS_PUBLIC_KEY>
Endpoint = 44.215.72.253:51820
AllowedIPs = 10.0.0.0/16,10.10.10.0/24
PersistentKeepalive = 25
```

Explanation:

- Interface section defines the laptop VPN adapter.
- Address assigns the VPN IP to the laptop.
- Endpoint specifies the AWS public IP and WireGuard port.
- AllowedIPs defines which networks should travel through the VPN tunnel.
- PersistentKeepalive maintains connectivity through NAT devices and home routers.

---

## WireGuard Tunnel Flow

```text
Windows Laptop
VPN IP: 10.10.10.2

        │
        │ Encrypted WireGuard Tunnel
        │ UDP 51820
        ▼

AWS Gateway EC2
Public IP: 44.215.72.253
VPN IP: 10.10.10.1

        │
        │ IP Forwarding Enabled
        ▼

10.0.2.40 (Nginx)

10.0.3.198 (MariaDB)
```

---

## Tunnel Validation

### Verify VPN Handshake

AWS Gateway:

```bash
sudo wg
```

Expected Output:

```text
peer: <WINDOWS_PUBLIC_KEY>

latest handshake:
few seconds ago

transfer:
RX bytes
TX bytes
```

---

### Verify Gateway Reachability

From Windows Laptop:

```cmd
ping 10.10.10.1
```

Result:

PASS

---

### Verify Private Web Server Reachability

```cmd
ping 10.0.2.40
```

Result:

PASS

---

### Verify Private Database Reachability

```cmd
ping 10.0.3.198
```

Result:

PASS

---

## Key Networking Concepts Demonstrated

- WireGuard Peer Relationships
- Public Key Cryptography
- VPN Tunnel Establishment
- VPN Routing
- IP Forwarding
- Transit Gateway Pattern Using EC2
- Private Resource Access Through VPN
- Secure Remote Administration

# Architecture Diagram

```text
Windows Laptop
VPN IP: 10.10.10.2

            │
            │ WireGuard VPN Tunnel
            │ UDP 51820
            ▼

+-------------------------+
| Gateway EC2             |
| Public IP: 44.215.72.253|
| Private IP: 10.0.1.123  |
| VPN IP: 10.10.10.1      |
+-------------------------+

            │
            │ Route Forwarding
            ▼

+-------------------------+
| Nginx EC2               |
| Private IP: 10.0.2.40   |
| Private Subnet          |
+-------------------------+

            │
            │ TCP 3306
            ▼

+-------------------------+
| MariaDB EC2             |
| Private IP: 10.0.3.198  |
| Private Subnet          |
+-------------------------+
```

---

# Environment Details

## VPC

```
10.0.0.0/16
```

## Public Subnet

```
10.0.1.0/24
```

Contains:

- WireGuard Gateway EC2
- NAT Gateway

## Private Web Subnet

```
10.0.2.0/24
```

Contains:

- Nginx EC2

## Private Database Subnet

```
10.0.3.0/24
```

Contains:

- MariaDB EC2

---

# VPN Network

## WireGuard Network

```
10.10.10.0/24
```

Gateway VPN Address

```
10.10.10.1
```

Laptop VPN Address

```
10.10.10.2
```

---

# EC2 Instances

| Instance | Role | IP Address |
|-----------|-----------|-----------|
| wg-gateway | WireGuard VPN Gateway | 10.0.1.123 |
| nginx-private | Web Server | 10.0.2.40 |
| mariadb-private | Database Server | 10.0.3.198 |

---

# Security Groups

## Gateway Security Group

Allowed:

- UDP 51820 (WireGuard)
- TCP 22 (SSH)
- ICMP

Source:

- My Public IP

---

## Nginx Security Group

Allowed:

- TCP 22
- TCP 80
- ICMP

Source:

```
10.10.10.0/24
```

---

## MariaDB Security Group

Allowed:

- TCP 22
- ICMP

Source:

```
10.10.10.0/24
```

Database Access:

- TCP 3306
- Source = nginx-sg

This prevents VPN clients from directly accessing the database.

---

# Route Tables

## Public Route Table

```
0.0.0.0/0 → Internet Gateway
```

Associated with:

- Public Subnet

---

## Private Route Table

```
0.0.0.0/0 → NAT Gateway

10.10.10.0/24 → WireGuard Gateway Instance
```

Associated with:

- Private Web Subnet
- Private Database Subnet

---

# Build Process

## Step 1 – Create VPC

Created VPC:

```
10.0.0.0/16
```

---

## Step 2 – Create Subnets

Created:

- Public Subnet
- Private Web Subnet
- Private Database Subnet

---

## Step 3 – Create Internet Gateway

Attached Internet Gateway to VPC.

---

## Step 4 – Create NAT Gateway

Created NAT Gateway in public subnet.

Purpose:

- Allow private instances to download packages and updates.

---

## Step 5 – Create Route Tables

Configured:

### Public Route Table

```
0.0.0.0/0 → IGW
```

### Private Route Table

```
0.0.0.0/0 → NAT Gateway
10.10.10.0/24 → WireGuard Gateway
```

---

## Step 6 – Create Security Groups

Configured:

- VPN Access
- SSH Access
- Web Access
- Database Isolation

---

## Step 7 – Launch EC2 Instances

Launched:

- wg-gateway
- nginx-private
- mariadb-private

All instances:

- Amazon Linux 2023
- t3.micro

---

## Step 8 – Disable Source/Destination Check

Disabled Source/Destination Check on:

```
wg-gateway
```

Purpose:

Allow EC2 instance to act as a router.

---

## Step 9 – Install WireGuard

Installed WireGuard on Gateway EC2.

Generated:

- Private Key
- Public Key

Configured:

- VPN Server
- VPN Client
- Tunnel Peering

---

## Step 10 – Enable IP Forwarding

Enabled Linux packet forwarding.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

---

## Step 11 – Establish VPN Tunnel

Connected Windows WireGuard client to AWS WireGuard Gateway.

Verified:

```bash
ping 10.10.10.1
```

Result:

SUCCESS

---

## Step 12 – Install Nginx

SSH from laptop through VPN:

```bash
ssh -i New-Lab.pem ec2-user@10.0.2.40
```

Installed:

```bash
sudo dnf install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## Step 13 – Install MariaDB

SSH from laptop through VPN:

```bash
ssh -i New-Lab.pem ec2-user@10.0.3.198
```

Installed:

```bash
sudo dnf install mariadb105-server -y
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

---

# Validation Testing

## Test 1 – VPN Tunnel

```bash
ping 10.10.10.1
```

Result:

PASS

---

## Test 2 – Reach Nginx Server

```bash
ping 10.0.2.40
```

Result:

PASS

---

## Test 3 – Reach MariaDB Server

```bash
ping 10.0.3.198
```

Result:

PASS

---

## Test 4 – SSH Through VPN

```bash
ssh -i New-Lab.pem ec2-user@10.0.2.40
```

Result:

PASS

---

## Test 5 – Web Access

Opened browser:

```
http://10.0.2.40
```

Result:

PASS

Nginx default page displayed.

---

## Test 6 – Database Isolation

From laptop:

```powershell
Test-NetConnection 10.0.3.198 -Port 3306
```

Result:

FAILED

Expected:

PASS

Reason:

Database should not be directly accessible from VPN clients.

---

## Test 7 – Nginx to MariaDB Connectivity

From Nginx EC2:

```bash
nc -zv 10.0.3.198 3306
```

Result:

Connected

Expected:

PASS

Reason:

Only application tier should communicate with database tier.

---

# Troubleshooting

## Issue

Unable to SSH to Gateway.

### Cause

Security Group source IP mismatch.

### Resolution

Updated SSH rule to current public IP.

---

## Issue

WireGuard keys created in wrong location.

### Cause

Attempted creation without root permissions.

### Resolution

Switched to root:

```bash
sudo -i
```

---

## Issue

Private instances unreachable over VPN.

### Cause

Missing return route.

### Resolution

Added:

```
10.10.10.0/24 → wg-gateway
```

to private route table.

---

## Issue

WireGuard installation initially appeared slow.

### Cause

Earlier testing used smaller resources.

### Resolution

Rebuilt environment using:

```
t3.micro
```

---

# Lessons Learned

- VPN connectivity requires proper return routing.
- Source/Destination Check must be disabled on routing instances.
- WireGuard provides lightweight encrypted remote access.
- NAT Gateway enables package installation without exposing private instances.
- Security Groups and Route Tables must work together.
- Database servers should remain isolated from end users.
- AWS private infrastructure can be managed securely through VPN access.

---

# Resume Bullet

Built a secure AWS WireGuard VPN architecture providing encrypted access to private EC2 workloads through a dedicated VPN gateway, custom route tables, NAT Gateway, Linux IP forwarding, and security-group-based database isolation. Deployed and validated Nginx and MariaDB across private subnets while implementing end-to-end network troubleshooting and connectivity testing.

---

# Project Outcome

Successfully built and validated a secure VPN-based AWS environment where private web and database servers were accessible through an encrypted WireGuard tunnel while maintaining strict database isolation and layered network security.
