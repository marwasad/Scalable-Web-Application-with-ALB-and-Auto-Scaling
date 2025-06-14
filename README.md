# Scalable-Web-Application-with-ALB-and-Auto-Scaling
## AWS Architecture Design for a Highly Available, Scalable, and Cost-Optimized Web Application
This architecture leverages AWS best practices to deploy a simple web application using EC2, Application Load Balancer (ALB), Auto Scaling Groups (ASG), and Amazon RDS, ensuring high availability, scalability, security, and cost optimization.
## Architecture Diagram 
Below is a high-level overview of the architecture used:
![AWS Scalable web application1](https://github.com/user-attachments/assets/65872447-972b-4328-80b9-16cfdbe148ea)
## Key Features:
-High Availability: Resources are distributed across two Availability Zones (AZs) to ensure service continuity in case of an AZ failure.

-Scalability: An Auto Scaling Group (ASG) dynamically adjusts the number of web application instances based on demand.

-Security: Segregation of resources into public and private subnets, use of security groups, and IAM roles for granular access control.

-Managed Services: Amazon RDS provides a managed, multi-AZ database backend for reliability and ease of management.

-Monitoring: AWS CloudWatch is integrated for monitoring and alerting

## Component Breakdown

-VPC: Provides isolated networking for all resources.

-Public Subnets: Host NAT Gateways, allowing outbound internet access for resources in private subnets.

-Private Subnets: Host web application EC2 instances and Amazon RDS databases, shielding them from direct internet exposure.

-Internet Gateway: Enables inbound/outbound internet connectivity for public subnets.

-Application Gateway (ALB): Distributes incoming user traffic across web application instances in multiple AZs.

-Web ASG (Auto Scaling Group): Manages EC2 instances running the web application, scaling them automatically based on load.

-Security Groups: Control inbound and outbound traffic at the instance and database levels.

-Amazon RDS: Provides a managed relational database with high availability via deployment across multiple AZs.

-IAM: Manages secure, role-based access to AWS resources.

-CloudWatch: Monitors resource health and application performance, enabling proactive alerting and operational insight.

## Step-by-Step Implementation Guide
## Phase 1 : Foundaion Setup 
#### Step 1: Create VPC and Network Infrastructure 
##### 1.Create VPC

    Go to AWS Console → VPC Dashboard

    Click "Create VPC" → Choose "VPC and more"

    Name: WebApp-VPC

    IPv4 CIDR: 10.0.0.0/16

    Number of AZs: 2 (us-east-1a, us-east-1b)

    Number of public subnets: 2

    Number of private subnets: 4 (2 for web tier, 2 for database tier)

    NAT Gateways: 1 per AZ

    VPC endpoints: None (for now)

##### 2.Verify Subnet Configuration

###### Public Subnets:

    -WebApp-Public-1A: 10.0.1.0/24 (AZ-a)

    -WebApp-Public-1B: 10.0.2.0/24 (AZ-b)

###### Private Subnets (Web Tier):

    WebApp-Private-Web-1A: 10.0.11.0/24 (AZ-a)

    WebApp-Private-Web-1B: 10.0.12.0/24 (AZ-b)

###### Private Subnets (Database Tier):

    WebApp-Private-DB-1A: 10.0.21.0/24 (AZ-a)

    WebApp-Private-DB-1B: 10.0.22.0/24 (AZ-b)

#### Step 2: Configure Security Groups
#### Application Load Balancer Security Group

Name: WebApp-ALB-SG

Inbound Rules:

- HTTP (80) from 0.0.0.0/0

- HTTPS (443) from 0.0.0.0/0

Outbound Rules:

- All traffic to 0.0.0.0/0

#### Web Application Security Group

Name: WebApp-EC2-SG

Inbound Rules:

- HTTP (80) from WebApp-ALB-SG

- SSH (22) from your IP (for management)

Outbound Rules:

- All traffic to 0.0.0.0/0

#### Database Security Group

Name: WebApp-RDS-SG

Inbound Rules:

- MySQL/Aurora (3306) from WebApp-EC2-SG

Outbound Rules:

- All traffic to 0.0.0.0/0

## Phase 3: Database Layer Setup
#### Step 3: Create RDS Database

##### Navigate to RDS Console

    -Click "Create database"

    -Choose "Standard create"

###### Database Configuration

    Engine: MySQL 8.0 (or PostgreSQL 13+)
    
    Template: Production
    
    DB Instance Identifier: webapp-database
    
    Master Username: admin
    
    Master Password: [Create strong password]
    
    DB Instance Class: db.t3.micro (for testing) / db.r5.large (production)
    
    Storage: 100 GB General Purpose SSD

##### Connectivity Settings

    VPC: WebApp-VPC
    
    DB Subnet Group: Create new (select private DB subnets)

    Public Access: No

    VPC Security Group: WebApp-RDS-SG
    
    Availability Zone: No preference (for Multi-AZ)

## Phase 4: Web Application Deployment
#### Step 4: Create Launch Template

##### Launch Template Configuration

  Name: WebApp-Launch-Template
  
  Description: Template for web application instances
  
  AMI: Amazon Linux 2023 AMI
  
  Instance Type: t3.micro (for testing) / t3.small (production)

  Key Pair: [Select existing or create new]
  
  Security Group: WebApp-EC2-SG

  IAM Instance Profile: WebApp-EC2-Role
  
 ##### User Data Script

  bash
  
 #!/bin/bash

  yum update -y

##### Install web server (Apache or Nginx)

yum install -y httpd

systemctl start httpd

systemctl enable httpd

##### Install CloudWatch agent

  wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm

  rpm -U ./amazon-cloudwatch-agent.rpm

##### Get instance metadata for identification
  
  INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
 
  AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)

