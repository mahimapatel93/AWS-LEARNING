# AWS Networking — Fresher DevOps Interview & Practical Preparation

AWS-focused networking guide for DevOps/AWS freshers.

## Goal

Understand AWS networking well enough to:
- Explain concepts confidently.
- Build networking practically.
- Understand traffic flow.
- Troubleshoot common connectivity issues.
- Answer fresher-level interview questions.

---

# Roadmap

```text
Region + Availability Zone
        ↓
VPC + CIDR
        ↓
Subnets
        ↓
Public / Private Subnet
        ↓
Route Tables
        ↓
Internet Gateway
        ↓
NAT Gateway
        ↓
Public / Private IP
        ↓
Security Group
        ↓
NACL
        ↓
VPC Peering
        ↓
VPC Endpoints
        ↓
Route 53
        ↓
ALB / NLB
        ↓
Troubleshooting
```

---

# 1. AWS Region

An AWS Region is a geographic area containing multiple isolated Availability Zones.

### Interview Q: What is an AWS Region?

**Answer:**

> An AWS Region is a geographic area containing multiple isolated Availability Zones where AWS provides cloud infrastructure and services.

---

# 2. Availability Zone

An Availability Zone (AZ) is an isolated location within an AWS Region.

```text
Region
├── AZ-A
├── AZ-B
└── AZ-C
```

### Interview Q: Why use multiple AZs?

**Answer:**

> We use multiple Availability Zones to improve availability and reduce the impact of failure in a single Availability Zone.

---

# 3. VPC

VPC = Virtual Private Cloud.

A VPC is a logically isolated virtual network in AWS.

Example:

```text
VPC
10.0.0.0/16
├── Public Subnet
└── Private Subnet
```

### Interview Q: What is a VPC?

**Answer:**

> A VPC is a logically isolated virtual network in AWS where we launch resources such as EC2. We define its IP range using CIDR and control networking using subnets, route tables, gateways and security controls.

---

# 4. CIDR

CIDR = Classless Inter-Domain Routing.

Example:

```text
10.0.0.0/16
```

The `/16` is the prefix length.

Common IPv4 sizes:

```text
/24 → 256 total addresses
/25 → 128
/26 → 64
/27 → 32
/28 → 16
```

Formula:

```text
2^(32 - prefix)
```

AWS reserves 5 IPv4 addresses in each subnet, so usable addresses are fewer than the total.

### Interview Q: What is CIDR?

**Answer:**

> CIDR defines an IP address range and network size using a prefix length such as `/16` or `/24`.

---

# 5. Public and Private IP

### Private IP

Used inside private networks.

Common private IPv4 ranges:

```text
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
```

### Public IP

An internet-routable IPv4 address.

### Interview Q: Can a private IP be directly reached from the internet?

**Answer:**

> No. Private IPv4 addresses are not directly routable over the public internet.

---

# 6. Elastic IP

An Elastic IP is a static public IPv4 address that can be associated with an AWS resource such as an EC2 instance or network interface.

### Interview Q: Why use Elastic IP?

**Answer:**

> When we need a stable public IPv4 address that should remain the same for the workload.

Elastic IP vs Public IP — Interview Answer

“Both are public IP addresses, but a normal public IP can change when an EC2 instance is stopped and started, while an Elastic IP is a static public IP that remains the same.”

Public IP	                   Elastic IP
Automatically assigned by AWS	  Allocated by us
Can change after stop/start       Remains fixed
Temporary	                  Static
No need to reserve	          We explicitly allocate it
Used for Internet access 	  Used when a fixed public IP is required

Easy example:

Public IP → 13.XXX.XXX.10 → stop/start → may change

Elastic IP → 13.XXX.XXX.20 → stop/start → stays the same

# “Normal public IP can change when an EC2 instance is stopped and started. So, if an application or DNS needs a fixed IP address, we use an Elastic IP.”
Example:

EC2 → Public IP 13.10.20.30
Stop → Start → Public IP change ho sakta hai ❌

