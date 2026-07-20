# Cost Analysis and Resource Cleanup

This document explains the main AWS cost considerations for the self-hosted WireGuard VPN project and how resources are managed when the environment is not required.

> AWS pricing and Free Tier benefits can change and may differ depending on account eligibility, region, resource type, and usage. Always verify current pricing in the AWS Billing console and official AWS pricing documentation.

---

# 1. Architecture and Cost Sources

The VPN architecture consists primarily of:

```text
Custom VPC
    │
    ├── Public Subnet
    │
    ├── Route Table
    │
    ├── Internet Gateway
    │
    └── EC2 Instance
            │
            ├── Ubuntu
            └── WireGuard
```

Potential cost sources include:

- EC2 compute
- EBS storage
- Public IPv4 addressing
- Internet data transfer
- Optional monitoring or logging services

Not every networking component independently creates an hourly charge, but the architecture must be evaluated as a whole.

---

# 2. EC2 Compute

The WireGuard server runs on an Amazon EC2 instance.

EC2 cost depends on factors including:

- Instance type
- Region
- Operating system
- Pricing model
- Running time
- Free Tier or promotional eligibility

The VPN server does not need to run continuously for this learning project.

When testing is complete, the instance can be stopped.

```text
Running EC2
     │
     ▼
Compute usage

Stopped EC2
     │
     ▼
No running instance compute usage
```

However:

```text
STOPPED ≠ EVERYTHING IS FREE
```

Other associated resources may still have costs.

---

# 3. EBS Storage

The EC2 instance uses an EBS volume as its virtual disk.

Stopping the EC2 instance does not automatically delete its EBS volume.

Therefore:

```text
EC2 Stopped
    │
    └── EBS Volume Still Exists
```

Storage can continue to exist until the volume is deleted.

When terminating an EC2 instance, the root volume may be configured for:

```text
Delete on termination
```

This setting should be verified rather than assumed.

---

# 4. Public IPv4 Addressing

The VPN server requires an internet-reachable endpoint for the WireGuard client.

AWS public IPv4 addressing can have cost implications depending on current AWS pricing and account benefits.

The project initially used an automatically assigned EC2 public IPv4 address.

One operational consideration is that stopping and starting an EC2 instance can result in a different automatically assigned public IPv4 address.

If the public IP changes, the WireGuard client endpoint must be updated:

```ini
Endpoint = <NEW_EC2_PUBLIC_IP>:51820
```

A stable public endpoint may improve usability, but the cost implications of options such as public IPv4 resources should be evaluated before deployment.

---

# 5. Data Transfer

A VPN forwards user internet traffic through AWS.

For example:

```text
Windows Client
      │
      ▼
AWS VPN Server
      │
      ▼
YouTube / Websites / Downloads
```

This means normal browsing traffic becomes AWS network traffic.

High-bandwidth activities can significantly increase data transfer compared with simple infrastructure testing.

Examples include:

- Video streaming
- Large downloads
- Game downloads
- Cloud backups
- Large software updates
- Continuous VPN use

For a learning project, usage should therefore be controlled and billing monitored.

---

# 6. Internet Gateway

The architecture uses an Internet Gateway attached to the VPC.

The Internet Gateway provides the VPC path to and from the public internet when combined with appropriate routing and public addressing.

The project uses:

```text
Public Route Table

0.0.0.0/0
      ↓
Internet Gateway
```

Cost analysis should focus on the resources and traffic associated with using the architecture rather than assuming that every diagram component has an independent hourly fee.

---

# 7. No AWS NAT Gateway

This project deliberately does **not** use an AWS managed NAT Gateway.

Instead:

```text
WireGuard Client
10.10.0.2
      │
      ▼
Ubuntu EC2
      │
iptables MASQUERADE
      │
      ▼
Internet
```

NAT is performed by Linux on the EC2 VPN server.

This distinction is important:

```text
NAT
=
Networking technique


AWS NAT Gateway
=
Managed AWS service
```

An AWS NAT Gateway was unnecessary for this architecture because the VPN server itself was deployed in a public subnet and handled VPN-client NAT using Linux.

Avoiding unnecessary cloud resources also helps reduce architecture complexity and potential cost.

---

# 8. Stop vs Terminate

There are two important lifecycle actions.

## Stop

Use **Stop** when the environment will be used again soon.

```text
EC2
Running
   ↓
STOP
   ↓
EC2 Stopped
```

Advantages:

- Instance can be started again.
- Configuration remains on the EBS volume.
- Useful for temporary learning environments.

Considerations:

- EBS storage remains.
- Public IPv4 behavior/cost should be reviewed.
- Other resources remain provisioned.

---

## Terminate

Use **Terminate** when the environment is no longer needed.

```text
EC2
Running
   ↓
TERMINATE
   ↓
Instance permanently removed
```

Termination is appropriate when:

- Testing is complete.
- Screenshots/documentation have been saved.
- The infrastructure will be rebuilt later.
- The project is moving to Terraform.