##### Create simple health check page
  cat <<EOF > /var/www/html/health
 
  OK
 
  EOF

##### Create sample application page

  cat <<EOF > /var/www/html/index.html
  
  <!DOCTYPE html>
 
  <html>
 
    <head>
     
      <title>Web Application</title>
 
  </head>
 
  <body>
      
    <h1>Hello from Web Server</h1>
      
      <p>Instance ID: $INSTANCE_ID</p>
      
      <p>Availability Zone: $AZ</p>
     
      <p>Server Time: $(date)</p>
  
  </body>
 
  </html>
  
  EOF

#### Step 5: Create Auto Scaling Group
##### Auto Scaling Group Configuration

    Name: WebApp-ASG
    
    Launch Template: WebApp-Launch-Template
    
    Version: Latest
    
    VPC: WebApp-VPC
    
    Subnets: Select both private web subnets

Group Size and Scaling

    Desired Capacity: 2
    
    Minimum Capacity: 1
    
    Maximum Capacity: 6
  
Scaling Policies

  Policy 1: Scale Out
  - Policy Type: Target Tracking
  - Metric: Average CPU Utilization
  - Target Value: 60%
  - Scale-out Cooldown: 300 seconds

  Policy 2: Scale In
  - Policy Type: Target Tracking
  - Metric: Average CPU Utilization
  - Target Value: 30%
  - Scale-in Cooldown: 300 seconds

## Phase 4: Application Load Balancer Setup
#### Step 6: Create Target Group

##### Target Group Configuration

    -Name: WebApp-TG
    -Target Type: Instances
    -Protocol: HTTP
    -Port: 80
    -VPC: WebApp-VPC
    -Protocol Version: HTTP1
  
 ##### Health Check Settings

    -Health Check Path: /health or /
    -Health Check Port: Traffic port
    -Healthy Threshold: 2
    -Unhealthy Threshold: 2
    -Timeout: 5 seconds
    -Interval: 30 seconds
    -Success Codes: 200
    
#### Step 7: Create Application Load Balancer

##### Basic Configuration

    -Name: WebApp-ALB
    -Scheme: Internet-facing
    -IP Address Type: IPv4
    -VPC: WebApp-VPC
    -Mappings: Select both public subnets
    -Security Group: WebApp-ALB-SG
##### Listener Configuration
    -Protocol: HTTP
    -Port: 80
    -Default Action: Forward to WebApp-TG
##### Attaching Load Balancer to auto scaling group 

    Load Balancer: Attach to existing load balancer
    Target Group: WebApp-TG
    Health Check Type: ELB
    Health Check Grace Period: 300 seconds
    
## Phase 5: Monitoring and Alerting Setup
#### Step 8: Configure CloudWatch Monitoring

Custom CloudWatch Dashboard

    Dashboard Name: WebApp-Monitoring
   
    Widgets:
    
    - EC2 CPU Utilization
   
    - ALB Request Count
    
    - ALB Target Response Time
    
    - RDS CPU Utilization
    
    - RDS Database Connections
    
CloudWatch Alarms

    
    Alarm 1: High CPU Usage
   
    - Metric: EC2 CPU Utilization
    
    - Threshold: > 80% for 2 consecutive periods
    
    - Action: Send SNS notification
    
    Alarm 2: ALB High Latency
    
    - Metric: Target Response Time
    
    - Threshold: > 2 seconds for 2 consecutive periods
    
    - Action: Send SNS notification

    Alarm 3: Unhealthy Targets
    
    - Metric: UnHealthyHostCount
   
    - Threshold: >= 1 for 1 period
    
    - Action: Send SNS notification
    
#### Step 9: Set Up SNS Notifications

##### Create SNS Topic

  Topic Name: WebApp-Alerts
  
  Display Name: Web Application Alerts
 
  Create Subscriptions

  Protocol: Email
  
  Endpoint: your-email@domain.com
 
  Link CloudWatch Alarms to SNS
  
  Edit each CloudWatch alarm
  
  Add SNS topic as notification action

## Phase 6: Testing and Validation
#### Step 10: Test the Complete Setup

##### Test Load Balancer
  
  -Access ALB DNS name in browser
  
  -Verify traffic distribution across instances
  
  -Check target group health in AWS Console
  
##### Test Auto Scaling

bash

   #SSH into an EC2 instance (via Session Manager)
  
  sudo yum install stress -y
   
   Generate CPU load
  
  stress --cpu 8 --timeout 300s
  
  Monitor scaling events in ASG console

##### Test High Availability

  -Terminate one instance manually
  
  -Verify ASG replaces it automatically
  
  -Confirm ALB removes unhealthy targets

##### Test Database Connectivity

  -From EC2 instance
  -mysql -h [RDS-ENDPOINT] -u admin -p

#### Step 11: Performance Optimization
##### Monitor Performance Metrics

    -Review CloudWatch dashboards daily
    
    -Analyze response times and error rates
    
    -Monitor database performance metrics

##### Right-size Resources

    -Use AWS Compute Optimizer
    
    -Adjust instance types based on utilization
    
    -Optimize RDS instance class

##### Cost Optimization

    -Implement scheduled scaling
    
    -Consider Reserved Instances for steady workloads
    
    -Use Spot Instances for development/testing
    
This comprehensive implementation provides a production-ready, highly available, and scalable web application architecture following AWS best practices for security, performance, and cost optimization.