Elastic IP → 13.10.20.40
Stop → Start → same IP ✅

So, Public IP = dynamic, Elastic IP = static.

---

# 7. Subnet

A subnet is a smaller IP network inside a VPC.

```text
VPC 10.0.0.0/16
├── Public Subnet  10.0.1.0/24
└── Private Subnet 10.0.2.0/24
```

### Why use subnets?

- Organize resources.
- Separate public/private workloads.
- Apply different routing.
- Improve architecture and security.

### Interview Q: What is a subnet?

**Answer:**

> A subnet is a smaller network inside a VPC used to organize resources and control routing.

---

# 8. Public vs Private Subnet

### Public Subnet

A subnet is considered public when its route table has a route to an Internet Gateway.

```text
EC2
 ↓
Public Subnet
 ↓
Route Table
 ↓
Internet Gateway
 ↓
Internet
```

### Private Subnet

A private subnet has no direct route to an Internet Gateway.

For outbound internet access:

```text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Internet Gateway
 ↓
Internet
```

A public subnet is a subnet that has a route to the Internet through an Internet Gateway. A private subnet does not have a direct route to the Internet Gateway. Public subnets are commonly used for load balancers or web servers, while private subnets are used for resources like application servers and databases.

### Easy Way to Remember

**Public Subnet:**
Internet → Internet Gateway → Public Subnet

**Private Subnet:**
Internet → ❌ Direct Access

For a private subnet to access the Internet, it can use a NAT Gateway, but the Internet cannot directly initiate connections to the private resources.

### Interview Q: What makes a subnet public?

**Answer:**

> A subnet is considered public when its associated route table contains a route such as `0.0.0.0/0 → Internet Gateway`. Resources also need appropriate public addressing and security rules.

---

# 9. Route Table

We use a route table to control and decide where network traffic should go. It tells AWS the next destination for the traffic.

Public example:

```text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         IGW
```

Private example:

```text
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         NAT Gateway
```

### Interview Q: What is `0.0.0.0/0`?

**Answer:**

> It represents all IPv4 destinations and is commonly used as a default route.

### Interview Q: What is the local route?

**Answer:**

> The local route allows communication between destinations inside the VPC CIDR, subject to applicable security controls.

---

# 10. Internet Gateway

An Internet Gateway (IGW) provides a path between a VPC and the internet.

```text
Internet
   ↓
Internet Gateway
   ↓
VPC
   ↓
Public Subnet
   ↓
EC2
```

### Important

Attaching an IGW alone does not make EC2 public.

You need:

```text
IGW
+
Correct Route Table
+
Public IP / public addressing
+
Security Rules
```

### Interview Q: What is an Internet Gateway?

**Answer:**

> An Internet Gateway provides a path for internet communication for resources in a VPC when the required routing and addressing are configured.

---

# 11. NAT Gateway

NAT Gateway allows resources in private subnets to initiate outbound internet connections.
NAT Gateway is an AWS service that allows resources in a private subnet to access the Internet for outbound traffic without assigning them public IP addresses. It is created in a public subnet and uses an Elastic IP

```text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
Internet Gateway
 ↓
Internet
```

### Interview Q: Why use NAT Gateway?

**Answer:**

> NAT Gateway allows private subnet resources to make outbound connections to the internet without making those resources directly reachable from the internet.

### Interview Q: Why is NAT Gateway placed in a public subnet?

**Answer:**

> A NAT Gateway needs connectivity to the Internet Gateway for internet access, so it is deployed in a public subnet.

---

# 12. Security Group

A Security Group is a virtual firewall for AWS resources like EC2. It controls inbound and outbound network traffic using rules based on protocols, ports, and IP addresses

Example:

```text
Inbound
SSH    TCP 22   My IP
HTTP   TCP 80   0.0.0.0/0
HTTPS  TCP 443  0.0.0.0/0
```

Properties:

- Stateful
- Allow rules
- Inbound and outbound rules
- Associated with network interfaces/resources

