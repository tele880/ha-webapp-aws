# High-Availability Web Application on AWS

**Production-grade, fault-tolerant web application built on Amazon Web Services**  
Multi-AZ Architecture | Auto Scaling | Application Load Balancer | RDS Multi-AZ

> **Region:** us-east-1 (N. Virginia) | **Status:** ✅ Deployed & Live | **Uptime Target:** 99.99%

---

## Architecture Overview

This project demonstrates the design and deployment of a production-grade HA web application targeting high-availability use cases such as Electronic Health Record (EHR) systems. All infrastructure was built manually through the AWS Management Console.

![Architecture Diagram]()

> Users reach the application through the Internet Gateway and Application Load Balancer. Traffic is distributed to EC2 instances in private subnets across both AZs. NAT Gateways allow private instances outbound internet access. The RDS primary in AZ-A replicates synchronously to a standby in AZ-B for automatic failover.

---

## Architecture Specifications

| Specification            | Value                                    |
|--------------------------|------------------------------------------|
| VPC CIDR                 | 10.0.0.0/16                              |
| Availability Zones       | us-east-1a, us-east-1b                   |
| EC2 Instances (desired)  | 2 — one per AZ                           |
| EC2 Instances (max)      | 4                                        |
| Instance Type            | t2.micro (Amazon Linux 2023)             |
| Web Server               | Apache httpd (auto-installed via User Data)|
| Database                 | RDS MySQL 8.4 — Multi-AZ                 |
| Load Balancer            | Application Load Balancer (internet-facing)|

---

## AWS Services Used

| Service                  | Purpose                                                              |
|--------------------------|----------------------------------------------------------------------|
| Amazon VPC               | Network isolation and subnet segmentation across two AZs             |
| EC2 Auto Scaling         | Automatic instance management, replacement, and capacity scaling      |
| Application Load Balancer| Layer 7 HTTP traffic distribution across healthy EC2 instances        |
| Amazon EC2               | Web server compute — Amazon Linux 2023 + Apache httpd                |
| Amazon RDS               | Managed MySQL 8.4 with automatic Multi-AZ failover                   |
| NAT Gateway              | Outbound internet access for private subnet instances                 |
| Internet Gateway         | Inbound internet access entry point for the VPC                      |
| Security Groups          | Stateful firewall rules at the ALB, EC2, and RDS tiers               |
| EC2 Launch Templates     | Standardized instance config with User Data bootstrap script          |
| Target Groups            | Health monitoring and ALB routing destination                         |

---

## Networking

### VPC & Subnets

| Subnet Name       | CIDR           | Type    | AZ         | Hosts                     |
|-------------------|----------------|---------|------------|---------------------------|
| Public-Subnet-AZ-A | 10.0.1.0/24   | Public  | us-east-1a | ALB, NAT Gateway          |
| Public-Subnet-AZ-B | 10.0.2.0/24   | Public  | us-east-1b | ALB, NAT Gateway (HA)     |
| Private-Subnet-AZ-A| 10.0.3.0/24   | Private | us-east-1a | EC2 Instance, RDS Primary |
| Private-Subnet-AZ-B| 10.0.4.0/24   | Private | us-east-1b | EC2 Instance, RDS Standby |

### Route Tables

| Route Table   | Destination  | Target       | Associated Subnets           |
|---------------|--------------|--------------|------------------------------|
| Public-RT     | 0.0.0.0/0    | IGW          | Public-AZ-A, Public-AZ-B     |
| Public-RT     | 10.0.0.0/16  | local        | Public-AZ-A, Public-AZ-B     |
| Private-RT    | 0.0.0.0/0    | NAT Gateway  | Private-AZ-A, Private-AZ-B   |
| Private-RT    | 10.0.0.0/16  | local        | Private-AZ-A, Private-AZ-B   |

---

## Security (Defense-in-Depth)

Security is implemented via layered security group chaining. No resource is directly reachable from the internet except through the ALB.

### Traffic Flow

```
[Internet] → ALB-SG (HTTP 80, HTTPS 443) → ALB
                                             │
                                             ▼
                               EC2-SG (port 80 from ALB-SG only)
                                             │
                                             ▼
                               RDS-SG (MySQL 3306 from EC2-SG only)
```

### Security Groups

| Security Group | Inbound Rule                          | Purpose                          |
|----------------|---------------------------------------|----------------------------------|
| ALB-SG         | HTTP (80) + HTTPS (443) from 0.0.0.0/0| Accept public web traffic        |
| EC2-SG         | TCP 80 from ALB-SG only               | Only ALB can reach EC2           |
| RDS-SG         | MySQL (3306) from EC2-SG only         | Only EC2 can reach the database  |

---

## Compute

### Launch Template (HA-WebApp-LT)

The Launch Template standardizes every EC2 instance launched by the Auto Scaling Group — AMI, instance type, security group, and a User Data script that installs Apache on first boot.

```bash
# scripts/user-data.sh
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>HA-WebApp - $(hostname -f)</h1>" > /var/www/html/index.html
```

### Auto Scaling Group (HA-WebApp-ASG)

| Setting                  | Value                                               |
|--------------------------|-----------------------------------------------------|
| Desired / Min / Max      | 2 / 2 / 4                                           |
| Health Check Types       | EC2 + ELB                                           |
| Health Check Grace Period| 300 seconds                                         |
| Availability Zones       | us-east-1a (Private-AZ-A), us-east-1b (Private-AZ-B)|

---

## Load Balancer

