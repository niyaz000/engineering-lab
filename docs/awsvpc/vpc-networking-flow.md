# AWS VPC Networking: Full Request Journey

> *You can outsource your thinking but not your understanding.*

A packet-level trace of a typical web service deployed on AWS — from DNS resolution through load balancer, private subnet, database, and back out to an external API. No theory, just the path.

Diagrams show the request direction. Responses retrace the same path — Security Groups handle this automatically (stateful), NACLs require explicit return rules (stateless).

---

## Acronyms

| Acronym | Full Form                          | What it means                                              |
|---------|------------------------------------|------------------------------------------------------------|
| VPC     | Virtual Private Cloud              | Your isolated private network inside AWS                   |
| IGW     | Internet Gateway                   | Connects the VPC to the public internet                    |
| NAT     | Network Address Translation        | Rewrites IP addresses as packets cross a boundary          |
| SNAT    | Source NAT                         | Specifically replaces the *source* IP — what NAT GW does to hide your EC2's private IP |
| NACL    | Network Access Control List        | Stateless firewall rules applied at the subnet boundary    |
| SG      | Security Group                     | Stateful firewall rules applied to a specific resource     |
| ALB     | Application Load Balancer          | Layer 7 load balancer — routes HTTP/S traffic to targets   |
| TG      | Target Group                       | The set of EC2s the ALB forwards traffic to                |
| EC2     | Elastic Compute Cloud              | A virtual machine (your app server)                        |
| RDS     | Relational Database Service        | AWS managed relational database (Postgres, MySQL, etc.)    |
| EIP     | Elastic IP                         | A static public IP address you own in AWS                  |
| CIDR    | Classless Inter-Domain Routing     | IP range notation — `10.0.0.0/16` means 65,536 addresses  |
| DNS     | Domain Name System                 | Translates `api.hookly.com` to an IP address               |
| TLS     | Transport Layer Security           | Encryption protocol behind HTTPS                           |

---

## Key Players

| Concept              | What it does                                                                 |
|----------------------|------------------------------------------------------------------------------|
| **Security Group**   | Firewall on a resource. **Stateful** — allow inbound `:443` and the response packet is automatically allowed out. No extra outbound rule needed. |
| **NACL**             | Firewall at the subnet boundary. **Stateless** — allow inbound `:443` and you must *also* explicitly allow outbound `:1024–65535` (ephemeral ports) or the response is dropped. Rules are evaluated **in order by rule number** — first match wins, so rule ordering matters. |
| **Route Table**      | Decides where a packet goes next (IGW, NAT GW, or stays local). Uses **longest prefix match** — most specific route wins. `10.0.2.0/24` beats `0.0.0.0/0` for traffic to `10.0.2.15`. |
| **Internet Gateway** | The door between your VPC and the public internet                            |
| **NAT Gateway**      | Lets private resources reach the internet without being reachable from it    |
| **ALB**              | Receives incoming traffic and forwards it to a healthy target (your EC2)     |

---

## Service Diagram

```
                                              ┌─────────────────────┐
                                        ┌───► │  RDS  (Postgres)    │
                                        │     └─────────────────────┘
  ┌──────────┐     ┌──────────┐     ┌───┴──────────┐
  │  Client  │────►│    LB    │────►│  App Server  │
  └──────────┘     └──────────┘     └───┬──────────┘
                                        │
                                        ▼
                                  ┌─────────────────────┐
                                  │  External Service   │
                                  │  (api.stripe.com)   │
                                  └─────────────────────┘
```

## Reference Values

| Resource         | Value                                  |
|------------------|----------------------------------------|
| Client           | `203.0.113.5`                          |
| Domain           | `api.hookly.com` → `54.23.145.67`      |
| VPC CIDR         | `10.0.0.0/16`                          |
| Public Subnet    | `10.0.1.0/24`                          |
| Private Subnet   | `10.0.2.0/24`                          |
| ALB              | `54.23.145.67:443`                     |
| NAT Gateway      | `10.0.1.100`  /  EIP `52.14.88.12`     |
| EC2 Web Server   | `10.0.2.15:8080`                       |
| RDS Postgres     | `10.0.2.50:5432`                       |
| External Service | `api.stripe.com`                       |

---

## Architecture Overview

