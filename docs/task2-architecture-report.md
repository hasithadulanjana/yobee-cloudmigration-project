# Task 2: Designing a Virtualization Architecture

**Student:** Hasitha
**Course:** Cloud Solutions Engineering
**Institution:** Yoobee College of Creative Innovation
**Date:** March 2026

---

## 1. Introduction

This report documents the virtualization architecture designed and deployed for Yoobee College of Creative Innovation's migration to Amazon Web Services (AWS). Acting as a Cloud Solutions Engineer, the objective was to design a secure, scalable, and highly available cloud infrastructure that supports the college's Learning Management System (LMS), faculty applications, and student data. The architecture follows AWS Well-Architected Framework principles, applying multi-tier design, strict network isolation, least-privilege access controls, and infrastructure automation through CloudFormation and a GitHub Actions deployment pipeline.

---

## 2. Architecture Overview

### Multi-Tier Design

The architecture separates concerns across three logical tiers deployed within a dedicated Virtual Private Cloud (VPC) using the CIDR block `10.0.0.0/16`:

- **Presentation Tier** — An internet-facing Application Load Balancer (ALB) deployed in two public subnets (`10.0.1.0/24` in AZ-a, `10.0.2.0/24` in AZ-b). This is the only layer directly reachable from the internet.
- **Application Tier** — EC2 instances hosting the LMS (Linux) and faculty applications (Windows) deployed in private subnets (`10.0.11.0/24` in AZ-a, `10.0.12.0/24` in AZ-b), unreachable from the public internet.
- **Data Tier** — An RDS MySQL instance deployed within the same private subnets, accessible only from the application tier.

### Public and Private Subnet Separation

Public subnets host only the ALB and NAT Gateways. All compute and data resources reside in private subnets, which have no direct route to the internet. Outbound internet access for private instances (for software updates and AWS API calls) is provided through two NAT Gateways — one per Availability Zone — ensuring that a single AZ failure does not disrupt outbound connectivity. This design dramatically reduces the attack surface by ensuring no EC2 instance or database has a publicly routable IP address.

---

## 3. Architecture Diagram

The architecture diagram is provided as a separate file (see `task2-architecture-diagram.pdf`). The following is a detailed textual description of the diagram's components and connections.

### Components and Layout

**Public Subnets (AZ-a: 10.0.1.0/24 / AZ-b: 10.0.2.0/24):**
- Internet Gateway — attached to the VPC, providing the entry point for all inbound internet traffic
- Application Load Balancer — spans both public subnets, receives traffic on port 80
- NAT Gateway (AZ-a) — allows private subnet AZ-a instances to initiate outbound connections
- NAT Gateway (AZ-b) — allows private subnet AZ-b instances to initiate outbound connections

**Private Subnets (AZ-a: 10.0.11.0/24 / AZ-b: 10.0.12.0/24):**
- LMSServer1 (AZ-a) and LMSServer2 (AZ-b) — Amazon Linux 2, t3.micro, managed by Auto Scaling Group
- FacultyServer1 (AZ-a) and FacultyServer2 (AZ-b) — Windows Server 2022, t3.micro
- RDS MySQL 8.0 — primary in one AZ, with subnet group spanning both AZs

**Supporting Services (logical, not subnet-bound):**
- S3 Bucket (yobee-lms-assets) — accessed via IAM role from EC2 instances
- CloudWatch — collects metrics and logs from all compute and database resources
- SNS Topic — delivers alarm notifications to admin@yoobee.ac.nz
- IAM Roles and Groups — applied to EC2 instances and human users

### Traffic Flow

1. A user's browser sends an HTTP request to the ALB's DNS name over the internet.
2. The Internet Gateway routes the request into the VPC to the ALB.
3. The ALB evaluates listener rules: requests matching `/faculty/*` are forwarded to the FacultyApp target group (FacultyServer1/2); all other requests are forwarded to the LMS target group (LMSServer1/2).
4. EC2 instances process the request, querying the RDS MySQL database as required.
5. For S3 access (course materials, assets), EC2 instances use their attached IAM role — no credentials are stored on the instance.
6. Outbound traffic from EC2 instances (e.g., package updates) exits via the NAT Gateway in the same AZ, then through the Internet Gateway.

### Drawing in Lucidchart