### Interview Q: What is a Security Group?

**Answer:**

> A Security Group is a stateful virtual firewall in AWS that controls inbound and outbound traffic for resources through their network interfaces.

---

# 13. NACL

NACL = Network Access Control List.

It works at the subnet level.

Properties:

- Stateless
- Allow and deny rules
- Inbound and outbound rules
- Rules evaluated by rule number

### Interview Q: What is NACL?

**Answer:**

> A NACL is a stateless network traffic filter applied at the subnet level. It supports both allow and deny rules.

---

# 14. Security Group vs NACL

| Security Group | NACL |
|---|---|
| Stateful | Stateless |
| Resource / ENI level | Subnet level |
| Allow rules | Allow + Deny |
| Return traffic automatically handled | Return traffic must be explicitly allowed |
| No rule ordering like NACL | Rules evaluated by rule number |

### Interview Answer

> Security Groups are stateful firewalls associated with network interfaces, while NACLs are stateless filters applied at the subnet level.

---

# 15. VPC Peering

VPC Peering creates private connectivity between two VPCs.

```text
VPC-A
10.0.0.0/16
   │
   │ Peering
   │
VPC-B
10.1.0.0/16
```

Appropriate routes and security rules are required.

### Interview Q: Is VPC Peering transitive?

**Answer:**

> No. VPC Peering is non-transitive.

```text
A ↔ B
B ↔ C
```

does not automatically mean:

```text
A ↔ C
```

---

# 16. VPC Endpoint

A VPC Endpoint provides private connectivity from a VPC to supported AWS services.

```text
Private EC2
    ↓
VPC Endpoint
    ↓
AWS Service
```

Basic types:

```text
Gateway Endpoint
→ S3
→ DynamoDB

Interface Endpoint
→ Many AWS services
→ Private IP / ENI
```

### Interview Q: What is a VPC Endpoint?

**Answer:**

> A VPC Endpoint allows resources in a VPC to access supported AWS services privately without requiring a public internet path.

---

# 17. Transit Gateway

Transit Gateway is a central networking hub.
AWS Transit Gateway is a central networking service used to connect multiple VPCs and on-premises networks through a single central hub. It simplifies network connectivity and routing between different networks.

```text
VPC-A ──┐
VPC-B ──┤
VPC-C ──┼── Transit Gateway
On-Prem ─┘
```

### Interview Q: What is Transit Gateway?

**Answer:**

> Transit Gateway is a central hub used to connect multiple VPCs and other networks, making large AWS network architectures easier to manage.

---

# 18. AWS VPN

AWS VPN is a secure and encrypted connection between an on-premises network and AWS VPC over the Internet

```text
On-Premises
     │
     │ VPN
     │
    AWS
     │
    VPC
```

### Interview Q: What is Site-to-Site VPN?

**Answer:**

> It provides encrypted network connectivity between an on-premises network and AWS over the internet.

---

# 19. Direct Connect

Direct Connect provides a dedicated network connection between an organization and AWS.

Basic comparison:

```text
VPN
→ Internet-based encrypted connection

Direct Connect
→ Dedicated network connection to AWS
```

AWS Direct Connect is a dedicated private network connection between an on-premises network and AWS.

### Easy Difference

**VPN** → Internet ke through secure connection 🔐

**Direct Connect** → Dedicated private connection 📡

### Interview Mein Yaad Rakho

👉 **VPN** = Secure tunnel over Internet  
👉 **Direct Connect** = Dedicated connection to AWS

For fresher interviews, basic understanding is enough.
---

# 20. Route 53

Amazon Route 53 is AWS's DNS service.

It supports:

- DNS resolution
- Hosted zones
- DNS records
- Health checks
- Routing policies
- Domain registration

### Common records

| Record | Purpose |
|---|---|
| A | Hostname → IPv4 |
| AAAA | Hostname → IPv6 |
| CNAME | Hostname → Hostname |
| MX | Mail servers |
| TXT | Text/verification |

