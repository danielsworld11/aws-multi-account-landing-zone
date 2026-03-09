┌──────────────────────────────────────────┐
                        │                 VPC                      │
                        │         10.x.0.0/16 (Dev/Prod)           │
                        └──────────────────────────────────────────┘
                                      │
        ┌─────────────────────────────┴─────────────────────────────┐
        │                                                           │
┌──────────────────────┐                                   ┌──────────────────────┐
│  Public Subnet A     │                                   │  Public Subnet B     │
│  10.x.1.0/24          │                                   │  10.x.2.0/24          │
│  • ALB                │                                   │  • ALB (AZ‑B)         │
│  • NAT Gateway (AZ‑A) │                                   │                      │
└──────────────────────┘                                   └──────────────────────┘
        │                                                           │
        │                                                           │
┌──────────────────────┐                                   ┌──────────────────────┐
│  Private Subnet A    │                                   │  Private Subnet B    │
│  10.x.3.0/24          │                                   │  10.x.4.0/24          │
│  • ECS Tasks          │                                   │  • ECS Tasks          │
│  • App Containers     │                                   │  • App Containers     │
└──────────────────────┘                                   └──────────────────────┘

Internet Gateway (IGW) → Public Subnets → NAT Gateway → Private Subnets

This architecture follows AWS best practices:  
two Availability Zones, public subnets for ingress, private subnets for compute, and NAT‑based egress.

CIDR Strategy

Each environment uses a clean, non‑overlapping /16 block:

•  Dev: 10.0.0.0/16
•  Prod: 10.1.0.0/16

Why this approach works well:

•  Simple and predictable  
Each environment gets its own address space, making subnetting and routing straightforward.
•  Future‑proof  
/16 provides enough IP space for ECS tasks, endpoints, scaling, and future services.
•  No overlap  
Using different second octets (10.0.x.x vs 10.1.x.x) avoids conflicts when adding:
  ⁠◦  VPC peering
  ⁠◦  Transit Gateway
  ⁠◦  Shared Services networking
•  Enterprise‑aligned  
Many organizations follow a pattern like 10.<env>.0.0/16 for clarity and scalability.

 
 Why Compute Runs in Private Subnets

Placing ECS tasks (or EC2 instances) in private subnets is a core AWS security best practice.

1. No direct internet exposure

Private subnets have no route to the Internet Gateway, meaning:

•  No public IPs
•  No inbound traffic
•  No external attack surface

Only the ALB in the public subnet can reach them.

2. Controlled outbound access via NAT

Compute can still reach:

•  ECR
•  SSM
•  AWS APIs
•  Package repositories

…but only through the NAT Gateway, ensuring:

•  Outbound‑only connectivity
•  No unsolicited inbound traffic
•  Full auditability

3. ALB handles all ingress

The Application Load Balancer terminates TLS and forwards traffic to private subnets, providing:

•  Centralized entry point
•  Health checks
•  WAF integration
•  Zero direct exposure of workloads

4. Production‑grade isolation

Private subnets enforce:

•  Least‑privilege networking
•  Segmentation between tiers
•  Compliance‑friendly architecture
•  Reduced blast radius

5. Matches AWS reference architectures

AWS recommends:

•  Public subnets: ALB, NAT
•  Private subnets: ECS, EC2, RDS, Redis, internal services

This is the same pattern used by Control Tower and enterprise landing zones.