When reproducing this diagram in Lucidchart, use the AWS shape library (search "AWS19" for the 2019 icon set). Key icons: VPC boundary (dashed rectangle), Internet Gateway (horizontal cylinder), ALB (Application Load Balancer icon), EC2 instances (instance icon, differentiated by colour for Linux vs Windows), RDS (database icon), S3 (bucket icon), NAT Gateway (gateway icon), CloudWatch (telescope icon), and SNS (notification icon). Group public-subnet components inside a lighter blue rectangle and private-subnet components inside a darker blue rectangle, with the VPC as the outer boundary.

---

## 4. AWS Services and Justification

| Service | Role in Architecture | Why Chosen | How It Meets Requirements |
|---|---|---|---|
| **VPC** | Network isolation boundary for all resources | Full control over IP addressing, routing, and security | Provides private network space; no resource is exposed unless explicitly configured |
| **EC2** | Hosts LMS (Linux) and faculty apps (Windows) | Flexible OS choice; supports both Linux and Windows workloads | t3.micro balances cost and performance for a college-scale deployment |
| **RDS MySQL** | Managed relational database for student records | Fully managed; automated backups, patching, and Multi-AZ support | Eliminates database administration overhead; ensures data durability |
| **S3** | Stores course materials and LMS assets | 99.999999999% durability; pay-per-use; no capacity planning needed | Lifecycle policies manage cost; versioning protects against accidental deletion |
| **ALB** | Routes HTTP traffic to the correct application tier | Layer 7 path-based routing; health checks; native Auto Scaling integration | Enables a single DNS endpoint to serve both LMS and faculty apps |
| **Auto Scaling** | Adjusts LMS server count based on demand | Prevents over-provisioning during low-traffic periods | CPU-based scaling at 70% threshold; scales from 0 to 6 instances |
| **IAM** | Controls all access to AWS resources | Native AWS service; no additional cost; deep integration | Least-privilege roles and groups enforce separation of duties |
| **CloudWatch** | Monitoring, alerting, and log aggregation | Native integration with all AWS services used | Alarms on CPU, RDS connections, storage, and ALB errors trigger SNS notifications |

---

## 5. Security Considerations

### VPC Network Isolation

All application and database resources are placed in private subnets with no inbound route from the internet. Network ACLs are applied to the private subnets as a stateless, subnet-level firewall, providing an additional layer of traffic filtering beyond Security Groups.

### Security Groups

Four Security Groups enforce least-privilege network access:

- **ALBSecurityGroup** — allows inbound HTTP (port 80) from any IP (`0.0.0.0/0`). Outbound restricted to the LMS and Faculty Security Groups.
- **LMSSecurityGroup** — allows inbound traffic only from ALBSecurityGroup on the application port. Allows outbound to RDSSecurityGroup on port 3306 and to the internet via NAT for updates.
- **FacultyAppSecurityGroup** — same pattern as LMSSecurityGroup, restricted to faculty application traffic.
- **RDSSecurityGroup** — allows inbound MySQL (port 3306) only from LMSSecurityGroup and FacultyAppSecurityGroup. No inbound from the internet.

### IAM Least Privilege

Every EC2 instance uses an IAM role rather than embedded credentials. Human users are assigned to IAM Groups with the minimum permissions required for their role (detailed in Section 6). MFA is enforced for the ITAdminGroup, which has elevated EC2/RDS/VPC access.

### S3 Security

The `yobee-lms-assets` bucket has server-side encryption (AES-256) enabled, public access fully blocked at the bucket and account level, and versioning enabled to allow recovery from accidental deletion or overwrite. Lifecycle policies automatically transition older object versions to cheaper storage classes, then expire them.

### Encryption in Transit

All traffic between the ALB and clients uses HTTP in the current deployment, with the architecture designed to add HTTPS (ACM certificate) as a straightforward future enhancement. EC2-to-RDS and EC2-to-S3 traffic remains within the AWS network and can be enforced to use TLS at the application and SDK level.

---

## 6. IAM Roles and Permissions

