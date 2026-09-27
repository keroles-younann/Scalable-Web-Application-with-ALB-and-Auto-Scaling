# Scalable-Web-Application-with-ALB-and-Auto-Scaling

## Overview:
EC2-based, highly available web application on AWS: public/private subnets across two Availability Zones, Application Load Balancer + Auto Scaling Group for the compute tier, Multi-AZ RDS for the database tier, CloudFront + WAF at the edge, Route 53 for DNS/health checks, Systems Manager Session Manager for bastion-free access, and CloudWatch/SNS for monitoring and alerting.

## Solution Architecture
<img width="1151" height="811" alt="AWS drawio" src="https://github.com/user-attachments/assets/1e8e45ec-8bc8-4018-8757-ac7cea3f27dc" />

## Architecture components

| Layer | Service | Purpose |
|---|---|---|
| DNS | Amazon Route 53 | Alias record pointing to the ALB; health check drives failover/monitoring |
| Edge / CDN | Amazon CloudFront | Caches static assets at edge locations, reduces latency to origin (ALB) |
| Edge security | AWS WAF (Web ACL) | `AWSManagedRulesCommonRuleSet` for OWASP Top 10 coverage + a rate-based rule, associated with the CloudFront distribution |
| Networking | VPC (`10.0.0.0/16`) | 2 AZs, each with a public subnet, an app-tier private subnet, and a DB-tier private subnet |
| Networking | Internet Gateway | Public subnet egress/ingress |
| Networking | NAT Gateway (one per AZ) | Outbound internet access for private subnets, without a public IP on instances |
| Networking | Security Groups + NACLs | SG on ALB allows 443 from internet; SG on EC2 allows only ALB SG on the app port; SG on RDS allows only the EC2 SG on 3306/5432; NACLs as a subnet-level backstop |
| Compute | Application Load Balancer | Layer 7 routing, target group with health checks, spans both AZs' public subnets |
| Compute | Auto Scaling Group + Launch Template | EC2 instances in private app subnets, target-tracking scaling policy (e.g., ALB request count per target or CPU) |
| Database | Amazon RDS (Multi-AZ) | Primary in AZ-A, synchronous standby in AZ-B, automated failover |
| Access | AWS Systems Manager – Session Manager | Shell access to EC2 instances with no bastion host, no inbound SSH rule, and a CloudTrail-logged session |
| Identity | AWS IAM | EC2 instance profile (least-privilege: CloudWatch + SSM only), separate admin group for human operators |
| Monitoring | Amazon CloudWatch | Alarms on CPU, ALB 5xx rate, target health, RDS failover events |
| Notification | Amazon SNS | Delivers CloudWatch alarm notifications to the administrator by email/SMS |

## Design rationale

A few choices in this diagram are worth calling out explicitly, because two commonly-shared reference diagrams for this exact project (see Acknowledgements) leave them out, and the assignment brief lists them as required services:

- **WAF is not optional here.** The brief pairs it with ALB explicitly ("ALB + WAF: Layer 7 routing, WAF rules for OWASP Top 10"). It's implemented as a Web ACL with the AWS-managed Core Rule Set, which covers OWASP Top 10-class attacks (SQLi, XSS, etc.) without hand-written rules.
- **WAF is attached to CloudFront, not directly to the ALB.** Blocking malicious traffic at the CloudFront edge means it never reaches the origin at all, which is the pattern AWS's own WAF guidance recommends when a CDN is already in front of the origin. If you don't deploy CloudFront, attach the Web ACL to the ALB instead (`scope = REGIONAL`); if you do deploy CloudFront, its Web ACL must be created in `us-east-1` regardless of your app's region (`scope = CLOUDFRONT`).
- **No jump host / bastion.** Systems Manager Session Manager replaces the traditional public-subnet bastion: instances need no inbound SSH rule and no public IP, and every session is logged. This is a named learning outcome in the brief ("Systems Manager Session Manager as a bastion-free access alternative") — a diagram that still shows a jump host in a public subnet contradicts that outcome.
- **Route 53 carries a health check, not just an alias record.** An alias with no health check can't drive Route 53–level failover or feed CloudWatch; the brief specifically asks for both.
- **RDS standby is a synchronous replica, not a read replica.** Multi-AZ failover and read replicas are different RDS features; this diagram is the former (HA/failover), matching "automated failover" in the brief.

## Deployment (manual console walkthrough)

1. **VPC & networking**
   - Create VPC `10.0.0.0/16`.
   - Create 2 public subnets, 2 app-tier private subnets, 2 db-tier private subnets (one of each per AZ).
   - Attach an Internet Gateway; add a NAT Gateway per AZ (or one shared NAT Gateway if optimizing for cost, at the expense of cross-AZ resilience for egress).
   - Configure route tables: public subnets → IGW; private subnets → NAT Gateway in their own AZ.
   - Define NACLs and Security Groups per the table above.

2. **Database tier**
   - Launch an RDS instance (MySQL or PostgreSQL) with Multi-AZ enabled, in the db-tier private subnets, using a DB subnet group spanning both AZs.

3. **Compute tier**
   - Create a Launch Template (AMI, instance type, user-data bootstrap, the EC2 instance profile from step 5).
   - Create an Auto Scaling Group across both app-tier private subnets using the Launch Template; set min/desired/max (e.g., 2/2/4) and a target-tracking scaling policy.

4. **Load balancing**
   - Create an Application Load Balancer in the public subnets, listener on 443 (redirect 80→443), target group pointing at the ASG with health checks on your app's health endpoint.

5. **IAM**
   - Create an EC2 instance role/profile with `AmazonSSMManagedInstanceCore` and a scoped CloudWatch agent policy — nothing else.
   - Create an admin IAM group with SSM Session Manager start/terminate permissions for operators; no SSH key pair is required.

6. **Edge**
   - Create a CloudFront distribution with the ALB as its origin; cache behaviors for static paths (e.g., `/static/*`) with a longer TTL, and a pass-through/short-TTL behavior for dynamic routes.
   - Create a WAF Web ACL (`AWSManagedRulesCommonRuleSet` + a rate-based rule) in `us-east-1` and associate it with the CloudFront distribution.

7. **DNS**
   - In Route 53, create an alias record pointing at the CloudFront distribution (or the ALB, if you're not fronting with CloudFront), and a health check on the ALB target group.

8. **Monitoring**
   - Create CloudWatch alarms: ASG average CPU, ALB `HTTPCode_Target_5XX_Count`, ALB `UnHealthyHostCount`, RDS `FailoverState`/`FreeStorageSpace`.
   - Create an SNS topic, subscribe the administrator's email/SMS, and set each alarm's action to publish to that topic.

## Cost considerations

- One NAT Gateway per AZ is the resilient choice but doubles NAT cost; a single shared NAT Gateway is a valid cost/availability trade-off for a course project — document whichever you pick and why.
- CloudFront + WAF both carry small fixed and per-request costs; free-tier usage is generally sufficient to demonstrate the pattern without meaningful spend.
- RDS Multi-AZ roughly doubles the database instance cost versus single-AZ — that's the cost of the automated failover the brief asks for.
