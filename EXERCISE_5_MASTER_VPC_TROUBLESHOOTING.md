# Exercise 5 — Master VPC Build & Troubleshooting

**Days:** 9–10

## Scenario

You are the Lead Cloud & Backend Infrastructure Engineer for **OrderHub**. Your organization has mandated a complete enterprise-grade multi-account, cross-region AWS networking footprint.

The system spans two AWS accounts and regions:
- **Development Environment:** Account A (`111111111111`), Region `us-east-1`, `VPC-A` (`10.0.0.0/16`).
- **Production Environment:** Account B (`222222222222`), Region `us-west-2`, `VPC-B` (`20.0.0.0/16`).

The architecture supports multi-AZ redundancy, isolated public and private subnets, NAT-based outbound access, secure cross-account administration over VPC Peering, private S3 Gateway Endpoints, SSM Session Management via Interface Endpoints, fine-grained Security Groups, stateless NACLs, and diagnostic logging tools.

In this master exercise, you will review and assemble the complete canonical OrderHub enterprise architecture, verify the end-to-end administration flow, and resolve 12 real-world production network incidents.

---

## Prerequisites

VPC, CIDR, Subnet, AZ, Route Table, IGW, NAT, Security Group, NACL, VPC Peering, Gateway Endpoint, Interface Endpoint, SSM, Flow Logs, Reachability Analyzer

---

## YouTube Videos

- **Abhishek.Veeramalla — Day-7 | AWS Project Used In Production | Complete Implementation**
  - Link: https://www.youtube.com/watch?v=FZPTL_kNvXc
- **Abhishek.Veeramalla — Day-8 | AWS Scenario Based Interview Questions on EC2, IAM and VPC**
  - Link: https://www.youtube.com/watch?v=qtkWHhikLh8

---

## Build

Review and assemble the full canonical OrderHub multi-account architecture:

```
[Account A — Development (111111111111 / us-east-1)]
VPC-A: 10.0.0.0/16
 ├── AZ-A (us-east-1a)
 │    ├── Public Subnet A (10.0.1.0/24) ──► Contains: Bastion EC2 (Public IP)
 │    └── Private Subnet A (10.0.11.0/24) ──► Contains: App / Other EC2 (Private IP)
 └── AZ-B (us-east-1b)
      ├── Public Subnet B (10.0.2.0/24) ──► Contains: NAT Gateway A (Elastic IP)
      └── Private Subnet B (10.0.12.0/24) ──► Contains: Other EC2 (Private IP)

 Account A Networking Components:
  • Internet Gateway (IGW-A)
  • Public Route Table:  10.0.0.0/16 -> local | 0.0.0.0/0 -> IGW-A | 20.0.0.0/16 -> pcx-xxxx
  • Private Route Table: 10.0.0.0/16 -> local | 0.0.0.0/0 -> NAT-A | 20.0.0.0/16 -> pcx-xxxx | S3 Prefix List -> vpce-s3
  • S3 Gateway Endpoint
  • SSM Interface Endpoints (ssm, ssmmessages, ec2messages)
  • Bastion Security Group: Inbound TCP 22 from 0.0.0.0/0 | Outbound All Traffic to 20.0.0.0/16
  • Target Security Group (Account A): Inbound TCP 22 from 10.0.1.0/24 | Outbound All Traffic

                     ▲
                     │ Cross-Account / Cross-Region VPC Peering (pcx-xxxx)
                     ▼

[Account B — Production (222222222222 / us-west-2)]
VPC-B: 20.0.0.0/16
 ├── AZ-A (us-west-2a)
 │    ├── Public Subnet C (20.0.1.0/24) ──► Contains: NAT Gateway B (Elastic IP)
 │    └── Private Subnet C (20.0.11.0/24) ──► Contains: App EC2 (Private IP)
 └── AZ-B (us-west-2b)
      ├── Public Subnet D (20.0.2.0/24)
      └── Private Subnet D (20.0.12.0/24) ──► Contains: Target EC2 (Private IP)

 Account B Networking Components:
  • Internet Gateway (IGW-B)
  • Public Route Table:  20.0.0.0/16 -> local | 0.0.0.0/0 -> IGW-B
  • Private Route Table: 20.0.0.0/16 -> local | 0.0.0.0/0 -> NAT-B | 10.0.0.0/16 -> pcx-xxxx | S3 Prefix List -> vpce-s3
  • S3 Gateway Endpoint
  • SSM Interface Endpoints
  • Target Security Group (Account B): Inbound TCP 22 from 10.0.1.0/24 | Outbound All Traffic
```

