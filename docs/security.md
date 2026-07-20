# Security Considerations

This document describes the security decisions, controls, and lessons learned while deploying the self-hosted WireGuard VPN on AWS.

The project consists of an Ubuntu EC2 instance running WireGuard inside a custom AWS VPC in the Asia Pacific (Sydney) Region.

---

## 1. Security Architecture

The VPN uses multiple security layers:

```text
Windows Client
      │
      │ WireGuard Encryption
      │ UDP 51820
      ▼
AWS Security Group
      │
      ▼
Ubuntu EC2
      │
      ├── WireGuard Key Authentication
      ├── Linux IP Forwarding
      └── NAT / iptables MASQUERADE
              │
              ▼
       Internet Gateway
              │
              ▼
           Internet
```

Security is therefore not provided by a single component.

The architecture combines:

- WireGuard encryption
- Public/private key authentication
- AWS Security Groups
- SSH key-based authentication
- Restricted administrative access
- Linux networking controls

---

## 2. WireGuard Encryption

WireGuard creates an encrypted tunnel between the Windows client and the Ubuntu EC2 VPN server.

The server listens on:

```text
UDP 51820
```

Traffic between the client and VPN server is encrypted before traveling across the public internet.

Conceptually:

```text
Normal Client Traffic
        │
        ▼
WireGuard Encryption
        │
        ▼
Encrypted UDP Traffic
        │
        ▼
Public Internet
        │
        ▼
AWS EC2 WireGuard Server
        │
        ▼
WireGuard Decryption
```

This prevents the traffic inside the WireGuard tunnel from being transmitted as plaintext between the VPN client and VPN server.

---

## 3. WireGuard Key Authentication

WireGuard peers authenticate using asymmetric cryptographic key pairs.

Each peer has:

- A private key
- A corresponding public key

The Windows client stores its private key locally and provides its public key to the server configuration.

The server similarly has its own key pair.

```text
Windows Client                     AWS VPN Server

Client Private Key                 Server Private Key
       │                                  │
       ▼                                  ▼
Client Public Key                  Server Public Key
       │                                  │
       └──── known by server     known by client ────┘
```

Private keys must never be:

- Uploaded to GitHub
- Included in screenshots
- Shared in documentation
- Stored in public repositories

Example configuration files in this repository use placeholders rather than real keys.

```text
<CLIENT_PRIVATE_KEY>
<SERVER_PRIVATE_KEY>
<CLIENT_PUBLIC_KEY>
<SERVER_PUBLIC_KEY>
```

---

## 4. Credential Rotation Lesson

During initial troubleshooting, a WireGuard client private key was accidentally exposed.

The key was immediately treated as compromised.

Instead of continuing to use the exposed credential:

1. A new WireGuard client key pair was generated.
2. The old client public key was removed/replaced on the server.
3. The new client public key was configured as the authorized WireGuard peer.
4. The new private key remained only on the client.

This reinforced an important security principle:

> A secret that has been exposed should be rotated rather than assumed to remain safe.

No real private keys are included in this repository.

---

## 5. AWS Security Group

The EC2 instance is protected by a dedicated AWS Security Group.

Required inbound access included:

| Protocol | Port | Purpose |
|---|---:|---|
| TCP | 22 | SSH administration |
| UDP | 51820 | WireGuard VPN |

SSH access was restricted to an authorized source IP during deployment rather than intentionally exposing administrative access to the entire internet.

WireGuard uses UDP port `51820` for VPN communication.

AWS Security Groups are **stateful**.

This means response traffic associated with an allowed connection is automatically permitted without requiring an explicit reverse inbound rule.

---

## 6. SSH Security

Administration of the Ubuntu EC2 instance was performed using SSH public-key authentication.

The EC2 key pair private key must remain confidential.

Files such as:

```text
australia-vpn-key.pem
```

must never be committed to Git.

Recommended `.gitignore` rules include:

```gitignore
*.pem
*.key
.env
*.tfvars
terraform.tfstate
terraform.tfstate.*
```

SSH access should follow the principle of least privilege.

