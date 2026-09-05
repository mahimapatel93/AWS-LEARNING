Practical Lab: Fix Changing Public IP Using WireGuard VPN

🎯 What You Will Build

This lab solves a common problem: your Jio/mobile hotspot/Wi-Fi public IP changes, which can break SSH access when an AWS Security Group allows only your current IP.

We will create a self-hosted WireGuard VPN server on AWS EC2 with an Elastic IP.

Final Architecture

Windows Laptop
     │
 Jio / Wi-Fi
     │
 WireGuard VPN
     │
     ▼
AWS EC2 — WireGuard Server
10.8.0.1 + Elastic IP
     │
     │ NAT / forwarding
     ▼
Target EC2
SSH allowed from VPN Elastic IP /32

1. Why Are We Doing This?

Suppose your Jio connection gives you:

Public IP = 49.x.x.x

You allow this IP in the target EC2 Security Group:

SSH | TCP | 22 | 49.x.x.x/32

Later Jio changes your public IP:

Old IP → 49.x.x.x
New IP → 106.x.x.x

The Security Group still allows the old IP, so SSH can fail.

Our solution

Instead of allowing the changing Jio IP, allow the fixed Elastic IP of the VPN server.

Laptop → WireGuard → AWS VPN Server → Target EC2
                         │
                    Fixed Elastic IP

2. Important Terminology

Term

Meaning

VPN

Creates an encrypted network connection

WireGuard

VPN software/protocol used in this lab

VPN Server

AWS EC2 instance running WireGuard

VPN Client

WireGuard running on Windows

Elastic IP

Fixed public IPv4 address in AWS

Public IP

Internet-facing IP

VPN IP

Private IP used inside the WireGuard tunnel

Security Group

AWS virtual firewall

IP Forwarding

Allows the VPN server to forward client traffic

NAT/MASQUERADE

Makes forwarded traffic appear to come from the VPN server

3. IP Addresses

We use three different types of addresses.

Laptop public IP

Provided by Jio/Wi-Fi/ISP:

49.x.x.x

It can change.

VPN server Elastic IP

Example:

3.x.x.x

This is the fixed public IP of the VPN server.

WireGuard VPN network

10.8.0.0/24

VPN Server  = 10.8.0.1
Windows     = 10.8.0.2

4. Create the VPN Server EC2

Recommended settings:

Name: vpn-server
AMI: Ubuntu 24.04 LTS
Instance type: t3.micro
VPC: Default VPC
Subnet: Public subnet
Auto-assign public IPv4: Enabled

Create/use an SSH key pair, for example:

key_vpn.pem

5. Configure the VPN Server Security Group

Inbound rules:

Type

Protocol

Port

Source

SSH

TCP

22

My IP

Custom UDP

UDP

51820

0.0.0.0/0

Port 22 is for SSH.

Port 51820 is for WireGuard.

6. Allocate an Elastic IP

AWS Console:

EC2
→ Elastic IPs
→ Allocate Elastic IP address
→ Associate
→ Select vpn-server

Record the Elastic IP:

YOUR_VPN_ELASTIC_IP

7. SSH Into the VPN Server

From Windows CMD/PowerShell:

cd C:\Users\patel\Downloads

ssh -i "key_vpn.pem" ubuntu@YOUR_VPN_ELASTIC_IP

8. Install WireGuard

Inside Ubuntu:

sudo apt update
sudo apt install wireguard -y

Verify:

wg --version

9. Generate Server Keys

sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard

sudo sh -c 'umask 077; wg genkey > /etc/wireguard/server_private.key'

sudo sh -c 'wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key'

sudo ls -l /etc/wireguard/

View keys when needed:

sudo cat /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_public.key

⚠️ Never share the private key.

10. Enable IP Forwarding

Temporarily:

sudo sysctl -w net.ipv4.ip_forward=1

Verify:

sysctl net.ipv4.ip_forward

Expected:

net.ipv4.ip_forward = 1

Make it permanent:

sudo nano /etc/sysctl.conf

Add/uncomment:

net.ipv4.ip_forward=1

Save with:

Ctrl + O
Enter
Ctrl + X

Apply:

sudo sysctl -p

11. Find the AWS Network Interface

Run:

ip route

Example:

default via 172.31.0.1 dev ens5

Here the interface is:

ens5

Your interface may be different.