```
                   ┌─────────────── INTERNET ────────────────────────────┐
                   │  Client 203.0.113.5           api.stripe.com        │
                   └──────────┬─────────────────────────────▲────────────┘
                              │                             │
                   ┌──────────▼───────────────────────────────────────────┐
                   │              Internet Gateway (IGW)                  │
                   └──────────┬───────────────────────────▲───────────────┘
                              │                           │
  ┌───────────────────────────▼───────────────────────────┼──────────────────┐
  │ VPC 10.0.0.0/16           │                           │                  │
  │                           │                           │                  │                                                      │                           ▼                           |                  │
  │  ┌──── Public Subnet 10.0.1.0/24 ─────────────────────┴──────────────┐   │
  │  │                        │                                          │   │
  │  │                        ▼                                          │   │
  │  │   ┌────────────────────┐         ┌──────────────────────────────┐ │   │
  │  │   │ ALB                │         │ NAT Gateway                  │ │   │
  │  │   │ 54.23.145.67:443   │◄────┐   │ 10.0.1.100                   │ │   │
  │  │   │ SG: 0.0.0.0/0:443  │     │   │ EIP: 52.14.88.12             │ │   │
  │  │   └────────┬───────────┘     │   └──────────────────────────────┘ │   │
  │  └────────────┼─────────────────────────────────────▲────────────────┘   │
  │               │ :8080           │                   │                    │
  │               ▼                 │                   │                    │
  │  ┌──── Private Subnet 10.0.2.0/24 ───────────────────────────────────┐   │
  │  │               │              │                   │                │   │
  │  │   ┌───────────▼──────────────┘                   │                │   │
  │  │   │ EC2  10.0.2.15:8080      │   outbound ───────┘                │   │
  │  │   │ SG in: ALB-SG → :8080    │                                    │   │
  │  │   └───────────┬──────────────┘                                    │   │
  │  │               │ :5432                                             │   │
  │  │   ┌───────────▼──────────────┐                                    │   │
  │  │   │ RDS  10.0.2.50:5432      │                                    │   │
  │  │   │ SG in: EC2-SG → :5432    │                                    │   │
  │  │   └──────────────────────────┘                                    │   │
  │  └───────────────────────────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

## Inbound Flow — `203.0.113.5 → api.hookly.com`

```
  ┌────────────────────────────────────────────────────────────────────────────┐
  │  INTERNET                                                                  │
  │                                                                            │
  │  ① Client  203.0.113.5                                                    │
  │       DNS:  api.hookly.com  ──►  54.23.145.67   (ALB public IP)            │
  │       TCP SYN  ──►  54.23.145.67:443                                       │
  └─────────────────────────────┬──────────────────────────────────────────────┘
                                │ ② HTTPS :443
  ┌─────────────────────────────▼─────────────────────────────────────────────┐
  │  Internet Gateway (IGW)                                                   │
  └─────────────────────────────┬─────────────────────────────────────────────┘
                                │ ③ routes to VPC
  ┌── Public Subnet 10.0.1.0/24 ▼───────────────────────────────────────────────┐
  │                                                                             │
  │  ④ NACL  (stateless — both directions must be explicitly allowed)          │
  │       in:  TCP :443  from 0.0.0.0/0              ✓  rule 100                │
  │       out: TCP :1024-65535  to 0.0.0.0/0         ✓  ephemeral ports         │
  │                                                                             │
  │  ⑤ Route Table                                                             │
  │       0.0.0.0/0    ──►  igw-0a1b2c3d   (internet traffic exits via IGW)     │
  │       10.0.0.0/16  ──►  local                                               │
  │                                                                             │
  │  ⑥ ALB  54.23.145.67                                                       │
  │       SG in:  TCP :443  src 0.0.0.0/0      ✓                                │
  │       SG in:  TCP :80   src 0.0.0.0/0      ✓  (redirect → HTTPS)            │
  │       SG out: all                           ✓                               │
  │       Listener :443  ──►  Target Group  ──►  10.0.2.15:8080                 │
  │                                                                             │
  └──────────────────────────────┬──────────────────────────────────────────────┘
                                 │ ⑦ ALB forwards to target group :8080
  ┌── Private Subnet 10.0.2.0/24 ▼────────────────────────────────────────────────┐
  │                                                                               │
  │  ⑧ NACL  (stateless)                                                         │
  │       in:  TCP :8080  from 10.0.1.0/24       ✓  (only from public subnet)     │
  │       out: TCP :1024-65535  to 10.0.1.0/24   ✓  (ephemeral return traffic)    │
  │                                                                               │
  │  ⑨ Route Table                                                               │
  │       10.0.0.0/16  ──►  local                                                 │
  │       0.0.0.0/0    ──►  nat-0x1y2z3w          (outbound internet via NAT)     │
  │                                                                               │
  │  ⑩ EC2  10.0.2.15:8080                                                       │
  │       SG in:  TCP :8080  src sg-alb       ✓  (SG ref — only ALB can reach)    │
  │       SG out: all                         ✓                                   │
  │              │                                                                │
  │              │ ⑪ :5432  (stays within private subnet, SG enforced)           │
  │              ▼                                                                │
  │       RDS  10.0.2.50:5432                                                     │
  │            SG in:  TCP :5432  src sg-ec2  ✓  (SG ref — only EC2 can reach)    │
  │                                                                               │
  └───────────────────────────────────────────────────────────────────────────────┘
