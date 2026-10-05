# Exercise 2 — Public & Private Networking + NAT Gateway

**Days:** 3–4

## Scenario

OrderHub's application architecture is expanding. The team is deploying public-facing frontend load balancers alongside private backend application microservices.

The security requirement for OrderHub is explicit:
1. **Private Backend Instances** must remain strictly private. They must never have public IP addresses or be directly accessible from the internet.
2. **Outbound Internet Access** is mandatory for backend instances to download security updates, install third-party dependencies, and call external third-party payment gateways.
3. **Inbound Block:** No external host on the internet must be capable of initiating a connection to private backend instances.

In this exercise, you will connect **VPC-A (Account A / us-east-1)** to the internet using an **Internet Gateway (IGW)** for public subnets and a **NAT Gateway** for private subnets according to the canonical architecture.

---

## Prerequisites

VPC, Subnets, Route Tables, Internet Gateway, NAT Gateway, Elastic IP, Security Groups, IPv4, CIDR

---

## YouTube Videos

- **Abhishek.Veeramalla — Learn Networking in 3 Hours | Networking Fundamentals + AWS VPC Networking**
  - Link: https://www.youtube.com/watch?v=iSOfkw_YyOU
- **Abhishek.Veeramalla — Day-4 | Best VPC explanation | VPC explained in 30 mins**
  - Link: https://www.youtube.com/watch?v=P8g7Z4NYk3Q

---

## Build

Building upon **VPC-A (Development VPC CIDR)** created in Exercise 1:

```text
Internet
   │
   ▼
Internet Gateway (IGW-A)
   │
   ├── Public Subnet B (Public Subnet B CIDR) ──► NAT Gateway A (with Elastic IP)
   │                                                     │
   │                                                     ▼
   └── Private Route Table ◄─────────────────────────────┘
            │
            ├── Private Subnet A (Private Subnet A CIDR)
            └── Private Subnet B (Private Subnet B CIDR)
```

### Step 1 — Create and Attach Internet Gateway
1. Open **VPC > Internet Gateways**.
2. Create an Internet Gateway named `IGW-A`.
3. Select `IGW-A`, click **Actions > Attach to VPC**, and attach it to `VPC-A`.

### Step 2 — Configure Public Route Table
1. Select `Public-Route-Table`.
2. Select the **Routes** tab and click **Edit routes**.
3. Add route:
   - **Destination:** `0.0.0.0/0`
   - **Target:** `Internet Gateway` (`IGW-A`)
4. Confirm `Public-Subnet-A` and `Public-Subnet-B` are associated with this route table.

### Step 3 — Create NAT Gateway A (Canonical Architecture)
1. Navigate to **VPC > NAT Gateways**.
2. Click **Create NAT Gateway**:
   - **Name:** `NAT-Gateway-A`
   - **Subnet:** Select `Public-Subnet-B` (`Public Subnet B CIDR`)
   - **Connectivity type:** Public
   - **Elastic IP allocation ID:** Click **Allocate Elastic IP** to allocate a public Elastic IP.
3. Wait until `NAT-Gateway-A` status turns to `Available`.

### Step 4 — Configure Private Route Table
1. Select `Private-Route-Table`.
2. Select **Routes > Edit routes**.
3. Add route:
   - **Destination:** `0.0.0.0/0`
   - **Target:** `NAT Gateway` (`NAT-Gateway-A`)
4. Confirm `Private-Subnet-A` and `Private-Subnet-B` are associated with `Private-Route-Table`.
5. **Do NOT** route private subnets directly to the Internet Gateway.

### Step 5 — Launch Test Instances to Verify Connectivity
1. Launch `Bastion-EC2` in `Public-Subnet-A` with **Auto-assign Public IP** enabled.
2. Launch `App-EC2` in `Private-Subnet-A` with **Auto-assign Public IP** disabled.

---

## Scenario-Based Verification & Troubleshooting

### Scenario 1 — Public Instance Cannot Reach Internet
- **Symptom:** `Bastion-EC2` in `Public-Subnet-A` cannot reach `google.com` or download updates.
- **Investigation Step-by-Step:**
  1. **Public IP:** Check if `Bastion-EC2` has a Public IPv4 address assigned.
  2. **Subnet Route Table:** Verify `Public-Subnet-A` is associated with `Public-Route-Table`.
  3. **IGW Route:** Inspect `Public-Route-Table` to confirm `0.0.0.0/0 → IGW-A` exists.
  4. **IGW Attachment:** Verify `IGW-A` is in `Attached` state to `VPC-A`.
  5. **Security Group:** Verify outbound rules allow HTTP/HTTPS or SSH traffic.
  6. **NACL:** Verify Network ACL permits outbound/inbound traffic on ephemeral ports.
