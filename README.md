# AWS Custom VPC Architecture Lab (TCS-HDFC Project)

This repository documents the step-by-step implementation of a highly secure, multi-tier AWS Virtual Private Cloud (VPC) network architecture. The setup isolates public-facing web servers from private application servers while maintaining secure outbound internet access for patches using a NAT Gateway.

---

## Architectural Overview

The deployment provisions an isolated networking environment mapped to the following specifications:
* **Total CIDR Block:** `10.0.0.0/16` (65,536 available IP addresses)
* **Public Subnet Tier:** `10.0.1.0/24` (Hosts public web servers with Elastic/Public IPs)
* **Private Subnet Tier:** `10.0.2.0/24` (Hosts isolated backend application instances)

```text
[Internet]
    │
    ▼
[ Internet Gateway (IGW) ]
    │
    ├────────► [ Public Subnet: 10.0.1.0/24 ] ──► [ Web Server (TCS-Web-Server-1) ]
    │                                                     │
    ▼                                                     ▼
[ NAT Gateway ] ◄───────────────────────────────── [ Private Subnet: 10.0.2.0/24 ]
                                                          │
                                                          ▼
                                              [ App Server (TCS-AS-1) ]
```

---

## Lab Implementation Steps

### Phase 1: Core Networking Foundation
1. **Create the Custom VPC:**
   * Navigated to VPC Dashboard -> **Create VPC**.
   * Named the VPC `TCS-VPC`.
   * Assigned the primary IPv4 CIDR Block: `10.0.0.0/16`.
2. **Subnet Provisioning:**
   * **Public Subnet:** Created `TCS-Pub-SUB` with CIDR `10.0.1.0/24` in the Mumbai region (`ap-south-1`).
   * **Private Subnet:** Created `TCS-Pvt-SUB` with CIDR `10.0.2.0/24`.
3. **Internet Gateway (IGW) Setup:**
   * Created a new Internet Gateway named `TCS-IGW`.
   * Selected `TCS-IGW` -> Actions -> **Attach to VPC** -> Selected `TCS-VPC`.

### Phase 2: Routing Table Configurations
1. **Public Route Table (`TCS-Pub-RT`):**
   * Created a new route table and attached it to `TCS-VPC`.
   * **Edit Routes:** Added an explicit route to the internet (`0.0.0.0/0`) targeting `TCS-IGW` as the Destination.
   * **Subnet Associations:** Explicitly attached `TCS-Pub-SUB` to this table.
2. **Private Route Table (`TCS-Pvt-RT`):**
   * Created a separate route table for the private tier.
   * **Subnet Associations:** Explicitly attached `TCS-Pvt-SUB` to this table.

### Phase 3: Perimeter Firewall Security Groups
1. **Public Security Group (`TCS-PUB-SG`):**
   * Attached to `TCS-VPC`.
   * **Inbound Rules:** Configured to allow **All TCP** traffic for public web server accessibility.
2. **Private Security Group (`TCS-PUT-SG` / `TCS-Pvt-SG`):**
   * Attached to `TCS-VPC`.
   * **Inbound Rules:** Configured to allow **All TCP** traffic constrained strictly to communication within internal server tiers.

### Phase 4: Compute Instance Provisioning
1. **Web Server Deployment (Public Tier):**
   * Launched an EC2 instance named `TCS-web-Server-1`.
   * **Instance Type:** `t3.small` using a designated key pair (`TCS-VPC-Server-Key`).
   * **Network Settings:** Selected `TCS-VPC` and placed it within `TCS-Pub-SUB`.
   * **Public IP:** Set **Auto-assign Public IP** to **Enable**.
   * **Firewall:** Attached the `TCS-PUB-SG` security group.
2. **Application Server Deployment (Private Tier):**
   * Launched a second EC2 instance named `TCS-AS-1`.
   * **Instance Type:** `t3.small`.
   * **Network Settings:** Placed within `TCS-Pvt-SUB`.
   * **Public IP:** Set **Auto-assign Public IP** to **Disable** (keeping it completely isolated from the open web).
   * **Firewall:** Attached the `TCS-PUT-SG` security group.

### Phase 5: Outbound Private Routing via NAT Gateway
1. **NAT Gateway Creation:**
   * Navigated to VPC Dashboard -> **NAT Gateways** -> **Create NAT Gateway**.
   * Named it `TCS-NAT`.
   * **Subnet Placement:** Crucially placed it inside the **Public Subnet** (`TCS-Pub-SUB`) so it can connect outwards.
   * **Elastic IP:** Clicked **Allocate Elastic IP** to give it a static public routing address.
2. **Updating Private Routes:**
   * Returned to the private route table (`TCS-Pvt-RT`).
   * **Edit Rules:** Added a route for `0.0.0.0/0` pointing directly to the target **NAT Gateway** (`nat-gw`). This allows your private application servers to pull software updates without exposing them to incoming internet threads.

---

## Verification & Connectivity Checks
* **Static Content Check:** Verify `TCS-web-Server-1` successfully resolves web requests externally using its allocated Public IP.
* **Internal Routing Verification:** Establish an active connection to the public web server, then run network health diagnostics (`ping`) targeting the private IP of `TCS-AS-1` to validate the safe multi-tier link.