| Role / Group | Who Uses It | Permissions | Justification |
|---|---|---|---|
| **LMSAppRole** | LMS EC2 instances (attached at launch) | S3 read/write on yobee-lms-assets; CloudWatch Logs write; SSM Parameter Store read | LMS needs to serve and store course assets; SSM avoids hardcoded secrets |
| **FacultyAppRole** | Faculty EC2 instances (attached at launch) | S3 read on yobee-lms-assets; CloudWatch Logs write; SSM read | Faculty app reads shared assets but does not write; reduces risk of accidental data modification |
| **DBAccessRole** | Services or scripts requiring DB metadata | RDS Describe actions only (no data access) | Allows monitoring scripts to check RDS status without exposing data |
| **StudentCognitoRole** | Student-facing application (via Cognito) | S3 read on `course-materials/` prefix only | Students can access published course content; cannot access other buckets or prefixes |
| **CloudAdminGroup** | Cloud administrators | Full AWS access (AdministratorAccess) | Required to deploy and manage the full infrastructure stack |
| **ITAdminGroup** | IT department staff | EC2, RDS, VPC, CloudWatch, SSM — MFA required | Operational access for infrastructure management without billing or IAM permissions |
| **FacultyGroup** | Teaching staff | Read-only across services; SSM Session Manager to faculty servers | Allows faculty to view infrastructure and connect to their servers without SSH key management |
| **StudentGroup** | Students (AWS console access) | S3 GetObject on `course-materials/` prefix only | Strictly scoped to prevent any unintended access to compute or database resources |
| **ReadOnlyOpsGroup** | Monitoring/support staff | CloudWatch read; EC2 Describe | Enables dashboard viewing and incident response without permission to modify resources |

---

## 7. Infrastructure as Code

The entire Yoobee AWS environment is defined as CloudFormation templates, split across seven stacks to reflect logical separation of concerns:

1. **network** — VPC, subnets, Internet Gateway, NAT Gateways, route tables, NACLs
2. **storage** — S3 bucket, bucket policy, lifecycle rules
3. **iam** — All IAM roles, instance profiles, and user groups
4. **database** — RDS MySQL instance, DB subnet group, parameter group
5. **compute** — EC2 launch templates, Auto Scaling Group, LMS and faculty instances
6. **alb** — Application Load Balancer, target groups, listener rules
7. **monitoring** — CloudWatch alarms, SNS topic, CloudWatch dashboards

A GitHub Actions pipeline automates deployments: on every push to the `main` branch, the workflow validates all templates with `cfn-lint`, then deploys each stack in dependency order using `aws cloudformation deploy`. This ensures that infrastructure changes are peer-reviewed (via pull request), version-controlled, and produce a full audit trail in GitHub. The use of CloudFormation eliminates configuration drift — the state of the environment is always derivable from the repository. If a deployment fails, CloudFormation automatically rolls back to the previous stable state, reducing the risk of partial deployments.

---

## 8. Justification for Selected Configurations

**t3.micro EC2 instances** were selected for both LMS and faculty servers because the college's current workload does not require large compute capacity, and t3.micro instances fall within the AWS Free Tier for new accounts. The burstable CPU model of the T3 family suits web application workloads that have intermittent CPU spikes rather than sustained high utilisation.

**Multi-AZ subnet layout** with resources spread across AZ-a and AZ-b ensures that a failure of a single AWS Availability Zone does not take down the application. The ALB, NAT Gateways, LMS servers, and faculty servers all have presence in both AZs, so traffic is automatically redistributed if one AZ becomes unavailable.

**Private subnets for all compute and data resources** ensure that no EC2 instance or database is reachable from the internet without explicitly traversing the ALB or a VPN. This is the most impactful single security decision in the architecture.

**ALB over NLB** was chosen because the workloads are HTTP-based web applications. The ALB operates at Layer 7, enabling path-based routing (`/faculty/*` to faculty servers, default to LMS), native health checks at the HTTP level, and straightforward integration with AWS WAF if needed in future. An NLB (Layer 4) would not support path-based routing.

**RDS MySQL over a self-managed database** eliminates the need for the IT team to manage database patching, backup schedules, or storage scaling. AWS handles automated daily backups (7-day retention), minor version patching, and storage autoscaling, allowing the team to focus on application delivery.

**S3 for course materials** provides eleven-nines of durability with no operational overhead. Unlike an NFS share or EBS volume, S3 scales automatically with no capacity planning, is accessible from any EC2 instance regardless of AZ, and costs a fraction of equivalent block or file storage for predominantly read workloads.

---

## 9. Conclusion

The architecture designed for Yoobee College of Creative Innovation demonstrates how AWS services can be combined to produce a secure, scalable, and cost-efficient cloud environment suited to an educational institution. By applying a three-tier model with strict subnet isolation, path-based load balancing, managed database and storage services, and automated infrastructure deployment, the solution addresses the college's requirements for availability, security, and operational simplicity. The use of CloudFormation and GitHub Actions ensures the environment is reproducible and auditable, reducing the risk of human error and providing a foundation that the IT team can confidently maintain and extend as Yoobee's cloud adoption matures.
