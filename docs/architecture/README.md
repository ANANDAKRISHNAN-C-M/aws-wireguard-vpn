# AWS WireGuard VPN — Architecture

This directory contains the architecture diagram for the **Self-Hosted WireGuard VPN on AWS** project.

The VPN was deployed in the **AWS Asia Pacific (Sydney) Region (`ap-southeast-2`)** using an Ubuntu EC2 instance running WireGuard.

The architecture routes IPv4 internet traffic from a Windows client through an encrypted WireGuard tunnel to AWS, where the EC2 instance acts as the VPN gateway and internet egress point.

---

## Architecture Diagram

![Self-Hosted WireGuard VPN on AWS](aws-vpn-architecture.png)

---

## Architecture Overview

The infrastructure consists of the following major components:

- Windows PC running the WireGuard VPN client
- AWS Asia Pacific (Sydney) Region
- Custom Amazon VPC
- Public subnet
- Ubuntu EC2 instance running WireGuard
- AWS Security Group
- Public route table
- Internet Gateway
- Linux IPv4 forwarding
- `iptables` NAT / MASQUERADE

The overall traffic path is:

```text
Windows VPN Client
        │
        │ Encrypted WireGuard Tunnel
        │ UDP 51820
        ▼
Ubuntu EC2 WireGuard Server
        │
        │ WireGuard Decryption
        ▼
Linux IPv4 Forwarding
        │
        ▼
iptables NAT / MASQUERADE
        │
        ▼
Public Subnet Routing
        │
        ▼
Internet Gateway
        │
        ▼
Internet
```

---

# 1. AWS Region

The infrastructure was deployed in:

```text
AWS Asia Pacific (Sydney)
Region: ap-southeast-2
```

The Sydney Region was selected so that the VPN server would be hosted in Australia.

When the VPN is active, the client's IPv4 internet traffic exits through the EC2 instance deployed in this region.

Conceptually:

```text
VPN Client
     │
     │ Encrypted Tunnel
     ▼
AWS Sydney
     │
     ▼
Internet
```

The geographical location of the client is not important to the architecture. Any properly configured WireGuard client with network access and valid credentials could connect to the VPN server.

---

# 2. Amazon VPC

A custom Amazon Virtual Private Cloud was created with the CIDR:

```text
10.0.0.0/16
```

Architecture:

```text
AWS Asia Pacific (Sydney)
│
└── VPC
    10.0.0.0/16
```

The VPC provides the isolated AWS network in which the VPN infrastructure is deployed.

It controls the network environment for resources such as:

- Subnets
- EC2 instances
- Route tables
- Internet connectivity
- Security controls

The VPC network should not be confused with the WireGuard tunnel network.

---

# 3. Public Subnet

Inside the VPC, a public subnet was created:

```text
10.0.1.0/24
```

Architecture:

```text
VPC
10.0.0.0/16
│
└── Public Subnet
    10.0.1.0/24
        │
        └── Ubuntu EC2
```

The Ubuntu EC2 VPN server is deployed inside this subnet.

A subnet is not considered public simply because it is named "public."

Its public behavior comes from its routing configuration.

The associated route table contains a default route to an Internet Gateway:

```text
0.0.0.0/0 → Internet Gateway
```

Together with appropriate public addressing and security configuration, this gives the EC2 instance internet connectivity.

---

# 4. Two Separate Private Networks

An important part of the architecture is that it uses **two different private IP networks**.

## AWS Infrastructure Network

```text
VPC:
10.0.0.0/16

Public Subnet:
10.0.1.0/24
```

This network is managed by AWS VPC and provides connectivity for AWS resources.

## WireGuard Tunnel Network

```text
WireGuard Network:
10.10.0.0/24

VPN Server:
10.10.0.1

VPN Client:
10.10.0.2
```

This network exists logically inside the WireGuard VPN tunnel.

The relationship is:

```text
AWS NETWORK

VPC
10.0.0.0/16
│
└── Public Subnet
    10.0.1.0/24
        │
        └── EC2
            │
            │ runs WireGuard
            ▼

WIREGUARD NETWORK

10.10.0.0/24
│
├── Server: 10.10.0.1
│
└── Client: 10.10.0.2
```

These address spaces serve different purposes and should not be confused.

---

# 5. Ubuntu EC2 — WireGuard VPN Server

The central component of the architecture is an Ubuntu EC2 instance.

It performs several roles:

```text
Ubuntu EC2
│
├── WireGuard VPN Server
├── VPN Interface: wg0
├── VPN IP: 10.10.0.1
├── UDP 51820
├── IPv4 Forwarding
├── NAT / MASQUERADE
└── Internet Egress
```

The EC2 instance receives encrypted WireGuard traffic from the client.

WireGuard decrypts the traffic, after which Linux forwards the packets toward the internet.

The EC2 instance therefore acts as both:

```text
VPN Endpoint
+
Routing / Forwarding Device
```

---

# 6. WireGuard Tunnel

The Windows client uses the WireGuard VPN address:

```text
10.10.0.2
```

The EC2 WireGuard server uses:

```text
10.10.0.1
```

Communication between them occurs through an encrypted WireGuard tunnel over:

```text
UDP 51820
```

Architecture:

```text
Windows Client
WireGuard: 10.10.0.2
        │
        │
        │ Encrypted WireGuard Tunnel
        │ UDP 51820
        │
        ▼
AWS EC2
WireGuard: 10.10.0.1
```

The tunnel crosses the public internet, but the traffic inside the WireGuard tunnel is encrypted.

---

# 7. Security Group

The EC2 instance is protected by an AWS Security Group.

The required inbound access includes:

| Protocol | Port | Purpose |
|---|---:|---|
| TCP | `22` | SSH administration |
| UDP | `51820` | WireGuard VPN |

Conceptually:

```text
Incoming Traffic
       │
       ▼
Security Group
       │
       ├── TCP 22 allowed from authorized source
       │
       └── UDP 51820 allowed for WireGuard
                │
                ▼
             EC2
```

The Security Group should be understood as a **stateful virtual firewall associated with the EC2 network interface**.

It is not a separate physical firewall appliance through which packets travel.

In the architecture diagram, it is therefore shown logically associated with the EC2 instance.

---

# 8. IPv4 Forwarding

WireGuard delivers decrypted VPN traffic to the EC2 instance.

However, receiving the traffic is not enough.

The EC2 instance must forward that traffic toward its destination.

IPv4 forwarding was enabled on Ubuntu:

```text
net.ipv4.ip_forward=1
```

Without forwarding:

```text
VPN Client
     │
     ▼
WireGuard EC2
     │
     X
Internet
```

With forwarding:

```text
VPN Client
     │
     ▼
WireGuard EC2
     │
     ▼
Linux IP Forwarding
     │
     ▼
Internet Path
```

This allows the EC2 instance to function as a routing device.

---

# 9. NAT / iptables MASQUERADE

The WireGuard client uses:

```text
10.10.0.2
```

This is a private IP address and cannot be directly routed across the public internet.

The Ubuntu EC2 instance therefore performs Network Address Translation using Linux `iptables MASQUERADE`.

Conceptually:

```text
VPN Client
Source: 10.10.0.2
        │
        ▼
WireGuard Server
        │
        ▼
IPv4 Forwarding
        │
        ▼
iptables MASQUERADE
        │
        ▼
EC2 Internet-Facing Path
        │
        ▼
Internet
```

This allows the VPN client's traffic to use the EC2 instance as its internet egress point.

### NAT vs AWS NAT Gateway

This architecture does **not** use an AWS managed NAT Gateway.

```text
Used:

Linux NAT
iptables MASQUERADE
running on EC2


Not Used:

AWS NAT Gateway
```

NAT is a networking technique.

An AWS NAT Gateway is a specific managed AWS service.

In this project, NAT is performed locally by Linux on the VPN server.

---

# 10. Public Route Table

The public subnet is associated with a route table containing:

```text
Destination       Target

10.0.0.0/16       local

0.0.0.0/0         Internet Gateway
```

The local route handles traffic within the VPC CIDR.

The route:

```text
0.0.0.0/0 → Internet Gateway
```

acts as the default IPv4 route for destinations that do not match a more specific route.

Conceptually:

```text
EC2
 │
 ▼
Public Subnet
 │
 ▼
Route Table
 │
 ├── 10.0.0.0/16 → local
 │
 └── 0.0.0.0/0 → IGW
                   │
                   ▼
            Internet Gateway
```

---

# 11. Internet Gateway

An Internet Gateway is attached to the custom VPC.

It provides a path between appropriately configured VPC resources and the public internet.

Architecture:

```text
VPC
 │
 ├── Public Subnet
 │       │
 │       └── EC2
 │
 └── Internet Gateway
          │
          ▼
       Internet
```

The public route table directs:

```text
0.0.0.0/0
```

toward the Internet Gateway.

The Internet Gateway is not a NAT device in this architecture.

The Linux EC2 instance performs NAT for the WireGuard client.

---

# 12. EC2 Source/Destination Check

AWS EC2 normally performs source/destination checking.

For a normal EC2 workload, network packets are generally expected to have the instance as either their source or destination.

The VPN server behaves differently.

It forwards packets on behalf of another system:

```text
Windows VPN Client
        │
        ▼
EC2 VPN Server
        │
        ▼
Internet
```

The EC2 instance is therefore acting as an intermediary.

Source/destination checking was disabled to support this forwarding behavior.

This concept is also relevant to:

- VPN appliances
- NAT instances
- Virtual routers
- Network security appliances

---

# 13. Complete Packet Flow

When the VPN client accesses a website, the packet follows this logical path.

### Step 1 — Client generates traffic

```text
Windows Application
        │
        ▼
IPv4 Packet
```

### Step 2 — WireGuard routing

The client is configured with:

```ini
AllowedIPs = 0.0.0.0/0
```

Therefore, IPv4 internet traffic is directed into the WireGuard tunnel.

### Step 3 — Encryption

WireGuard encrypts the traffic.

```text
Normal Packet
      │
      ▼
WireGuard Encryption
      │
      ▼
Encrypted UDP Packet
```

### Step 4 — Public internet transport

The encrypted packet travels across the internet to the AWS EC2 WireGuard endpoint on:

```text
UDP 51820
```

### Step 5 — Security Group

The AWS Security Group permits the configured WireGuard traffic to reach the EC2 instance.

### Step 6 — WireGuard decryption

The EC2 WireGuard server decrypts the packet.

The VPN client's internal source address is:

```text
10.10.0.2
```

### Step 7 — Linux IP forwarding

Linux forwards the packet from the WireGuard interface toward the EC2 network interface.

### Step 8 — NAT / MASQUERADE

`iptables` performs NAT masquerading for the VPN client traffic.

### Step 9 — VPC routing

Internet-bound traffic follows the public subnet route:

```text
0.0.0.0/0 → Internet Gateway
```

### Step 10 — Internet Gateway

The traffic leaves the VPC through the Internet Gateway.

### Step 11 — Destination

The packet reaches the destination website or internet service.

### Step 12 — Return traffic

Responses return through AWS to the EC2 VPN server.

Linux networking associates the response with the NAT state and forwards it back toward the WireGuard client.

### Step 13 — WireGuard encryption

Return traffic is encrypted through WireGuard.

### Step 14 — Client receives response

The Windows client decrypts the response and delivers it to the application.

The complete logical flow is:

```text
Windows Application
        │
        ▼
WireGuard Client
10.10.0.2
        │
        │ Encrypted UDP 51820
        ▼
Public Internet
        │
        ▼
AWS Security Group
        │
        ▼
Ubuntu EC2
WireGuard 10.10.0.1
        │
        ▼
WireGuard Decryption
        │
        ▼
Linux IPv4 Forwarding
        │
        ▼
iptables NAT / MASQUERADE
        │
        ▼
Public Subnet Routing
        │
        ▼
0.0.0.0/0 → Internet Gateway
        │
        ▼
Internet Gateway
        │
        ▼
Internet
```