### Main Administration Architecture Flow:
```
Engineer (Internet)
   │
   ▼ (SSH TCP 22)
Bastion EC2 [Account A / Public Subnet A (10.0.1.x)]
   │
   ▼ (VPC Peering pcx-xxxx)
Target EC2 [Account B / Private Subnet D (20.0.12.x)]
```

---

## Scenario-Based Verification & Troubleshooting

Solve the following 12 realistic production incidents:

### Incident 1 — Bastion Cannot Reach Target Across Peering
- **Symptom:** Developer on `Bastion-EC2` (`10.0.1.50`) attempts to SSH to `Target-EC2` (`20.0.12.50`) in Account B over VPC Peering. SSH command times out.
- **Troubleshooting Steps:**
  1. Inspect Peering Connection state in Account A/B (`Active`).
  2. Inspect Account A `Public-Route-Table`: Verify route `20.0.0.0/16 -> pcx-xxxx` exists.
  3. Inspect Account B `Private-Route-Table`: Verify return route `10.0.0.0/16 -> pcx-xxxx` exists.
  4. Inspect Account B `Target-EC2-SG`: Verify inbound TCP 22 from source CIDR `10.0.1.0/24`.
  5. Inspect Account B `Private-Subnet-D` NACL: Verify inbound/outbound rules permit TCP 22 & ephemeral ports.
- **Verification:** Correct the missing route or security group rule. SSH from Bastion to Target EC2 succeeds.
- **Reasoning:** Cross-account peering traffic requires matching route table targets and explicitly allowed firewalls on both sending and receiving sides.

### Incident 2 — Production Target Cannot Reach Internet for Package Updates
- **Symptom:** `Target-EC2` (`20.0.12.50`) in Account B fails to download OS security patches via `apt-get` or `yum`.
- **Troubleshooting Steps:**
  1. Inspect `Private-Subnet-D` route table in Account B. Verify route `0.0.0.0/0 -> NAT-Gateway-B`.
  2. Inspect `NAT-Gateway-B` status in `us-west-2` console. Ensure status is `Available`.
  3. Verify `NAT-Gateway-B` is located in `Public-Subnet-C` (`20.0.1.0/24`).
  4. Inspect `Public-Route-Table` in Account B for `0.0.0.0/0 -> IGW-B`.
  5. Confirm Elastic IP is attached to `NAT-Gateway-B`.
- **Verification:** Run `curl https://ifconfig.me` on `Target-EC2`. Confirm output matches the Elastic IP of `NAT-Gateway-B`.
- **Reasoning:** Outbound internet flow requires: `Private EC2 -> Private RT -> NAT GW (in Public Subnet) -> Public RT -> IGW -> Internet`. Breaking any link disables outbound connectivity.

### Incident 3 — S3 Access Uses NAT Gateway Unexpectedly
- **Symptom:** The finance department alerts the team that Account A NAT Gateway data charges surged by \$1,200 due to S3 data transfers.
- **Troubleshooting Steps:**
  1. Inspect Account A `Private-Route-Table`. Check if `S3 Gateway Endpoint` route is missing.
  2. If S3 Gateway Endpoint route (`pl-63a5400a -> vpce-xxxx`) is absent, traffic to S3 defaults to `0.0.0.0/0 -> NAT-Gateway-A`.
