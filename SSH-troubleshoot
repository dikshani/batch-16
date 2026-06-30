SSH Connection Timeout – Root Cause Analysis (AWS EC2)
Project Information

Project: Linux Server Setup and Static Site Deployment using Nginx
Environment: AWS EC2 (Ubuntu)
Access Method: SSH (Port 22)
Objective: Configure Linux server and deploy static site remotely

Issue Summary

While connecting to the EC2 instance using SSH, the connection failed.

Command Used
ssh -i "/c/Users/interndig/Downloads/nginx-deploy-demo.pem" ubuntu@3.111.30.212
Error
ssh: connect to host 3.111.30.212 port 22: Connection timed out
Impact
Unable to connect to EC2 remotely
Deployment process stopped
Static site deployment using rsync could not proceed
Investigation and Troubleshooting
Step 1 — SSH Key Verification

Verified:

Correct .pem file selected
File path was valid
Correct username (ubuntu) used
Result

PASS — SSH key configuration was correct.

Step 2 — Security Group Verification

Verified EC2 Security Group.

Inbound Rules
Type	Protocol	Port	Source
SSH	TCP	22	0.0.0.0/0
Outbound Rules
Type	Destination
All Traffic	0.0.0.0/0
Result

PASS — Security Group allowed SSH traffic.

Step 3 — Route Table Verification

Verified VPC route configuration.

Destination	Target
10.0.0.0/16	Local
0.0.0.0/0	Internet Gateway
Result

PASS — Internet routing configured correctly.

Step 4 — Subnet Verification

Verified subnet settings.

Setting	Status
Auto Assign Public IPv4	Enabled
Result

PASS — Instance running inside public subnet.

Step 5 — Network ACL Verification

Verified network access rules.

Inbound
Rule	Port	Action
SSH	22	Allow
Outbound
Rule	Action
All Traffic	Allow
Result

PASS — NACL not blocking traffic.

Step 6 — SSH Service Verification

Command:

sudo systemctl status ssh

Output:

Active: active (running)
Result

PASS — SSH service running successfully.

Step 7 — Port Verification

Command:

sudo ss -tlnp | grep :22

Output:

LISTEN 0.0.0.0:22
Result

PASS — SSH daemon actively listening on port 22.

Step 8 — Firewall Verification

Command:

sudo ufw disable

Output:

Firewall stopped and disabled on system startup

SSH connection retried.

Result

SUCCESS — Connection established.

Root Cause

The issue was caused by the Ubuntu UFW (Uncomplicated Firewall) blocking incoming SSH traffic.

Although:

Security Group configuration was correct
Route Table configuration was correct
Network ACL allowed traffic
SSH service was active

the OS-level firewall prevented external SSH access.

Resolution
Temporary Fix
sudo ufw disable
Recommended Production Fix
sudo ufw allow 22
sudo ufw reload
Network Flow
Local Machine
      ↓
SSH Client
      ↓
Security Group
      ↓
Route Table
      ↓
Public Subnet
      ↓
Network ACL
      ↓
OS Firewall (UFW)
      ↓
SSH Service
      ↓
EC2 Instance
Key Learning

Recommended SSH troubleshooting order:

1. Public IP
2. Security Group
3. Route Table
4. Subnet
5. Network ACL
6. SSH Service
7. Port Listening
8. Firewall
9. Logs
Outcome

SSH connectivity was successfully restored and the EC2 instance became accessible for deployment and server configuration.


Project: Linux Server Setup + Nginx + rsync Deployment
