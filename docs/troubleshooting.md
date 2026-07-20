# Troubleshooting and Lessons Learned

Building the VPN manually provided practical experience troubleshooting AWS networking, SSH authentication, WireGuard, Linux routing, and VPN client configuration.

This document describes the major issues encountered and how they were diagnosed.

---

## Troubleshooting Method

A layered troubleshooting approach was used:

```text
1. Is the EC2 instance running?
        ↓
2. Is basic network connectivity available?
        ↓
3. Is the required Security Group rule present?
        ↓
4. Can SSH reach the server?
        ↓
5. Is authentication working?
        ↓
6. Is WireGuard running?
        ↓
7. Is a WireGuard handshake occurring?
        ↓
8. Is traffic being transferred?
        ↓
9. Is IP forwarding/NAT working?
        ↓
10. Is client internet traffic actually using the VPN?
```

This avoids randomly changing multiple configurations at once.

---

# Issue 1 — AWS Browser SSH Connection Failed

An initial attempt to connect to the EC2 instance resulted in an SSH connection error.

The infrastructure was checked systematically.

The EC2 instance had:

- A public IPv4 address
- A public subnet
- An Internet Gateway
- A default route
- A Security Group rule for SSH

Direct SSH from Windows PowerShell was then tested.

This helped separate browser-based EC2 connectivity from direct SSH connectivity.

### Lesson

A public IPv4 address alone does not guarantee internet connectivity.

The complete path requires:

```text
EC2
  ↓
Public Subnet
  ↓
Route Table
0.0.0.0/0 → IGW
  ↓
Internet Gateway
```

and appropriate Security Group rules.

---

# Issue 2 — `Permission denied (publickey)`

Direct SSH reached the EC2 instance but returned:

```text
ubuntu@<EC2-IP>: Permission denied (publickey)
```

This error was an important diagnostic clue.

Because the SSH server responded, basic network connectivity was already working.

Therefore, the problem was not primarily:

- VPC routing
- Internet Gateway
- Public subnet
- TCP connectivity

The problem was at the authentication layer.

The following were verified:

```text
AMI:
Ubuntu

SSH Username:
ubuntu

EC2 Key Pair:
australia-vpn-key
```

The correct `.pem` private key associated with the EC2 instance had to be used.

Example:

```bash
ssh -i "path/to/australia-vpn-key.pem" ubuntu@<EC2_PUBLIC_IP>
```

### Lesson

Different errors indicate different layers.

```text
Connection timeout
        ↓
Likely connectivity/routing/firewall investigation

Permission denied (publickey)
        ↓
Authentication/key/username investigation
```

Do not modify working VPC infrastructure when the error clearly indicates an authentication problem.

---

# Issue 3 — Understanding AWS SSH Keys vs WireGuard Keys

Two completely separate cryptographic systems were involved.

### EC2 SSH Key

```text
australia-vpn-key.pem
```

Purpose:

```text
Windows Administrator
        ↓
SSH
        ↓
Ubuntu EC2
```

### WireGuard Keys

Each VPN peer has its own WireGuard public/private key pair.

Purpose:

```text
Windows WireGuard Client
        ↓
Encrypted VPN
        ↓
EC2 WireGuard Server
```

These keys are not interchangeable.

### Lesson

A cloud VPN environment may involve multiple independent authentication mechanisms.

Understanding which credential belongs to which protocol is essential.

---

# Issue 4 — WireGuard Private Key Accidentally Exposed

During configuration troubleshooting, a WireGuard client private key was accidentally exposed.

The key was immediately considered compromised.

The response was:

1. Generate a new WireGuard client key pair.
2. Keep the new private key only on the Windows client.
3. Replace the old client public key in the server configuration.
4. Restart/reload WireGuard.
5. Verify the new peer.

The server then showed the new client public key.

### Lesson

Never continue using an exposed secret.

Use:

```text
Exposure
   ↓
Revoke / Replace / Rotate
   ↓
Verify new credential
```

rather than attempting to hide the old credential.

---

# Issue 5 — WireGuard Peer Configured But No Handshake

Initially, the server displayed a configured peer:

```text
peer: <CLIENT_PUBLIC_KEY>
allowed ips: 10.10.0.2/32
```

but there was no:

```text
latest handshake
```

A configured peer does not prove that the client has successfully connected.

A successful connection should show information such as:

```text
endpoint: <CLIENT_ENDPOINT>
latest handshake: <RECENT TIME>
transfer: <RECEIVED>, <SENT>
```