- **Verification:** Associate `S3 Gateway Endpoint` with `Private-Route-Table`. Run `aws s3 ls --region us-east-1` from private EC2 and confirm traffic matches the prefix list route instead of default NAT.
- **Reasoning:** S3 Gateway Endpoints inject prefix list routes that are more specific than `0.0.0.0/0`. Missing endpoints force S3 traffic through NAT Gateway, incurring heavy per-GB data processing charges.

### Incident 4 — SSM Session Manager Fails for Private Instance
- **Symptom:** An engineer tries to open an SSM Session Manager shell to `Private-EC2` in Account A (`10.0.11.50`). The SSM console reports `Target instance not connected`.
- **Troubleshooting Steps:**
  1. **IAM Role:** Verify EC2 instance profile has `AmazonSSMManagedInstanceCore` policy attached.
  2. **SSM Agent:** Verify SSM Agent daemon is running on OS.
  3. **VPC DNS:** Verify VPC settings **Enable DNS resolution** and **Enable DNS hostnames** are `True`.
  4. **Interface Endpoints:** Verify endpoints for `ssm`, `ssmmessages`, `ec2messages` exist in `VPC-A`.
  5. **Endpoint Security Group:** Verify `SSM-VPCE-SG` permits Inbound TCP 443 from `10.0.0.0/16`.
- **Verification:** Click **Start Session** in SSM Console. Verify successful terminal shell prompt.
- **Reasoning:** SSM Session Manager on private EC2 instances without internet access requires working IAM credentials, active OS daemon, private DNS, and reachable VPC Interface Endpoints on port 443.

### Incident 5 — Security Group Inbound Rule Is Allowed but Connection Fails
- **Symptom:** Security Group for a database instance permits TCP port 5432 from `10.0.11.0/24`. However, database connection attempts hang indefinitely.
- **Troubleshooting Steps:**
  1. Check Subnet Network ACLs (NACLs) attached to the database subnet.
  2. Inspect NACL Inbound Rules: Confirm rule allows TCP 5432.
  3. Inspect NACL Outbound Rules: Check if outbound rule blocking return traffic (ephemeral ports 1024–65535) exists.
- **Verification:** Add NACL Outbound rule allowing TCP 1024–65535 to `10.0.11.0/24`. Retest database connection.
- **Reasoning:** Security Groups are stateful (auto-allow return packets), but NACLs are stateless. If a NACL outbound rule blocks return traffic on client ephemeral ports, the TCP handshake fails despite valid Security Group rules.

### Incident 6 — Multiple Routes Exist: Which Route Wins?
- **Symptom:** `Private-Route-Table` contains the following entries:
  - Route A: `10.0.0.0/16 -> local`
  - Route B: `0.0.0.0/0 -> NAT-Gateway-A`
  - Route C: `20.0.0.0/16 -> pcx-xxxx`
  - Route D: `20.0.12.0/24 -> pcx-xxxx`
  A packet is addressed to `20.0.12.50`.
- **Troubleshooting Inquiry:**
  - Which route entry matches `20.0.12.50`?
  - Why does Route D (`20.0.12.0/24`) win over Route C (`20.0.0.0/16`) and Route B (`0.0.0.0/0`)?
- **Verification:** Confirm AWS router routes the packet to `Route D` based on **Longest Prefix Match** (`/24` has 24 matching network bits, making it more specific than `/16` or `/0`).
- **Reasoning:** Routers always prioritize the route with the highest mask length (longest prefix) matching the destination IP.

### Incident 7 — One-Direction Traffic Initiation Works, Reverse Fails
- **Symptom:** Host in Account A (`10.0.1.50`) can initiate SSH to Host in Account B (`20.0.12.50`). However, Host in Account B (`20.0.12.50`) cannot initiate SSH to Host in Account A (`10.0.1.50`).
- **Troubleshooting Steps:**
  1. Inspect `Bastion-SG` in Account A. Does it have an Inbound Rule allowing SSH (TCP 22) from `20.0.0.0/16` or `20.0.12.0/24`?
  2. Inspect Account A `Bastion-SG` outbound rules vs Account B `Target-SG` inbound rules.