| Setting        | Value                                                         |
|----------------|---------------------------------------------------------------|
| Name           | HA-WebApp-ALB                                                 |
| Type           | Application Load Balancer (Layer 7)                           |
| Scheme         | Internet-facing                                               |
| Subnets        | Public-Subnet-AZ-A, Public-Subnet-AZ-B                        |
| Listener       | HTTP:80 → Forward to HA-WebApp-TG (100%)                      |
| Target Group   | HA-WebApp-TG — Instance type, port 80, HTTP health checks     |

---

## Database

| Setting        | Value                                                         |
|----------------|---------------------------------------------------------------|
| DB Identifier  | ha-webapp-db                                                  |
| Engine         | MySQL Community 8.4                                           |
| Instance Class | db.t3.micro                                                   |
| Storage        | 20 GB General Purpose SSD (gp2)                               |
| Multi-AZ       | ✅ Yes — synchronous standby replica in second AZ             |
| Primary AZ     | us-east-1b                                                    |
| Security Group | RDS-SG — MySQL (3306) inbound from EC2-SG only               |

---

## Proof of Functionality

Load balancing was confirmed live by refreshing the ALB DNS endpoint and observing hostname changes:

- **Request 1:** `HA-WebApp - ip-10-0-3-46.ec2.internal` → Private-Subnet-AZ-A (us-east-1a)
- **Request 2:** `HA-WebApp - ip-10-0-4-84.ec2.internal` → Private-Subnet-AZ-B (us-east-1b)

The IP difference (10.0.3.x vs 10.0.4.x) confirms both AZs are actively serving traffic in real time.

### Infrastructure Health Summary

| Component          | Detail                                          | Status       |
|--------------------|-------------------------------------------------|--------------|
| VPC                | HA-WebApp-VPC — 10.0.0.0/16, us-east-1         | ✅ Active    |
| Internet Gateway   | HA-WebApp-IGW — Attached to VPC                 | ✅ Attached  |
| NAT Gateway        | HA-WebApp-NAT — Public-Subnet-AZ-A              | ✅ Available |
| Load Balancer      | HA-WebApp-ALB — Internet-facing, HTTP:80         | ✅ Active    |
| Target Group       | HA-WebApp-TG — 2/2 Healthy instances            | ✅ Healthy   |
| EC2 Instance 1     | us-east-1b — 2/2 status checks                  | ✅ Running   |
| EC2 Instance 2     | us-east-1a — 2/2 status checks                  | ✅ Running   |
| Auto Scaling Group | HA-WebApp-ASG — Desired=2, At capacity          | ✅ Active    |
| RDS Database       | ha-webapp-db — MySQL Multi-AZ, us-east-1b       | ✅ Available |

---

## Challenges & Lessons Learned

### Challenge 1 — Blackhole NAT Gateway Route

**Problem:** All EC2 instances showed Unhealthy in the Target Group. Apache never installed.

**Root Cause:** A previous NAT Gateway had been deleted and recreated. The Private-RT still pointed `0.0.0.0/0` to the old deleted gateway (shown as **Blackhole**). Private subnet instances silently failed to run `yum install httpd`.

**Resolution:**
1. Identified Blackhole route in VPC → Route Tables → Private-RT
2. Updated `0.0.0.0/0` target to the new NAT Gateway ID
3. Terminated stale instances — ASG relaunched fresh ones with working internet
4. New instances installed Apache successfully and health checks passed

### Challenge 2 — ALB in Wrong Subnets (502 Bad Gateway)

**Problem:** After health checks turned green, the ALB DNS name returned `502 Bad Gateway`.

**Root Cause:** The ALB had been associated with private subnets instead of public subnets. An internet-facing ALB must reside in public subnets with a route to the Internet Gateway.

**Resolution:**
1. Identified public vs private subnets by checking Route Tables (IGW route present = public)
2. Edited ALB Network Mapping to use Public-Subnet-AZ-A and Public-Subnet-AZ-B
3. 502 errors resolved immediately

### Key Takeaways

- NAT Gateway route tables are critical — a Blackhole route silently breaks all private subnet outbound connectivity
- Internet-facing ALBs must always be placed in **public subnets** with IGW routes
- User Data scripts are the correct pattern to bootstrap software on Auto Scaling instances
- Health check grace periods (300s) prevent premature termination during initialization
- Security group chaining (ALB-SG as source for EC2-SG) is more secure than CIDR-based rules

---

## Potential Enhancements

- **HTTPS / TLS** — AWS Certificate Manager + HTTPS listener + HTTP-to-HTTPS redirect (required for HIPAA compliance)
- **Amazon CloudWatch** — Alarms on CPU, ALB 5xx error rate, and RDS connection count
- **Auto Scaling Policies** — Target tracking on CPU utilization for dynamic scale-in/out
- **AWS WAF** — Web Application Firewall for SQL injection and XSS protection
- **Infrastructure as Code** — CloudFormation or Terraform for repeatable, version-controlled deployments
- **CI/CD Pipeline** — AWS CodePipeline + CodeDeploy for automated zero-downtime deployments

---

## Repository Structure

```
ha-webapp-aws/
├── README.md               # This file
├── scripts/
│   └── user-data.sh        # EC2 bootstrap script (Apache install)
├── architecture/
│   └── architecture-diagram.png  # Multi-AZ architecture diagram
├── docs/
│   └── HA-WebApp-Portfolio-Documentation.pdf  # Full project documentation
└── .gitignore
```

---

## Author

**Bernard Tetteh**  
AWS Certified Solutions Architect – Associate | AWS Certified Cloud Practitioner  
[GitHub](https://github.com/tele880) | [Portfolio](https://bernardtetteh.cloud)