- **Verification:** Once missing route or public IP is fixed, run `curl -I https://aws.amazon.com` from `Bastion-EC2` to confirm HTTP responses.
- **Reasoning:** A subnet is only effectively public if its route table directs internet-bound traffic (`0.0.0.0/0`) to an attached Internet Gateway AND the EC2 instance has a public IP address to map via 1:1 NAT at the IGW.

### Scenario 2 — Private Instance Cannot Reach Internet
- **Symptom:** `App-EC2` in `Private-Subnet-A` fails to run `sudo yum update` or curl external APIs.
- **Investigation Step-by-Step:**
  1. Inspect `Private-Subnet-A` route table association.
  2. Check if `Private-Route-Table` has `0.0.0.0/0 → NAT-Gateway-A`.
  3. Inspect `NAT-Gateway-A` status in console (must be `Available`, not `Failed` or `Deleting`).
  4. Verify `NAT-Gateway-A` resides in a **Public Subnet** (`Public-Subnet-B`).
  5. Verify `NAT-Gateway-A` has an Elastic IP assigned.
  6. Verify `Public-Subnet-B` route table contains `0.0.0.0/0 → IGW-A`.
- **Verification:** Run `curl https://ifconfig.me` from `App-EC2`. It should return the Elastic IP address assigned to `NAT-Gateway-A`.
- **Reasoning:** The private instance sends packets to NAT Gateway via its default route. NAT Gateway translates the private source IP to its public Elastic IP and forwards packets to IGW. If any segment of this chain (Private RT → NAT GW → Public RT → IGW) is broken, outbound internet access fails.

### Scenario 3 — Private Instance Accidentally Has Direct Internet Route
- **Symptom:** A developer attempts to fix outbound internet access for private instances by adding `0.0.0.0/0 → IGW-A` directly to `Private-Route-Table`.
- **Investigation:**
  1. Inspect `Private-Route-Table` and identify the route `0.0.0.0/0 → IGW-A`.
  2. Check whether private EC2 instances have public IP addresses.
- **Questions to Answer:**
  - Why is routing a private subnet directly to an IGW invalid for instances without public IPs?
  - What happens when an instance without a public IP sends a packet to the IGW?
  - Why does this ruin the intended private subnet isolation architecture?
- **Verification:** Remove `0.0.0.0/0 → IGW-A` from `Private-Route-Table` and restore `0.0.0.0/0 → NAT-Gateway-A`.
- **Reasoning:** Internet Gateways perform 1:1 NAT between an EC2 instance's private IP and its public IP. If an EC2 instance lacks a public IP, the IGW cannot perform NAT, causing internet traffic to drop even if the route exists. Furthermore, assigning public IPs to private workloads exposes them to direct inbound attack.

### Scenario 4 — NAT Gateway Created in Private Subnet
- **Symptom:** NAT Gateway is created, but private instances using it still cannot reach the internet.
- **Investigation:**
  1. Go to **VPC > NAT Gateways > NAT-Gateway-A**.
  2. Check the **Subnet** attribute of the NAT Gateway.
  3. Inspect the route table associated with that subnet.
- **Questions to Answer:**
  - If NAT Gateway is placed in `Private-Subnet-A`, does that subnet have a route to `IGW-A`?
  - Can a NAT Gateway forward traffic to the internet if its own subnet lacks an Internet Gateway route?
- **Verification:** Re-create `NAT-Gateway-A` in `Public-Subnet-B`, which has a valid default route to `IGW-A`.
- **Reasoning:** A NAT Gateway must reside in a public subnet. It relies on the public subnet's route table (`0.0.0.0/0 → IGW`) to forward source-translated packets out to the internet.

### Scenario 5 — NAT Works but Connection Still Fails
- **Symptom:** `Private-Route-Table` correctly points to `NAT-Gateway-A`, but `App-EC2` cannot reach external sites.
- **Required Systematic Troubleshooting Order:**
  ```text
  1. Route Table ──► 2. NAT Gateway Status ──► 3. NAT Subnet Route Table ──► 4. IGW Attachment ──► 5. Security Group ──► 6. NACL
  ```