### Lesson

There is a difference between:

```text
Peer configured
```

and:

```text
Peer connected
```

The `latest handshake` field is an important WireGuard diagnostic indicator.

---

# Issue 6 — SSH Connection Reset When VPN Was Activated

During testing, activating the WireGuard full tunnel caused an existing SSH connection to reset.

The Windows WireGuard configuration used:

```text
AllowedIPs = 0.0.0.0/0
```

This changes IPv4 routing so that internet-bound traffic is directed through the VPN.

An existing SSH session to the EC2 public IP can therefore be affected by the route change.

### Lesson

Changing a client's default route can affect existing management connections.

This is particularly important when configuring:

- VPNs
- Routers
- Firewalls
- Remote servers

Remote network changes should be made carefully because an incorrect route can disconnect the administrator.

---

# Issue 7 — WireGuard Connected but Public IP Did Not Change

A major troubleshooting lesson occurred when WireGuard successfully established a handshake.

The server showed:

```text
latest handshake: <RECENT>
transfer: <DATA RECEIVED/SENT>
```

However, an IP-checking service still showed the client's original ISP IPv4 address.

This demonstrated:

```text
Successful VPN handshake
          ≠
All internet traffic routed through VPN
```

The Windows client routing configuration needed to support a full tunnel.

The important setting was:

```text
AllowedIPs = 0.0.0.0/0
```

This represents all IPv4 destinations.

Conceptually:

```text
Without full-tunnel routing:

Normal Internet → ISP
VPN Network      → WireGuard


With 0.0.0.0/0:

IPv4 Internet Traffic
        ↓
WireGuard
        ↓
AWS VPN Server
```

### Lesson

VPN connectivity and VPN routing are separate concepts.

Always verify the actual traffic path.

---

# Issue 8 — Verifying the VPN Properly

The VPN was validated at multiple levels.

## Level 1 — WireGuard Service

```bash
sudo wg
```

Confirmed:

- WireGuard interface running
- UDP listening port
- Configured peer

## Level 2 — Cryptographic Connection

A recent:

```text
latest handshake
```

confirmed peer communication.

## Level 3 — Traffic

Increasing:

```text
transfer
```

counters confirmed data movement.

## Level 4 — Internet Egress

The client's public IPv4 address was checked before and after VPN activation.

Before:

```text
Client → ISP → Internet
```

After:

```text
Client
   ↓
WireGuard
   ↓
AWS Sydney EC2
   ↓
Internet
```

The final public IPv4 matched the AWS EC2 public IPv4.

This confirmed full-tunnel internet routing.

---

# Issue 9 — IPv6 Behavior

During an earlier test, the client still had detectable IPv6 connectivity while the VPN architecture primarily handled IPv4.

This highlighted a potential issue:

```text
IPv4 → VPN

IPv6 → Potentially different path
```

The final verification showed:

```text
IPv4:
AWS EC2 public IPv4

IPv6:
Not detected
```

However, the project does not claim a complete dual-stack IPv6 VPN implementation.

### Lesson

When testing a VPN, checking only IPv4 may not be sufficient.

IPv4 and IPv6 routing should be considered separately.

---

# Useful Diagnostic Commands

### Check WireGuard

```bash
sudo wg
```

### Check WireGuard service

```bash
sudo systemctl status wg-quick@wg0
```

### Check interfaces

```bash
ip addr
```

### Check Linux routing

```bash
ip route
```

### Check IPv4 forwarding

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

### Check NAT rules

```bash
sudo iptables -t nat -L -n -v
```

### Test public IPv4 from Windows

```powershell
curl.exe -4 https://ifconfig.me
```

Sensitive IP addresses should be removed from screenshots before publication.

---

# Key Troubleshooting Lessons

The most important lesson was to troubleshoot from the lowest relevant layer upward.

For example:

```text
Can I reach the EC2 instance?
        ↓
Can I authenticate?
        ↓
Is WireGuard running?
        ↓
Is UDP 51820 reachable?
        ↓
Is there a handshake?
        ↓
Is traffic transferring?
        ↓
Is Linux forwarding traffic?
        ↓
Is NAT working?
        ↓
Is Windows routing through the tunnel?
        ↓
Did the public IP actually change?
```

This approach is significantly more effective than changing multiple AWS and Linux settings simultaneously.

It also makes troubleshooting applicable beyond this project to cloud support, system administration, networking, and DevOps environments.