### Interview Q: What is Route 53?

**Answer:**

> Route 53 is AWS's DNS service used for DNS resolution, hosted zones, health checks and routing policies.

---

# 21. Load Balancer

A Load Balancer distributes incoming network traffic across multiple servers or EC2 instances to improve performance, availability, and reliability.

```text
Users
  ↓
Load Balancer
  ├── EC2-1
  ├── EC2-2
  └── EC2-3
```

Benefits:

- High availability
- Traffic distribution
- Health checks
- Reduced single point of failure

---

# 22. ALB

ALB = Application Load Balancer.

It operates at Layer 7 and is mainly used for HTTP/HTTPS applications.
ALB is an AWS load balancer that distributes HTTP and HTTPS traffic across multiple targets, such as EC2 instances, containers, or IP addresses

Supports:

- Host-based routing
- Path-based routing
- Listeners
- Target Groups
- Health Checks

Example:

```text
example.com/api
      ↓
     ALB
      ↓
API Target Group

example.com/web
      ↓
     ALB
      ↓
Web Target Group
```

### Interview Q: What is ALB?

**Answer:**

> ALB is a Layer 7 load balancer designed mainly for HTTP and HTTPS applications. It supports features such as host-based and path-based routing.

---

## Path-Based Routing

### What is path-based routing?

> “Path-based routing means ALB routes the request based on the URL path. For example, if the request is `/api`, ALB sends it to the backend target group, and if it is `/admin`, it sends it to the admin target group.”

### Example:

```text
example.com/api    → Backend
example.com/admin  → Admin
```

---

## Host-Based Routing

### What is host-based routing?

> “Host-based routing means ALB routes the request based on the hostname or domain name. For example, `api.example.com` can go to the backend target group and `admin.example.com` can go to the admin target group.”

### Example:

```text
api.example.com    → Backend
admin.example.com  → Admin
```

---

## ⭐ Difference — Interview Mein

> “Path-based routing checks the URL path, while host-based routing checks the hostname or domain name.”

# 23. NLB

NLB = Network Load Balancer.

It operates at Layer 4.

Common protocols:

```text
TCP
UDP
TLS
```

Remember:

```text
ALB → Layer 7 → HTTP/HTTPS
NLB → Layer 4 → TCP/UDP/TLS
```

### Interview Q: ALB vs NLB?

**Answer:**

> ALB works at Layer 7 and is mainly designed for HTTP/HTTPS application traffic, while NLB works at Layer 4 and is designed for high-performance TCP, UDP and TLS traffic.

---

# 24. Target Group

A Target Group is a collection of resources, such as EC2 instances, to which a Load Balancer sends traffic.
ALB first receives the request, then forwards it to the Target Group, and the Target Group sends it to healthy EC2 instances.

```text
ALB
 ↓
Target Group
 ├── EC2-1
 ├── EC2-2
 └── EC2-3
```

### Interview Q: What is a Target Group?

**Answer:**

> A Target Group is a collection of backend targets, such as EC2 instances, that receive traffic from a load balancer.
> Target Group = Group of servers (EC2s) behind the Load Balancer.

---

# 25. Health Check

A load balancer uses health checks to determine whether targets can receive traffic.

```text
ALB
 ↓
GET /health
 ↓
EC2
 ↓
200 OK
```

If a target fails health checks, the load balancer stops sending traffic to that target.

### Interview Q: What is a health check?

**Answer:**

> A health check is used by the Load Balancer to check whether the target server is healthy and ready to receive traffic
> /health

If the server gives the expected response, ALB considers it healthy and sends traffic to it.
Health check verifies whether a target is healthy before sending traffic to it

---

# 🧪 Practical Labs

## Lab 1 — VPC + CIDR

Create:

```text
VPC
CIDR: 10.0.0.0/16
```

Understand:

- VPC CIDR
- Custom VPC
- Network range

### Interview Q: Why create a custom VPC?

**Answer:**