12. Create WireGuard Server Configuration

sudo nano /etc/wireguard/wg0.conf

Use:

[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = YOUR_SERVER_PRIVATE_KEY

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o YOUR_INTERFACE -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o YOUR_INTERFACE -j MASQUERADE

Replace:

YOUR_SERVER_PRIVATE_KEY

with the server private key.

Replace:

YOUR_INTERFACE

with the interface from:

ip route

Example:

ens5

Protect the configuration:

sudo chmod 600 /etc/wireguard/wg0.conf

13. Disable EC2 Source/Destination Check

AWS Console:

EC2
→ Instances
→ Select vpn-server
→ Actions
→ Networking
→ Change source/destination check
→ Stop

Why?

The VPN server is forwarding traffic for the Windows client, so AWS must allow it to act as a router.

14. Start WireGuard

sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

Check:

sudo systemctl status wg-quick@wg0

And:

sudo wg

15. Install WireGuard on Windows

Install the WireGuard application on Windows.

Then:

WireGuard
→ Add Tunnel
→ Add empty tunnel

WireGuard generates a client key pair.

You will have:

PrivateKey
PublicKey

⚠️ Never share the Windows private key.

16. Configure the Windows Client

Use:

[Interface]
PrivateKey = WINDOWS_PRIVATE_KEY
Address = 10.8.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = YOUR_VPN_ELASTIC_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25

Replace the placeholders with your real values.

AllowedIPs = 0.0.0.0/0 means IPv4 internet traffic is routed through the VPN.

17. Add Windows as a Peer on the Server

On Ubuntu:

sudo nano /etc/wireguard/wg0.conf

Add:

[Peer]
PublicKey = WINDOWS_PUBLIC_KEY
AllowedIPs = 10.8.0.2/32

Restart:

sudo systemctl restart wg-quick@wg0

Check:

sudo wg

18. Activate the Windows Tunnel

Open WireGuard.

Select your tunnel.

Click:

Activate

19. Verify the VPN

Before activating WireGuard:

curl -4 ifconfig.me

This should show your Jio/Wi-Fi public IP.

After activating WireGuard:

curl -4 ifconfig.me

It should now show the VPN server's Elastic IP.

Main proof

Before VPN:
Laptop → Jio public IP

After VPN:
Laptop → WireGuard → AWS VPN Server → Elastic IP

20. Configure the Target EC2 Security Group

On the target EC2 Security Group, add:

Type: SSH
Protocol: TCP
Port: 22
Source: YOUR_VPN_ELASTIC_IP/32

Example:

3.x.x.x/32

Do not normally use:

0.0.0.0/0

for SSH.

21. SSH to the Target EC2

With WireGuard active:

ssh -i "your-target-key.pem" ubuntu@TARGET_EC2_PUBLIC_IP

The target should receive the connection through the VPN path.

22. Prove That the Jio IP Can Change

Disconnect and reconnect the Jio hotspot/Wi-Fi.

Check:

curl -4 ifconfig.me

Your ISP public IP may change.

With WireGuard active, check again:

curl -4 ifconfig.me

The internet-facing IP should remain the VPN server's Elastic IP.

23. Troubleshooting

Check WireGuard

sudo wg

Check service

sudo systemctl status wg-quick@wg0

Restart

sudo systemctl restart wg-quick@wg0

Logs

sudo journalctl -u wg-quick@wg0

IP forwarding

sysctl net.ipv4.ip_forward

Expected:

1

Network interface

ip route

Listening port

sudo ss -lunp | grep 51820

NAT rules

sudo iptables -t nat -L -n -v

24. SSH Connection Timeout

If Windows shows:

ssh: connect to host ... port 22: Connection timed out

Check:

EC2 is Running.

Status checks are 2/2 passed.

You are using the current public IP/Elastic IP.

Security Group allows:

TCP 22 → your current IP

If using Jio/mobile hotspot, your public IP may have changed.

For troubleshooting, update the SSH Security Group rule to:

SSH → My IP

25. Useful SSH Server Logs

On Ubuntu:

sudo journalctl -u ssh --since "15 minutes ago"

And:

last

These help verify SSH activity.

26. Common Mistakes

Mistake 1: Allowing the Jio IP on the target

Not the main goal:

Target SG → Jio IP

Use:

Target SG → VPN Elastic IP /32

Mistake 2: No Elastic IP

The VPN server should have a fixed Elastic IP.

Mistake 3: UDP 51820 not allowed

The VPN server Security Group must allow:

UDP 51820

Mistake 4: IP forwarding disabled

Check:

sysctl net.ipv4.ip_forward

Mistake 5: Source/destination check still enabled

Disable it because the VPN server forwards traffic.

Mistake 6: Wrong interface

Do not assume eth0.

Find it:

ip route

Mistake 7: Sharing private keys

Never share:

server_private.key
Windows PrivateKey

27. What We Actually Built

We built a self-hosted WireGuard VPN.

Components:

AWS EC2
+
Elastic IP
+
WireGuard
+
IP Forwarding
+
NAT
+
Windows WireGuard Client

The Windows laptop connects to the AWS VPN server.

The VPN server forwards traffic to the internet.

The outside world sees the VPN server's fixed Elastic IP.

28. This Is NOT AWS Site-to-Site VPN

Do not confuse these two.

This lab

Windows Laptop
      ↓
WireGuard
      ↓
AWS EC2 VPN Server
      ↓
Internet / Target EC2

AWS Site-to-Site VPN

On-Premises Network
      ↓
Customer Gateway
      ↓
IPsec VPN
      ↓
Virtual Private Gateway /
Transit Gateway
      ↓
AWS VPC

They solve different problems.

29. Interview Questions and Answers

Q: What problem did you solve?

Answer:

I solved the problem of changing public IP addresses affecting SSH access to an AWS EC2 instance. I created a WireGuard VPN server on AWS EC2 and assigned it an Elastic IP. My Windows laptop connects to the VPN, so the target EC2 can allow SSH from the VPN server's fixed IP instead of my changing ISP IP.

Q: Why WireGuard?

Answer:

WireGuard is a modern, lightweight VPN protocol. I used it to create an encrypted tunnel between my Windows laptop and the AWS VPN server.

Q: Why Elastic IP?

Answer:

Elastic IP provides a stable public IPv4 address for the VPN server. The target EC2 Security Group can allow that fixed address.

Q: Why enable IP forwarding?

Answer:

The VPN server acts as a router. It needs to forward traffic from the VPN client toward the internet and back.

Q: Why NAT/MASQUERADE?

Answer:

NAT changes the source of forwarded traffic so it can leave through the VPN server's network interface and appear to originate from the VPN server.

Q: Why disable source/destination checking?

Answer:

The VPN EC2 is forwarding traffic that belongs to the VPN client. Disabling source/destination checking allows the instance to act as a router.

Q: Which port does WireGuard use?

Answer:

In this lab, WireGuard uses UDP port 51820.

Q: What does /32 mean?

Answer:

/32 represents exactly one IPv4 address. So 3.x.x.x/32 allows only that specific IP.

30. Security Group vs WireGuard

They are different.

Security Group

AWS firewall
→ Controls which traffic can reach an EC2 instance.

WireGuard

VPN
→ Creates an encrypted tunnel between the client and VPN server.

Together:

WireGuard
    ↓
Secure tunnel

Security Group
    ↓
Controls AWS access

31. Lab Success Checklist

Created vpn-server EC2

Ubuntu installed

Security Group configured

UDP 51820 allowed

Elastic IP associated

SSH connection successful

WireGuard installed

Server keys generated

IP forwarding enabled

Network interface identified

wg0.conf created

Source/destination check disabled

WireGuard started

Windows WireGuard installed

Client keys generated

Windows peer added to server

Tunnel activated

Public IP changed to VPN Elastic IP

Target SG allows VPN Elastic IP /32

Target EC2 SSH tested

Jio IP change tested

32. Final Flow to Remember

                 Jio / Wi-Fi
                      │
                 Changing IP
                      │
                      ▼
              Windows Laptop
              WireGuard Client
                 10.8.0.2
                      │
                Encrypted VPN
                      │
                      ▼
              AWS EC2 VPN Server
                 10.8.0.1
                Elastic IP
                      │
                 NAT / Forward
                      │
                      ▼
                 Target EC2
             SG allows VPN IP
                    /32

One-line interview summary

I used a self-hosted WireGuard VPN on an AWS EC2 instance with an Elastic IP so that my laptop's changing ISP public IP does not affect SSH access to the target EC2 instance.