- **Verification:** Explain why stateful firewalls allow return packets for connections initiated by Account A, but drop new connection attempts initiated by Account B unless explicit inbound rules exist in Account A's Security Group.
- **Reasoning:** Security Groups automatically permit return traffic for connections initiated locally. Initiating a connection in the reverse direction requires explicit inbound Security Group rules on the receiving end.

### Incident 8 — Private Endpoint DNS Resolution Failure
- **Symptom:** Application on `Private-EC2` calls `https://sqs.us-east-1.amazonaws.com` but receives public IP address `52.94.233.36` instead of internal private IP `10.0.11.99`.
- **Troubleshooting Steps:**
  1. Inspect `VPC-A` settings: **Enable DNS resolution** (`True`) and **Enable DNS hostnames** (`True`).
  2. Inspect SQS Interface Endpoint settings: Verify **Enable Private DNS name** is checked.
- **Verification:** Enable Private DNS name on SQS Interface Endpoint. Run `dig sqs.us-east-1.amazonaws.com` and confirm answer resolves to private IP `10.0.11.x`.
- **Reasoning:** Private DNS overrides public DNS hostnames to point to Interface Endpoint ENIs. If Private DNS is disabled, DNS resolves to AWS public endpoints over the internet/NAT path.

### Incident 9 — Cross-Region Peering Rule Misconfiguration
- **Symptom:** Administrator configures an inbound Security Group rule in Account B (`us-west-2`) referencing `sg-0a1b2c3d4e5f` (Bastion SG in Account A `us-east-1`). The AWS Console throws an error.
- **Troubleshooting Steps:**
  1. Check AWS Region placement of source SG (`us-east-1`) vs destination SG (`us-west-2`).
- **Verification:** Replace Security Group ID reference with explicit source CIDR `10.0.1.0/24`.
- **Reasoning:** AWS does not support Security Group ID references across different AWS Regions over VPC Peering. Cross-region rules must strictly use IPv4/IPv6 CIDR blocks.

### Incident 10 — VPC Flow Logs Investigation (`ACCEPT` vs `REJECT`)
- **Symptom:** A backend microservice cannot send events to a database. You need concrete diagnostic evidence.
- **Troubleshooting Steps:**
  1. Open CloudWatch Log Insights for `VPC-A` Flow Logs.
  2. Run query:
     `fields @timestamp, srcAddr, dstAddr, dstPort, action, logStatus`
     `| filter dstAddr = '10.0.12.50' and dstPort = 5432`
  3. Analyze results:
     - If `action == 'REJECT'`: Traffic blocked by SG or NACL.
     - If `action == 'ACCEPT'`: Network path is open; check database service status or local OS firewall (`iptables`/`ufw`).
- **Verification:** Use Flow Log empirical evidence to apply the exact fix.
- **Reasoning:** VPC Flow Logs eliminate guesswork by documenting whether AWS ENI firewalls explicitly accepted or rejected packets.

### Incident 11 — Reachability Analyzer Path Diagnostics
- **Symptom:** Complex multi-hop path between `Bastion-EC2` and `Target-EC2` is failing, and manual inspection is inconclusive.
- **Troubleshooting Steps:**
  1. Open **VPC > Reachability Analyzer**.
  2. Create path analysis from `Bastion-EC2` ENI to `Target-EC2` ENI on port 22.
  3. Analyze intermediate hop analysis output:
     `Source ENI -> Public RT -> Peering Connection -> Private RT -> NACL -> Target SG -> Target ENI`
- **Verification:** Identify the exact component highlighted in red (e.g., `Target SG: No matching inbound rule`). Fix configuration and re-run Reachability Analyzer until status reads `REACHABLE`.
- **Reasoning:** Reachability Analyzer deterministically evaluates network configuration models across VPC components without sending live packets.