For a small learning environment, restricting TCP port 22 to the administrator's current public IP significantly reduces unnecessary exposure compared with:

```text
0.0.0.0/0
```

---

## 7. Principle of Least Privilege

Only network access required by the project should be permitted.

The VPN server requires:

```text
TCP 22
```

for administration and:

```text
UDP 51820
```

for WireGuard.

Unnecessary inbound ports should remain closed.

In a production environment, administrative access could be further improved using approaches such as AWS Systems Manager Session Manager, depending on the architecture and requirements.

---

## 8. IP Forwarding and Routing Security

IPv4 forwarding was enabled on the Ubuntu server because the EC2 instance acts as a router for VPN client traffic.

```text
net.ipv4.ip_forward=1
```

This allows packets to move between:

```text
WireGuard Interface
        │
        ▼
EC2 Network Interface
```

Enabling forwarding changes the role of the server from a normal endpoint to a network forwarding device.

Firewall and routing rules must therefore be configured carefully.

---

## 9. NAT / MASQUERADE

The project uses Linux NAT through `iptables MASQUERADE`.

This should not be confused with an **AWS NAT Gateway**.

The architecture uses:

```text
Ubuntu EC2
└── Linux iptables
    └── NAT / MASQUERADE
```

It does **not** use:

```text
AWS Managed NAT Gateway
```

NAT allows traffic originating from the WireGuard private network:

```text
10.10.0.0/24
```

to be forwarded toward the internet through the EC2 instance.

The Windows client's VPN address:

```text
10.10.0.2
```

is a private address and is not directly routable on the public internet.

---

## 10. EC2 Source/Destination Check

EC2 instances normally have source/destination checking enabled.

A standard EC2 instance generally sends or receives traffic for itself.

A VPN server acting as a router is different:

```text
VPN Client
     │
     ▼
EC2 VPN Router
     │
     ▼
Internet
```

The EC2 instance forwards packets whose original source or final destination is another system.

Source/destination checking was therefore disabled for the VPN instance to support this forwarding behavior.

This concept is also relevant to:

- NAT instances
- Virtual routers
- VPN appliances
- Network security appliances

---

## 11. IPv6 Considerations

The implemented VPN was designed primarily as an IPv4 full tunnel.

The client used:

```text
AllowedIPs = 0.0.0.0/0
```

which routes IPv4 destinations through WireGuard.

During final public-IP verification, IPv6 was reported as not detected.

However, this project does not claim to implement a complete production-grade dual-stack IPv6 VPN architecture.

A production design should explicitly decide whether to:

- Route IPv6 through the VPN
- Disable IPv6 where appropriate
- Implement equivalent IPv6 firewall and routing policies

IPv6 behavior should never simply be assumed to match IPv4 behavior.

---

## 12. Sensitive Information Excluded From GitHub

The following information must never be committed:

```text
AWS Access Keys
AWS Secret Access Keys
EC2 .pem private keys
WireGuard private keys
Passwords
Authentication tokens
.env files containing secrets
Terraform state containing sensitive information
```

Example configuration files should use placeholders:

```ini
PrivateKey = <SERVER_PRIVATE_KEY>
```

rather than real credentials.

Screenshots should also be reviewed before publication for:

- Personal/home public IP addresses
- AWS account IDs
- Credentials
- Private keys
- Tokens

---

## 13. Production Security Improvements

This project was built primarily as an educational AWS/networking environment.

Potential production improvements include:

- Stronger centralized logging and monitoring
- Automated patch management
- AWS Systems Manager for administration
- Host firewall hardening
- Automated key rotation procedures
- Explicit IPv6 security design
- CloudWatch monitoring and alarms
- Infrastructure as Code security validation
- Automated backups where required
- Multi-AZ architecture if high availability is required

---

## Security Summary

The project demonstrates several security principles:

- Encrypt data in transit
- Use key-based authentication
- Restrict administrative access
- Apply least-privilege network rules
- Never commit secrets to source control
- Rotate exposed credentials immediately
- Understand the security implications of routing and forwarding
- Verify security behavior rather than assuming configuration is correct

The project is intended for educational and portfolio purposes rather than as a production VPN service.