> A custom VPC gives us control over IP addressing, subnet structure, routing and network security.

---

## Lab 2 — Public Subnet

Create:

```text
VPC
10.0.0.0/16

Public Subnet
10.0.1.0/24
```

Then create:

```text
Internet Gateway
Route Table
```

Route:

```text
0.0.0.0/0 → IGW
```

Associate the route table with the subnet.

---

## Lab 3 — Public EC2

Architecture:

```text
Internet
   ↓
IGW
   ↓
Public Subnet
   ↓
EC2
```

Security Group:

```text
SSH 22 → Your IP
HTTP 80 → 0.0.0.0/0
```

SSH into the instance.

---

## Lab 4 — Break SSH

1. Remove TCP 22 from the Security Group.
2. Try SSH.
3. Observe the failure.
4. Restore the rule.
5. Connect again.

### Lesson

```text
Client
 ↓
Internet
 ↓
IGW
 ↓
Route
 ↓
NACL
 ↓
Security Group
 ↓
Port 22
 ↓
SSH service
```

---

## Lab 5 — Private Subnet

Create:

```text
Private Subnet
10.0.2.0/24
```

Do not give it a direct route to the Internet Gateway.

---

## Lab 6 — NAT Gateway

Create NAT Gateway in the public subnet.

Private route:

```text
0.0.0.0/0 → NAT Gateway
```

Architecture:

```text
Private EC2
 ↓
Private Route Table
 ↓
NAT Gateway
 ↓
IGW
 ↓
Internet
```

Test outbound connectivity.

---

## Lab 7 — Security Group Troubleshooting

Practice:

```text
Allow 80
Block 80
Allow 22
Block 22
```

Understand:

```text
Timeout
vs
Connection Refused
```

---

## Lab 8 — NACL

Practice:

- Allow traffic
- Deny traffic
- Inbound rules
- Outbound rules
- Stateless behavior

---

## Lab 9 — Application Load Balancer

Architecture:

```text
Internet
    ↓
   ALB
    ↓
Target Group
    ↓
EC2-1 + EC2-2
```

Practice:

- Listener
- Target Group
- Health Check
- Security Groups
- Traffic distribution

---

## Lab 10 — Route 53

Practice:

```text
Domain
  ↓
Route 53
  ↓
ALB
  ↓
EC2
```

Test DNS resolution.

---

## Lab 11 — VPC Peering

Create:

```text
VPC-A
10.0.0.0/16

VPC-B
10.1.0.0/16
```

Create peering and configure routes.

Test private connectivity.

---

## Lab 12 — VPC Endpoint

Create a private subnet and an appropriate endpoint.

Understand:

```text
Private EC2
    ↓
VPC Endpoint
    ↓
AWS Service
```

---

# 🐧 Linux Networking Commands

```bash
# IP address
ip addr

# Routing table
ip route

# Listening ports
ss -tulnp

# Connectivity
ping 8.8.8.8

# DNS
nslookup google.com
dig google.com

# HTTP
curl -I https://example.com

# DNS configuration
cat /etc/resolv.conf

# Trace path
traceroute google.com

# Packet capture
sudo tcpdump -i any
```

---

# 🔧 AWS Networking Troubleshooting

Use this order:

```text
DNS
 ↓
IP / Public IP
 ↓
Route Table
 ↓
IGW / NAT
 ↓
NACL
 ↓
Security Group
 ↓
Port
 ↓
Service
 ↓
Application
```

---

# 🎤 Fresher Interview Questions & Answers

## Networking Basics

### 1. What is networking?

> Networking is communication between connected devices or systems so they can exchange data.

### 2. What is an IP address?

> An IP address identifies a network interface in an IP network and allows traffic to be addressed and routed.

### 3. What is a port?

> A port is a logical endpoint used to identify a service or application on a host.

### 4. What is TCP?

> TCP is a connection-oriented transport protocol that provides reliable, ordered delivery of data.

### 5. What is UDP?