```

---

## Outbound Flow — `EC2 → api.stripe.com`

> EC2 is in a **private subnet** — no public IP, no direct internet access.
> All outbound internet traffic routes through NAT Gateway in the public subnet.

```
  ┌── Private Subnet 10.0.2.0/24 ───────────────────────────────────────────────────┐
  │                                                                                 │
  │  ⑫ EC2  10.0.2.15  calls  api.stripe.com                                       |
  │       Route Table:  0.0.0.0/0  ──►  NAT GW (nat-0x1y2z3w)                       │
  │                                                                                 │
  └──────────────────────────────────────┬──────────────────────────────────────────┘
                                         │ ⑬
  ┌── Public Subnet 10.0.1.0/24 ─────────▼──────────────────────────────────────────┐
  │                                                                                 │
  │  ⑭ NAT Gateway  10.0.1.100                                                     │
  │       SNAT:  src 10.0.2.15  ──►  EIP 52.14.88.12                                │
  │                                                                                 │
  │       Stripe sees the request from 52.14.88.12, not the private EC2 IP.         │
  │       Response returns:  api.stripe.com → IGW → NAT GW → EC2 10.0.2.15          │
  │                                                                                 │
  └──────────────────────────────────────┬──────────────────────────────────────────┘
                                         │ ⑮
  ┌──────────────────────────────────────▼─────────────────────────────────────────┐
  │  Internet Gateway (IGW)                                                        │
  └──────────────────────────────────────┬─────────────────────────────────────────┘
                                         │ ⑯
  ┌──────────────────────────────────────▼──────────────────────────────────────────┐
  │  INTERNET  ──►  api.stripe.com                                                  │
  └─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Flow Reference

| Step | Direction | From                  | To                    | Port  | Layer        | Note                                      |
|------|-----------|-----------------------|-----------------------|-------|--------------|-------------------------------------------|
| ①   | Inbound   | Client                | DNS                   | 53    | DNS          | Resolves `api.hookly.com` → `54.23.145.67` |
| ②   | Inbound   | Client `203.0.113.5`  | ALB `54.23.145.67`    | 443   | Internet     | TCP SYN to ALB public IP                  |
| ③   | Inbound   | IGW                   | Public Subnet         | 443   | IGW          | Routes public IP into VPC                 |
| ④   | Inbound   | IGW                   | Public Subnet         | 443   | NACL         | Stateless check — inbound :443 allowed    |
| ⑤   | Inbound   | IGW                   | Public Subnet         | —     | Route Table  | `0.0.0.0/0 → IGW`, `10.0.0.0/16 → local` |
| ⑥   | Inbound   | Internet              | ALB `54.23.145.67`    | 443   | Security Group | SG allows :443 from `0.0.0.0/0`         |
| ⑦   | Inbound   | ALB                   | EC2 `10.0.2.15`       | 8080  | ALB          | Listener forwards to Target Group         |
| ⑧   | Inbound   | ALB subnet            | Private Subnet        | 8080  | NACL         | Stateless check — inbound :8080 from `10.0.1.0/24` |
| ⑨   | Inbound   | ALB subnet            | Private Subnet        | —     | Route Table  | `10.0.0.0/16 → local`, `0.0.0.0/0 → NAT GW` |
| ⑩   | Inbound   | ALB                   | EC2 `10.0.2.15`       | 8080  | Security Group | SG allows :8080 from ALB-SG only        |
| ⑪   | Internal  | EC2 `10.0.2.15`       | RDS `10.0.2.50`       | 5432  | Security Group | SG allows :5432 from EC2-SG only        |
| ⑫   | Outbound  | EC2 `10.0.2.15`       | NAT GW `10.0.1.100`   | any   | Route Table  | `0.0.0.0/0 → NAT GW` in private subnet   |
| ⑬   | Outbound  | Private Subnet        | Public Subnet         | any   | NACL         | Outbound ephemeral ports allowed          |
| ⑭   | Outbound  | NAT GW `10.0.1.100`   | IGW                   | any   | NAT Gateway  | SNAT: `10.0.2.15` → EIP `52.14.88.12`    |
| ⑮   | Outbound  | NAT GW EIP            | IGW                   | any   | IGW          | Routes EIP traffic to internet            |
| ⑯   | Outbound  | IGW                   | `api.stripe.com`      | 443   | Internet     | Stripe sees src `52.14.88.12`             |

---

## Security Layers at a Glance

| Layer          | Stateful | Key Rule                              |
|----------------|----------|---------------------------------------|
| NACL           | No  — each packet judged independently | First matching rule number wins. Allow inbound `:443` → must also allow outbound `:1024–65535` or response is dropped |
| Security Group | Yes — tracks the connection            | Allow inbound `:443` → outbound response is automatically permitted                                                   |
| Route Table    | —                                      | Longest prefix match — `10.0.2.0/24` beats `0.0.0.0/0`. Routing decision only, not allow/deny                       |
| ALB            | —        | Terminates TLS, forwards to TG        |
| NAT Gateway    | —        | SNAT: hides private IP behind EIP     |
