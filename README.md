Here's the README file based on your provided structure and the diagram:

# AWS Architecture: Scalable Web Application

## High-level overview of the project:

This project outlines a robust and scalable web application architecture deployed on Amazon Web Services (AWS). It leverages a multi-tier design to ensure high availability, fault tolerance, and automatic scaling capabilities. The architecture separates concerns into distinct layers: a public-facing web tier, a private application tier, and a highly available database layer, all designed for optimal performance and resilience.

## Key AWS Services Used:

This is an architecture for a scalable web application on AWS using the following services:

* **EC2:** Launch instances for the web app and application tiers.
* **Application Load Balancer (ALB):** Distributes traffic across multiple instances for both web and application tiers.
* **Auto Scaling Group (ASG):** Ensures instances scale based on demand for both web and application tiers.
* **Amazon RDS (Optional):** Backend database (MySQL/PostgreSQL) with Multi-AZ for high availability.
* **IAM:** Role-based access for AWS services and administrative access.
* **CloudWatch & SNS:** Monitor performance, log events, and send alerts.
* **Route 53:** DNS resolution and traffic management, integrated with CloudFront (CDN).
* **CloudFront:** Content Delivery Network (CDN) for improved performance and security.
* **VPC:** Isolated network environment for AWS resources, including public and private subnets.
* **NAT Gateway:** Allows instances in private subnets to initiate outbound connections to the internet.
* **AWS WAF:** Web Application Firewall to protect web applications from common web exploits.

## Some Notes

* **AWS NAT Gateways** are provided in case the EC2 instances need to be updated or patched.
* **Jump-Host EC2 instance** is acting like a bastion host in case the admin needed to access any private instance for any reason.
* **AWS WAF (Web Application Firewall)** is implemented on the public-facing `ALP (Web-tier)` to provide protection against common web exploits.
* **CloudFront** is used in conjunction with Route 53 for DNS resolution and content delivery, improving latency and security.

## IAM Roles:

* **ALB:** `AWSElasticLoadBalancingServiceRolePolicy`
* **ASG:** `AWSServiceRoleForAutoScaling`
* **CloudWatch:** `CloudWatchReadOnlyAccess`
* **Administrator (User/Group):** `Systems Manager (Session Manager)` for secure remote access, and an `Attached Policy: Administrator` for full administrative control.

## Security Groups:

| Inbound traffic                     | Outbound traffic                            |
| :---------------------------------- | :------------------------------------------ |
| **ALB web tier** |                                             |
| `0.0.0.0/0 :80 , 443`               | `IP range of web EC2 subnets :80 ,443`      |
| **Web tier EC2** |                                             |
| `IP of Web ALB : 80 , 443`          | `IP of App ALB:5000`                        |
| **ALB app tier** |                                             |
| `IP range of the web EC2 Subnets :5000` | `IP range of App EC2 subnets : 5000`        |
| **App tier EC2** |                                             |
| `IP range of the web tier ALB:5000` | `IP range of the RDS subnets : 3306`        |
| **RDS (MYSQL)** |                                             |
| `IP range of the app tier subnets :3306` | `-` (typically, only inbound is explicitly managed from app tier) |

## Project Flow:

1.  An external user types the DNS name of the website.
2.  **AWS Route 53** resolves the DNS record and, integrated with **CloudFront**, redirects the request to the **AWS WAF** (Web-tier Public ALB).
3.  The request passes through the **VPC Internet Gateway** to access the public **ALB (Web Tier)**.
4.  The **ALB** redistributes the traffic to the **EC2 instances** under the **ASG** attached to the ALB on the web tier.
5.  The **ASG** reacts to the load on the application:
    * If there is a high load, the ASG scales up the number of instances.
    * If the load drops down, the ASG scales the instances down again.
6.  When the web tier has a request for the application tier, it sends the request to the **ALB (App Tier) endpoint**, which is private because it’s not meant to be accessed from the internet; only the web tier needs to access the app tier.
7.  Then, the **ALB (App Tier)** performs load balancing among the App Tier EC2 instances. The **ASG** for the App Tier functions similarly, scaling instances based on demand.
8.  When the Application needs to query data from the database, it queries this data from **RDS (primary)**.
9.  In case of the failure of the primary RDS instance, **Multi-AZ** is enabled to recover with minimum downtime, automatically failing over to the standby DB.
10. **CloudWatch** monitors the application status for the following cases:
    * Events on the ASG (Launch, Terminate, Fail to launch, Fail to terminate).
    * RDS failover and switching to the standby DB.
11. **CloudWatch** publishes a message to the **SNS topic**.
12. The Administrator is subscribed to the **SNS topic**, and they receive an email/SMS about the events, ensuring timely alerts and operational awareness.