> UDP is a connectionless transport protocol with lower overhead than TCP and without TCP's delivery guarantees.

### 6. TCP vs UDP?

> TCP is connection-oriented and provides reliable ordered delivery, while UDP is connectionless and has lower overhead but does not provide TCP-style delivery guarantees.

---

## AWS Networking

### 7. What is VPC?

> A VPC is a logically isolated virtual network in AWS where we launch resources and control IP addressing, routing and network security.

### 8. What is a subnet?

> A subnet is a smaller IP network inside a VPC used to organize resources and control routing.

### 9. Public vs private subnet?

> A public subnet has a route to an Internet Gateway, while a private subnet does not have a direct route to an Internet Gateway.

### 10. What is an Internet Gateway?

> An Internet Gateway provides a path between a VPC and the internet when routing, addressing and security are correctly configured.

### 11. What is NAT Gateway?

> NAT Gateway allows private subnet resources to initiate outbound internet connections without exposing them directly to the internet.

### 12. What is a route table?

> A route table contains routing rules that determine where traffic should be sent.

### 13. What is a Security Group?

> A Security Group is a stateful virtual firewall associated with network interfaces that controls inbound and outbound traffic.

### 14. What is NACL?

> A NACL is a stateless subnet-level traffic filter that supports allow and deny rules.

### 15. Security Group vs NACL?

> Security Group is stateful and works at the resource/network-interface level. NACL is stateless and works at the subnet level.

### 16. What is Route 53?

> Route 53 is AWS's DNS service used for DNS resolution, hosted zones, health checks and routing policies.

### 17. What is ALB?

> ALB is a Layer 7 load balancer designed mainly for HTTP and HTTPS applications.

### 18. What is NLB?

> NLB is a Layer 4 load balancer designed for high-performance TCP, UDP and TLS traffic.

### 19. What is VPC Peering?

> VPC Peering provides private connectivity between two VPCs using private IP addressing, with appropriate routing and security rules.

### 20. What is VPC Endpoint?

> VPC Endpoint provides private connectivity from a VPC to supported AWS services without requiring a public internet path.

---

# 🚨 Scenario Questions

## 21. EC2 is running but SSH is not working. What will you check?

> I would first confirm the instance is running and has the required public connectivity. Then I would check the Security Group for TCP port 22, subnet route table, Internet Gateway, NACL, SSH service, username and key.

Checklist:

```text
EC2 running?
 ↓
Public IP?
 ↓
Route?
 ↓
IGW?
 ↓
NACL?
 ↓
Security Group port 22?
 ↓
SSH service?
 ↓
Correct key/user?
```

---

## 22. EC2 cannot access the internet. What will you check?

### Public EC2

```text
Public IP
Route Table
0.0.0.0/0 → IGW
Internet Gateway
Security Group
NACL
```

### Private EC2

```text
NAT Gateway
NAT Gateway in public subnet
Public subnet route → IGW
Private route → NAT Gateway
Security Group
NACL
```

---

## 23. Website is not opening. How do you troubleshoot?

> I would troubleshoot layer by layer, starting from DNS and then checking the network path, security controls, load balancer and application.

```text
DNS
 ↓
ALB / Public IP
 ↓
Route
 ↓
NACL
 ↓
Security Group
 ↓
Listener
 ↓
Target Group
 ↓
Health Check
 ↓
Application Port
 ↓
Application
```

---

## 24. EC2 can reach an IP but cannot reach a domain. What could be wrong?

> DNS resolution may be failing. I would check DNS configuration and use `nslookup` or `dig`.

```bash
nslookup google.com
dig google.com
cat /etc/resolv.conf
```

---

## 25. What is connection timeout?

> A connection timeout means the connection attempt did not successfully complete within the expected time. I would investigate routing, Security Groups, NACLs, firewalls and the destination service.

---

## 26. What is connection refused?

> Connection refused generally means the destination was reached but the target port is not accepting the connection or the connection was actively rejected.

Check:

