---
Title: Deploying a Fault-Tolerant Web Architecture on AWS with CloudFormation
Date: 2026-11-22
Category: Cloud
Excerpt: Automating a Multi-AZ high-availability VPC, Application Load Balancer, Auto Scaling Group, and least-privilege Security Groups on AWS using Infrastructure as Code.
---

Running a web application on a single EC2 instance is fine for testing, but in production, it represents a single point of failure. If that instance crashes or an Availability Zone goes down, the entire service goes offline.

To address this, I designed and automated a multi-AZ, high-availability web architecture on AWS (`us-east-1`) using **AWS CloudFormation**. Instead of manual console configuration, the entire infrastructure—from custom VPC networking to load balancing and auto-scaling—is defined as Infrastructure as Code (IaC).

---

## Architecture Overview

The infrastructure resides within a custom VPC (`10.0.0.0/16`) spanning two Availability Zones (`us-east-1a` and `us-east-1b`):

* **Networking Layer:** Two public subnets (`10.0.1.0/24` and `10.0.2.0/24`) routed to an Internet Gateway (`HA-Project-IGW`), alongside two isolated private subnets (`10.0.3.0/24` and `10.0.4.0/24`) reserved for backend tiers.
* **Load Balancing:** An internet-facing Application Load Balancer (`HA-Application-Load-Balancer`) listening on HTTP port 80 and routing traffic across healthy targets.
* **Compute & Scaling:** An Auto Scaling Group maintaining a minimum of 2 Apache web servers distributed across both AZs, with automatic scaling capacity up to 4 instances.

![AWS High Availability Architecture Diagram](images/AWS-HA/AWS-HA-ARCHDesign.png)

### Design Decision: Subnet Placement & Cost Optimization
In strict corporate setups, web servers are often placed in private subnets behind a NAT Gateway. Since NAT Gateways incur continuous hourly charges, I intentionally placed the EC2 instances in public subnets for this deployment. This allowed the `UserData` startup script to fetch Apache updates directly via the Internet Gateway without extra NAT overhead, while keeping the private subnets provisioned for future backend components. Even within public subnets, direct external access to the servers remains completely restricted at the firewall layer.

---

## Security Posture: Strict Least Privilege

To prevent exposing web servers directly to the internet (`0.0.0.0/0`), I implemented Security Group Chaining. The Application Load Balancer accepts inbound HTTP traffic, while the EC2 Security Group strictly limits HTTP access to traffic coming exclusively from the load balancer's Security Group ID.

```yaml
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP traffic from everywhere to Load Balancer
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: ALB-Security-Group

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP traffic from ALB Security Group only
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup
      Tags:
        - Key: Name
          Value: Web-Server-SG
```

This strict boundary guarantees that direct connections to instance public IP addresses are dropped automatically.

---

## Provisioning & Self-Healing Automation

To eliminate manual software installation on launch, I combined an EC2 Launch Template with an Auto Scaling Group. The `UserData` script automatically bootstraps Apache (`httpd`) and deploys the default web page on startup, while the ASG enforces instance health and availability across both Availability Zones.

```yaml
  MyLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: WebServerTemplate
      LaunchTemplateData:
        ImageId: ami-0a3c3a20c09d6f377
        InstanceType: t2.micro
        SecurityGroupIds:
          - !Ref InstanceSecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd
            echo "<h1>Welcome to Our AWS Team Project</h1>" > /var/www/html/index.html

  MyAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      VPCZoneIdentifier:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2
      LaunchTemplate:
        LaunchTemplateId: !Ref MyLaunchTemplate
        Version: !GetAtt MyLaunchTemplate.LatestVersionNumber
      MinSize: '2'
      MaxSize: '4'
      TargetGroupARNs:
        - !Ref MyTargetGroup
      Tags:
        - Key: Name
          Value: Team-Web-Server
          PropagateAtLaunch: true
```

---

## Deployment & Console Verification

Once CloudFormation finished provisioning the stack (`CREATE_COMPLETE`), I verified the functionality of each component directly inside the AWS Console.

### 1. Custom VPC & Routing Architecture
The VPC resource map confirms `HA-Project-VPC` (`10.0.0.0/16`) is fully active with four subnets across `us-east-1a` and `us-east-1b`, correctly linked to the Internet Gateway.

![VPC Resource Map](images/AWS-HA/VPC.png)

### 2. Multi-AZ EC2 Deployment
The Auto Scaling Group successfully launched two healthy `t2.micro` instances (`Team-Web-Server`), distributed evenly across both Availability Zones.

![EC2 Instances Status](images/AWS-HA/EC2.png)

### 3. Target Group Health Checks
Health check logs show both registered web instances reporting a `Healthy` status on HTTP port 80.

![Target Group Health Status](images/AWS-HA/TargetGroup.png)

### 4. Application Load Balancer Status
The Application Load Balancer (`HA-Application-Load-Balancer`) shows an `Active` status with configured HTTP listener rules.

![Active Application Load Balancer](images/AWS-HA/ALP.png)

### 5. End-to-End Live Verification
Accessing the ALB's DNS endpoint via browser successfully routes requests to the backend instances and renders the Apache web application.

![Live Web Application via ALB Endpoint](images/AWS-HA/WebPage.png)

---

## Project Repository

The full CloudFormation template and implementation steps are hosted on GitHub:  
[github.com/RanaAbdelsalamm/AWS-HA-Architecture](https://github.com/RanaAbdelsalamm/AWS-HA-Architecture)