### Incident 12 — Architecture Health & Vulnerability Review
- **Symptom:** Audit the entire OrderHub network architecture for single-point-of-failures, security risks, and routing misconfigurations.
- **Required Architecture Review Findings:**
  1. **Single-AZ NAT Gateway Risk:** `NAT-Gateway-A` resides in `AZ-B` (`Public-Subnet-B`). If `AZ-B` fails, private instances in `AZ-A` (`Private-Subnet-A`) lose outbound internet access. *Fix for Production:* Deploy one NAT Gateway per AZ.
  2. **Security Risk (Overly Permissive Bastion SG):** `Bastion-SG` inbound SSH is open to `0.0.0.0/0`. *Fix:* Restrict SSH to explicit corporate IP CIDR.
  3. **Overly Permissive Outbound Rules:** Private instances allow `0.0.0.0/0` outbound. *Fix:* Restrict outbound rules to required service CIDRs/ports.
  4. **Unnecessary NAT Dependency:** S3 traffic using NAT instead of Gateway Endpoint. *Fix:* Use S3 Gateway Endpoint.
  5. **Admin Access Strategy:** SSH Bastion requires key management and public IP exposure. *Fix:* Migrate to SSM Session Manager via Interface Endpoints to eliminate bastion host requirement.

---

## Networking

- **CIDR (Classless Inter-Domain Routing):** Standard IP address allocation method defining network masks (e.g., `10.0.0.0/16` vs `10.0.1.0/24`).
- **Subnets:** Segmented subdivisions of a VPC IP range bound to a single Availability Zone.
- **Availability Zones:** Physically isolated AWS data center facilities designed for fault tolerance.
- **Route Tables:** Dynamic routing rule tables that direct traffic leaving subnets and gateways.
- **Longest Prefix Match:** IP routing rule prioritizing the most specific subnet mask matching a destination IP address.
- **Internet Gateway (IGW):** VPC component enabling bi-directional 1:1 NAT communication between public subnets and the internet.
- **NAT Gateway:** Zonal AWS service enabling outbound-only 1:Many NAT for private subnets.
- **Security Groups:** Stateful firewalls controlling traffic at the ENI level.
- **Network ACLs (NACLs):** Stateless firewalls controlling traffic at the subnet boundary.
- **VPC Peering:** Private networking connection linking two VPCs across accounts and regions.
- **Gateway Endpoint:** Route table target endpoint for S3 and DynamoDB (no hourly charge).
- **Interface Endpoint:** ENI-based endpoint powered by AWS PrivateLink for private service access.
- **VPC DNS:** Amazon-provided DNS Resolver (`10.0.0.2`) required for private domain name resolution.
- **VPC Flow Logs:** Packet metadata logging mechanism capturing ACCEPT/REJECT status at ENIs.
- **Reachability Analyzer:** Static path analysis tool for testing VPC component connectivity.
- **SSM Session Manager:** Secure instance management service eliminating SSH keys and bastion hosts.

---

## Useful Tips

- **10-Step Practical AWS Network Troubleshooting Protocol:**
  1. Confirm exact **Source IP** and **Destination IP**.
  2. Check **Source Subnet Route Table** for valid route target.
  3. Check **Peering / NAT / Endpoint Path** state and placement.
  4. Inspect **Source and Destination Security Groups** (Stateful: check inbound/outbound).
  5. Inspect **Subnet Network ACLs (NACLs)** (Stateless: check inbound AND outbound ephemeral ports 1024–65535).
  6. Check **IAM Roles & SSM Agent** status when inspecting instance management.
  7. Check **VPC DNS Settings** (**DNS resolution** & **DNS hostnames**) when private endpoints fail.
  8. Run **VPC Flow Logs** to confirm `ACCEPT` vs `REJECT` action.
  9. Run **Reachability Analyzer** to pinpoint exact blocking configuration component.
  10. Retest connectivity after every single change and document root cause.