Always verify associated resources after termination.

---

# 9. Cleanup Procedure

Before deleting resources:

```text
1. Capture required screenshots.
2. Sanitize screenshots.
3. Save architecture documentation.
4. Save non-secret configuration examples.
5. Verify no credentials are being published.
```

Then clean up project resources.

A typical cleanup process includes reviewing:

```text
EC2 Instance
      ↓
EBS Volumes
      ↓
Elastic IPs / Public IP Resources
      ↓
Security Group
      ↓
Custom Route Table
      ↓
Internet Gateway
      ↓
Subnet
      ↓
Custom VPC
```

Dependencies may require resources to be removed in a particular order.

---

# 10. EC2 Instance

Terminate the VPN EC2 instance when it is no longer required.

Before termination, verify that:

- Documentation has been saved.
- Screenshots have been captured.
- No required files exist only on the instance.

After termination, verify whether associated EBS volumes were automatically deleted.

---

# 11. EBS Volumes

Review:

```text
EC2
→ Elastic Block Store
→ Volumes
```

Check for unnecessary volumes remaining after EC2 termination.

Unused storage should not be left running simply because the EC2 instance was deleted.

---

# 12. Elastic IP / Public IPv4 Resources

Check whether any Elastic IP or other separately managed public IPv4 resource was created.

If an Elastic IP is no longer required, release it according to AWS resource-management best practices.

This project did not require an AWS NAT Gateway.

---

# 13. Security Group

Delete the project-specific Security Group after dependent resources have been removed.

Example:

```text
australia-vpn-sg
```

Do not accidentally delete unrelated/default security infrastructure.

---

# 14. Internet Gateway

The custom Internet Gateway can be removed when the VPC is being deleted.

Conceptually:

```text
Internet Gateway
      │
      ▼
Detach from custom VPC
      │
      ▼
Delete Internet Gateway
```

AWS dependency rules may require resources to be detached before deletion.

---

# 15. Subnet and Route Table

Delete the custom project resources:

```text
vpn-public-subnet

vpn-public-route-table
```

after dependent resources have been removed.

The exact order may depend on AWS resource dependencies and associations.

---

# 16. VPC

Finally, delete only the custom project VPC:

```text
australia-vpn-vpc
```

Do not accidentally delete an unrelated/default VPC.

Before deletion, verify that no required resources remain inside it.

---

# 17. Post-Cleanup Verification

After cleanup, review the AWS console for project-related resources.

Check:

```text
EC2 Instances
EBS Volumes
Elastic IPs
NAT Gateways
Load Balancers
VPC Resources
```

The project did not intentionally require:

```text
AWS NAT Gateway
Load Balancer
RDS
```

so unexpected resources in those categories should be investigated rather than ignored.

Also review:

```text
AWS Billing
AWS Cost Explorer
AWS Free Tier / Credits
AWS Budgets
```

as available for the account.

---

# 18. Cost Optimization Lessons

This project demonstrated several cloud cost principles.

## Stop resources when they are not needed

Learning environments do not need to run continuously.

## Delete resources after experiments

Cloud resources can persist after testing unless explicitly removed.

## Understand data transfer

A VPN can generate substantially more network traffic than a simple EC2 lab.

## Avoid unnecessary architecture

The project did not deploy an AWS NAT Gateway because the architecture did not require one.

## Use budgets and billing alerts

Cost monitoring should be configured before experimenting extensively with cloud infrastructure.

## Treat cost as an architecture requirement

A technically working architecture is not necessarily an economically efficient architecture.

Cloud design should consider:

```text
Security

Reliability

Performance

Operational Excellence

Cost
```

rather than focusing only on whether the application works.

---

# 19. Future Cost Improvement

The Terraform phase of this project will make resource lifecycle management easier.

Instead of manually creating and deleting resources:

```text
AWS Console
Create resources manually
        ↓
Test
        ↓
Delete individually
```

Infrastructure as Code can provide a reproducible workflow:

```text
terraform apply
        ↓
Infrastructure Created
        ↓
Testing
        ↓
terraform destroy
        ↓
Infrastructure Removed
```

This is particularly useful for temporary learning environments.

It reduces the chance of forgetting manually created resources, although the Terraform plan and state must still be managed carefully.

---

# Summary

The main potential cost areas for this project are:

| Component | Cost Consideration |
|---|---|
| EC2 | Compute while running, subject to account benefits/pricing |
| EBS | Storage persists while provisioned |
| Public IPv4 | Check current AWS pricing/account benefits |
| Data Transfer | VPN internet usage can generate transfer charges |
| VPC networking | Evaluate each provisioned resource individually |
| NAT Gateway | Not used in this project |

The safest practice for a temporary learning environment is:

```text
Build
  ↓
Test
  ↓
Document
  ↓
Stop when temporarily unused
  ↓
Terminate/Delete when finished
  ↓
Verify Billing and remaining resources
```

This project is intended as a hands-on AWS networking lab and portfolio project, not as a permanently running production VPN service.