```bash
ss -tulnp
```

---

# ⭐ Important DevOps Ports

| Port | Service |
|---:|---|
| 22 | SSH |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | Common alternate HTTP |
| 27017 | MongoDB |

---

# 🏗️ Architecture You Must Understand

```text
                         INTERNET
                             │
                             ▼
                     Internet Gateway
                             │
              ┌──────────────┴──────────────┐
              │            VPC              │
              │         10.0.0.0/16         │
              │                             │
              │   PUBLIC SUBNET             │
              │   10.0.1.0/24               │
              │        │                    │
              │       ALB                   │
              │        │                    │
              │       EC2                   │
              │                             │
              │   PRIVATE SUBNET            │
              │   10.0.2.0/24               │
              │        │                    │
              │       EC2                   │
              │        │                    │
              │   NAT Gateway               │
              │        │                    │
              └────────┼────────────────────┘
                       │
                       ▼
                    INTERNET
```

You should be able to explain:

- Why VPC?
- Why CIDR?
- Why subnets?
- Why public/private?
- Why route tables?
- Why IGW?
- Why NAT?
- Where does Security Group work?
- Where does NACL work?
- How does traffic flow?

---

# ⭐ Fresher Priority

## Must Know

```text
VPC
CIDR
Subnet
Public vs Private Subnet
Route Table
Internet Gateway
NAT Gateway
Security Group
NACL
Public/Private IP
Elastic IP
ALB
Target Group
Health Check
Route 53
```

## Should Know

```text
VPC Peering
VPC Endpoint
NLB
Transit Gateway
VPN
Multi-AZ
```

## Basic Awareness

```text
Direct Connect
IPv6
BGP
Advanced routing
```

---

# 📅 14-Day Plan

| Day | Topics |
|---|---|
| 1 | Region, AZ, VPC, CIDR |
| 2 | Subnets, Public vs Private |
| 3 | Route Tables, Local Routes |
| 4 | Internet Gateway |
| 5 | NAT Gateway |
| 6 | Public/Private IP, Elastic IP |
| 7 | Security Groups |
| 8 | NACL |
| 9 | VPC Peering |
| 10 | VPC Endpoints |
| 11 | Route 53 |
| 12 | ALB, NLB, Target Groups |
| 13 | Troubleshooting |
| 14 | Mock Interview |

---

# 🧠 Interview Answer Formula

For most AWS networking questions:

```text
1. Definition
2. Purpose
3. How it works
4. AWS example
5. Important/troubleshooting point
```

### Example: NAT Gateway

**Weak:**

> NAT Gateway provides internet access to private instances.

**Interview-ready:**

> NAT Gateway allows resources in private subnets to initiate outbound internet connections. It is deployed in a public subnet and uses the Internet Gateway for internet connectivity. The private subnet's route table sends internet-bound traffic to the NAT Gateway. It does not provide direct inbound internet access to private instances.

---

# 🏆 Final Rule

For every topic, understand:

```text
WHAT is it?
      ↓
WHY do we need it?
      ↓
WHERE is it used?
      ↓
HOW does traffic flow?
      ↓
WHAT can go wrong?
      ↓
HOW do I troubleshoot it?
```

If you can answer all six, you understand the concept.

---

# 🚀 Practical Order

```text
1. VPC + CIDR
       ↓
2. Public Subnet
       ↓
3. Route Table
       ↓
4. Internet Gateway
       ↓
5. Public EC2
       ↓
6. Security Group
       ↓
7. Break SSH + Troubleshoot
       ↓
8. Private Subnet
       ↓
9. NAT Gateway
       ↓
10. Private EC2
       ↓
11. ALB + Target Group
       ↓
12. Route 53
       ↓
13. NACL
       ↓
14. VPC Peering
       ↓
15. VPC Endpoint
       ↓
16. Final Troubleshooting
       ↓
17. Mock Interview
```

**Goal:** You should be able to look at an AWS networking architecture and explain the complete traffic flow in your own words.