- **Verification:** Follow the troubleshooting pipeline sequentially:
  1. Confirm `Private-Route-Table` has `0.0.0.0/0 → NAT-Gateway-A`.
  2. Confirm `NAT-Gateway-A` state is `Available`.
  3. Confirm `Public-Subnet-B` (NAT's subnet) has `0.0.0.0/0 → IGW-A`.
  4. Confirm `IGW-A` is attached to `VPC-A`.
  5. Confirm Security Group allows outbound traffic (default allows all).
  6. Confirm NACL permits ephemeral ports return traffic.

### Scenario 6 — NAT Gateway vs Internet Gateway Concepts
- **Symptom:** A security auditor asks: "Why do we pay for a NAT Gateway when an Internet Gateway is free?"
- **Questions to Answer:**
  - What is the fundamental functional difference between an Internet Gateway and a NAT Gateway?
  - Which device allows **bi-directional** (inbound & outbound) traffic, and which allows **outbound-only** traffic?
- **Verification:** Provide a technical comparison explaining why private backend instances require NAT for outbound-only access.
- **Reasoning:** IGW provides 1:1 bi-directional mapping (allows external internet hosts to initiate connections to public IPs). NAT Gateway provides 1:Many Network Address Translation for outbound initiation only (blocks external hosts from initiating connections to private IPs).

### Scenario 7 — High Availability & NAT Architecture Review
- **Symptom:** Availability Zone `us-east-1b` (which hosts `NAT-Gateway-A` in `Public-Subnet-B`) suffers a total outage.
- **Investigation:**
  1. Trace traffic from `Private-Subnet-A` (in `us-east-1a`). Its route points to `NAT-Gateway-A` in `us-east-1b`.
  2. Determine what happens to outbound internet traffic from `us-east-1a` when `us-east-1b` fails.
- **Questions to Answer:**
  - Why does placing a single NAT Gateway in `AZ-B` make internet access for `AZ-A` dependent on `AZ-B`?
  - How do enterprise architectures achieve high availability (e.g., deploying one NAT Gateway per AZ)?
  - What is the difference between zonal NAT Gateways and managed Regional NAT Gateway options?
  - Why does the canonical training architecture intentionally specify one NAT Gateway per VPC?
- **Verification:** Document the trade-off: 1 NAT Gateway per VPC reduces AWS hourly NAT costs during development, but creates a cross-AZ dependency. Production environments deploy 1 NAT Gateway per AZ for fault isolation.
- **Reasoning:** NAT Gateway is a zonal resource. If the AZ hosting the NAT Gateway goes down, all subnets depending on that NAT Gateway lose internet access, even if those subnets are in surviving AZs.

---

## Networking

- **Public Subnet:** A subnet whose route table contains an explicit route to an Internet Gateway (`0.0.0.0/0 → IGW`).
- **Private Subnet:** A subnet whose route table has no route to an Internet Gateway, enforcing network isolation from direct internet inbound access.
- **Internet Gateway (IGW):** A horizontally scaled, highly available VPC component that enables communication between instances in your VPC and the internet (1:1 NAT).
- **NAT Gateway:** A managed Network Address Translation service that enables instances in a private subnet to connect to the internet (outbound-only) while preventing internet-initiated connections.
- **Elastic IP (EIP):** A static IPv4 address designed for dynamic cloud computing, allocated to a NAT Gateway to represent its public footprint.
- **Default Route (`0.0.0.0/0`):** The catch-all route entry used by routers to forward any packet whose destination IP is not explicitly covered in the route table.
- **Outbound-Only Access:** Network design pattern where internal hosts can make requests out to the internet, but external hosts cannot initiate inbound traffic.
- **AZ Resilience:** Designing network pathways so that the failure of one Availability Zone does not impair traffic flow in surviving AZs.

---

## Useful Tips

- Always inspect subnet route tables first before diagnosing application or EC2 operating system network issues.
- A NAT Gateway must be created inside a **Public Subnet** with a route to an Internet Gateway.
- Private EC2 instances do not require public IP addresses to access the internet via a NAT Gateway.
- NAT Gateways do NOT make private instances directly reachable from the internet; NAT is outbound-only.
- NAT Gateways incur hourly and data-processing charges; place 1 per VPC for dev/test and 1 per AZ for high-availability production workloads.