---

# 14. Why the Public IP Changes

Without the VPN:

```text
Windows Client
      │
      ▼
Normal ISP
      │
      ▼
Internet

Public IP:
ISP-provided address
```

With the VPN:

```text
Windows Client
      │
      │ Encrypted WireGuard
      ▼
AWS EC2 — Sydney
      │
      │ NAT + Forwarding
      ▼
Internet

Public IPv4:
AWS EC2 public IPv4
```

This is why public IP verification showed the AWS EC2 address when the VPN was active.

The result confirmed that the EC2 server was functioning as the client's internet egress point.

---

# 15. Architecture Design Decisions

## Why a Custom VPC?

A custom VPC was used instead of relying entirely on the default VPC.

This provided hands-on experience with:

- CIDR planning
- Subnet creation
- Route tables
- Internet Gateways
- Security Groups
- Resource placement

---

## Why a Public Subnet?

The WireGuard server needs to receive VPN connections from clients over the public internet.

The EC2 instance was therefore placed in a public subnet with:

```text
0.0.0.0/0 → Internet Gateway
```

and appropriate public addressing.

---

## Why WireGuard?

WireGuard provides a lightweight encrypted VPN protocol based on public/private key authentication.

It allowed the project to focus on both:

- Cloud infrastructure
- Linux/networking fundamentals

rather than using a fully managed VPN service.

---

## Why NAT on EC2?

The VPN client's `10.10.0.2` address is private.

Linux NAT allows the EC2 instance to provide internet egress for traffic originating from the WireGuard network.

No AWS NAT Gateway was required for this design.

---

## Why Disable Source/Destination Check?

The EC2 instance forwards traffic on behalf of the VPN client.

This differs from a standard EC2 workload where the instance is normally the direct source or destination of its traffic.

Disabling source/destination checking supports the EC2 instance's routing role.

---

# 16. Architecture Summary

The final architecture can be summarized as:

```text
VPN Client
Windows
10.10.0.2
     │
     │ Encrypted WireGuard
     │ UDP 51820
     ▼
┌──────────────── AWS Sydney ────────────────┐
│                                            │
│ VPC — 10.0.0.0/16                         │
│                                            │
│ ┌──── Public Subnet — 10.0.1.0/24 ─────┐ │
│ │                                       │ │
│ │     Security Group                    │ │
│ │     TCP 22 / UDP 51820                │ │
│ │              │                        │ │
│ │              ▼                        │ │
│ │        Ubuntu EC2                     │ │
│ │        WireGuard Server               │ │
│ │        wg0: 10.10.0.1                 │ │
│ │              │                        │ │
│ │        IPv4 Forwarding                │ │
│ │              │                        │ │
│ │        NAT / MASQUERADE               │ │
│ │              │                        │ │
│ │              ▼                        │ │
│ │        Public Route Table             │ │
│ │        0.0.0.0/0 → IGW                │ │
│ │                                       │ │
│ └──────────────────┬────────────────────┘ │
│                    ▼                      │
│             Internet Gateway             │
│                                            │
└────────────────────┬───────────────────────┘
                     ▼
                  Internet
```

The architecture demonstrates how multiple AWS and Linux networking components work together to create a functioning self-hosted VPN.

---

## Related Documentation

For additional project details:

- [Main Project README](../README.md)
- [Complete Setup Guide](../docs/setup-guide.md)
- [Security Considerations](../docs/security.md)
- [Troubleshooting & Lessons Learned](../docs/troubleshooting.md)
- [Cost Analysis & Resource Cleanup](../docs/cost-analysis.md)

---

## Architecture File

The architecture diagram is stored in this directory as:

```text
architecture/
├── README.md
└── aws-vpn-architecture.png
```

The diagram represents the manually deployed **Phase 1** architecture.

A future phase of the project will recreate the infrastructure using **Terraform Infrastructure as Code**.
