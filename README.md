# AWS Static Webpage Deployment

> Deployment of a static webpage on AWS using a highly available and scalable multi-AZ architecture.

---

# 1. Project Overview

## 1.1 Introduction

This project demonstrates the deployment of a **static webpage on Amazon Web Services (AWS)** using a highly available and scalable cloud architecture.

The webpage, created by **Azeez Salu**, was deployed across multiple **Amazon EC2 instances** distributed across two Availability Zones. An **Application Load Balancer (ALB)** was used to distribute incoming traffic between the web servers, while an **Auto Scaling Group (ASG)** was configured to dynamically adjust the number of EC2 instances based on application demand.

The infrastructure was built within an **Amazon VPC** containing public and private subnets, with **NAT Gateways** providing outbound internet connectivity for resources in the private subnets.

## 1.2 Project Objectives

The main objectives of this project were to:

* Deploy a static webpage on AWS.
* Design a highly available infrastructure across multiple Availability Zones.
* Deploy EC2 web servers in private subnets.
* Distribute incoming traffic using an Application Load Balancer.
* Implement Auto Scaling for dynamic compute capacity.
* Configure secure network communication using Security Groups.
* Provide outbound internet access to private instances using NAT Gateways.
* Configure DNS resolution using Amazon Route 53.
* Gain practical experience designing AWS cloud infrastructure.

## 1.3 Scope

This project covers the design and deployment of the AWS infrastructure required to host and serve a static webpage.

The implementation includes networking, compute, load balancing, Auto Scaling, DNS, and security configuration.

The project does not include a database layer because the application is a static webpage and does not require persistent application data.

---

# 2. Architecture

## 2.1 Architecture Diagram

<img width="1275" height="691" alt="Screenshot (407)" src="https://github.com/user-attachments/assets/91d578fe-5580-4004-bd14-4d8b674120c6" />

### Architecture Overview

The infrastructure was deployed inside an Amazon VPC spanning **two Availability Zones**.

The VPC contains:

* Two public subnets
* Two private application subnets

The **Application Load Balancer** and **NAT Gateways** are located in the public subnets, while the **EC2 web servers** are deployed in the private subnets.

An **Internet Gateway** provides internet connectivity to the public subnets. The NAT Gateways allow private EC2 instances to initiate outbound internet connections without requiring the instances to be directly accessible from the internet.

Amazon Route 53 was configured for DNS resolution.

## 2.2 Architecture Flow

```text
                         INTERNET
                            │
                            ▼
                       Amazon Route 53
                            │
                            ▼
                Application Load Balancer
                     /              \
                    /                \
                   ▼                  ▼
          Private Subnet AZ1   Private Subnet AZ2
                   │                  │
                   ▼                  ▼
              EC2 Web Server     EC2 Web Server
                   │                  │
                   └────────┬─────────┘
                            │
                     Auto Scaling Group
                            │
                     Outbound Requests
                            │
                   ┌────────┴────────┐
                   ▼                 ▼
             NAT Gateway AZ1   NAT Gateway AZ2
                   │                 │
                   └────────┬────────┘
                            ▼
                     Internet Gateway
                            │
                            ▼
                         INTERNET
```

---

# 3. Networking Configuration

## 3.1 Amazon VPC

An Amazon VPC named **dev-vpc** was created to provide an isolated networking environment for the application.

### Configuration

* **VPC Name:** `dev-vpc`
* **CIDR Block:** `10.0.0.0/16`
* **Region:** `us-east-1`
* **Availability Zones:** `us-east-1a`, `us-east-1b`
* **DNS Hostnames:** Enabled

The VPC provides the foundation for the entire AWS infrastructure and allows the network resources to be logically separated and controlled.

---

## 3.2 Public Subnets

Two public subnets were created across the two Availability Zones.

| Subnet              | Availability Zone | CIDR          |
| ------------------- | ----------------- | ------------- |
| `public-subnet-az1` | `us-east-1a`      | `10.0.0.0/24` |
| `public-subnet-az2` | `us-east-1b`      | `10.0.1.0/24` |

The public subnets were configured to automatically assign IPv4 addresses to resources that require public connectivity.

The **Application Load Balancer** and **NAT Gateways** were deployed in these subnets.

---

## 3.3 Private Application Subnets

Two private application subnets were created for the EC2 web servers.

| Subnet                   | Availability Zone | CIDR          |
| ------------------------ | ----------------- | ------------- |
| `private-app-subnet-az1` | `us-east-1a`      | `10.0.2.0/24` |
| `private-app-subnet-az2` | `us-east-1b`      | `10.0.3.0/24` |

The EC2 instances hosting the webpage were deployed in these private subnets.

Placing the web servers in private subnets reduces their direct exposure to inbound internet traffic. Incoming application traffic is instead handled by the Application Load Balancer.

---

# 4. Internet Connectivity

## 4.1 Internet Gateway

An **Internet Gateway (IGW)** was created and attached to the `dev-vpc`.

The Internet Gateway provides internet connectivity for resources in the public subnets.

The public route table was configured with the following default route:

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

This allows resources in the public subnets to communicate with the internet.

---

## 4.2 NAT Gateways

Two NAT Gateways were created to provide outbound internet connectivity for resources in the private subnets.

| NAT Gateway     | Subnet              | Availability Zone |
| --------------- | ------------------- | ----------------- |
| NAT Gateway AZ1 | `public-subnet-az1` | `us-east-1a`      |
| NAT Gateway AZ2 | `public-subnet-az2` | `us-east-1b`      |

Each NAT Gateway was allocated an **Elastic IP address**.

The private route tables were configured so that:

```text
Private Subnet AZ1 → NAT Gateway AZ1
Private Subnet AZ2 → NAT Gateway AZ2
```

This design provides separate outbound paths for the private subnets and avoids relying on a single NAT Gateway for both Availability Zones.

---

# 5. Route Tables

## 5.1 Public Route Table

A public route table was created and associated with both public subnets.

### Default Route

```text
Destination: 0.0.0.0/0
Target: Internet Gateway
```

This makes the associated subnets public by providing a route to the internet through the Internet Gateway.

## 5.2 Private Route Tables

Separate private route tables were configured for the private application subnets.

### Private AZ1

```text
Private Subnet AZ1
        │
        ▼
NAT Gateway AZ1
        │
        ▼
Internet Gateway
        │
        ▼
Internet
```

### Private AZ2

```text
Private Subnet AZ2
        │
        ▼
NAT Gateway AZ2
        │
        ▼
Internet Gateway
        │
        ▼
Internet
```

The NAT Gateways allow private EC2 instances to initiate outbound connections while preventing direct inbound internet connections to those instances.

---

# 6. Security Groups

Security Groups were used as virtual firewalls to control network traffic to the AWS resources.

Three Security Groups were created:

## 6.1 ALB Security Group

The ALB Security Group controls traffic reaching the Application Load Balancer.

It was configured to allow application traffic through the required web ports.

## 6.2 SSH Security Group

A separate SSH Security Group was created to control SSH access to the web servers.

This separates administrative access from normal application traffic.

## 6.3 Web Server Security Group

The web server Security Group was configured to allow:

* HTTP traffic from the ALB Security Group
* HTTPS traffic from the ALB Security Group
* SSH traffic from the SSH Security Group

This configuration prevents the web servers from accepting normal web traffic directly from arbitrary internet sources.

### Security Group Flow

```text
Internet
   │
   ▼
ALB Security Group
   │
   │ HTTP / HTTPS
   ▼
Web Server Security Group
   │
   ▼
EC2 Web Servers
```

> **Security Note:** Security Groups should be configured with the most restrictive rules practical for the application's requirements.

---

# 7. Compute Configuration

## 7.1 Amazon EC2

Amazon EC2 was used to host the static webpage.

The instances were launched using **Amazon Linux 2023** and deployed across the two private application subnets.

### Deployment

```text
Availability Zone 1
└── private-app-subnet-az1
    └── EC2 Web Server

Availability Zone 2
└── private-app-subnet-az2
    └── EC2 Web Server
```

Deploying the web servers across two Availability Zones provides redundancy and reduces dependence on a single Availability Zone.

---

# 8. Web Server Configuration

## 8.1 Apache HTTP Server

The webpage was hosted using the **Apache HTTP Server (`httpd`)**.

The required packages were installed on the Amazon Linux 2023 instances using:

```bash
dnf update -y

dnf install -y httpd unzip wget
```

The installed packages were required for the deployment process:

* `httpd` — web server used to serve the webpage
* `wget` — used to download the application files
* `unzip` — used to extract the downloaded ZIP archive

---

# 9. Application Deployment

The webpage source code was obtained from the GitHub repository of **Azeez Salu**.

The application was downloaded directly onto the EC2 instance.

The deployment directory was:

```text
/var/www/html
```

The webpage files were then copied into the Apache web root.

### Deployment Process

1. Update the Amazon Linux system.
2. Install Apache, `wget`, and `unzip`.
3. Navigate to `/var/www/html`.
4. Download the webpage source code.
5. Extract the downloaded archive.
6. Copy the webpage files into the Apache web root.
7. Remove the temporary files.
8. Enable Apache to start automatically.
9. Start the Apache service.

---

# 10. Deployment Automation

## 10.1 Bash Script

A Bash script was used to automate the installation and configuration of the web server.

```bash
#!/bin/bash

dnf update -y

dnf install -y httpd unzip wget

cd /var/www/html

wget https://github.com/azeezsalu/jupiter/archive/refs/heads/main.zip

unzip main.zip

cp -r jupiter-main/* /var/www/html/

rm -rf jupiter-main main.zip

systemctl enable httpd

systemctl start httpd
```

## 10.2 What the Script Does

### Step 1 — Update the System

```bash
dnf update -y
```

Updates the installed packages on the Amazon Linux instance.

### Step 2 — Install Dependencies

```bash
dnf install -y httpd unzip wget
```

Installs the Apache web server and the utilities required to download and extract the webpage.

### Step 3 — Move to the Web Root

```bash
cd /var/www/html
```

Changes the working directory to Apache's default web root.

### Step 4 — Download the Application

```bash
wget https://github.com/azeezsalu/jupiter/archive/refs/heads/main.zip
```

Downloads the webpage source code from GitHub.

### Step 5 — Extract the Application

```bash
unzip main.zip
```

Extracts the downloaded ZIP archive.

### Step 6 — Deploy the Files

```bash
cp -r jupiter-main/* /var/www/html/
```

Copies the webpage files into the Apache web root.

### Step 7 — Remove Temporary Files

```bash
rm -rf jupiter-main main.zip
```

Removes the extracted source directory and downloaded archive after deployment.

### Step 8 — Start Apache

```bash
systemctl enable httpd
systemctl start httpd
```

Configures Apache to start automatically and starts the service immediately.

## 10.3 Why Automation Was Used

Using a Bash script makes the deployment process more consistent and repeatable.

Instead of manually installing packages and configuring Apache on every EC2 instance, the same script can be used to configure each web server.

This becomes particularly useful when working with multiple instances or an Auto Scaling environment.

---

# 11. Application Load Balancer

## 11.1 ALB Configuration

An **Application Load Balancer** was deployed across the two public subnets.

### Configuration

* **Load Balancer:** Application Load Balancer
* **Scheme:** Internet-facing
* **VPC:** `dev-vpc`
* **Subnets:** `public-subnet-az1`, `public-subnet-az2`
* **Listener:** HTTP
* **Target:** EC2 web servers

The ALB acts as the entry point for users accessing the application.

```text
User
 │
 ▼
Application Load Balancer
 │
 ├───────────────┐
 ▼               ▼
EC2 AZ1         EC2 AZ2
```

This prevents users from having to connect directly to individual EC2 instances.

---

# 12. Target Group

A target group was created to register the EC2 web servers with the Application Load Balancer.

The two EC2 instances were added as targets.

### Health Check

The target group's health check was configured to recognize the following HTTP response codes as successful:

```text
200
301
302
```

These response codes allow the ALB to determine whether the web servers are available to receive traffic.

If a target fails its health checks, the ALB can stop routing requests to that unhealthy target.

---

# 13. Auto Scaling

## 13.1 Auto Scaling Group

An **Auto Scaling Group (ASG)** was configured to manage the EC2 web servers.

The instances were distributed across the two private application subnets in separate Availability Zones.

The ASG allows the infrastructure to dynamically adjust compute capacity according to configured scaling requirements.

## 13.2 Scaling Behaviour

When application demand increases, the Auto Scaling Group can launch additional EC2 instances.

When demand decreases, it can terminate unnecessary instances.

The overall architecture therefore becomes:

```text
                 Auto Scaling Group
                         │
            ┌────────────┼────────────┐
            ▼            ▼            ▼
         EC2 AZ1      EC2 AZ2      New EC2
                                      │
                              When demand increases
```

## 13.3 Integration with the ALB

The Auto Scaling Group works together with the Application Load Balancer.

When new instances are launched, they can be registered with the target group so that the ALB can route traffic to healthy instances.

This allows the application layer to scale without requiring users to know which EC2 instance is serving their request.

---

# 14. DNS Configuration

## 14.1 Amazon Route 53

**Amazon Route 53** was configured to provide DNS resolution for the application.

Route 53 allows users to access the application using a domain name instead of directly accessing the ALB's DNS name.

The traffic flow is therefore:

```text
User
 │
 ▼
Domain Name
 │
 ▼
Route 53
 │
 ▼
Application Load Balancer
 │
 ▼
EC2 Web Servers
```

---

# 15. Testing and Validation

Testing was performed to verify that the different components of the architecture worked together correctly.

## 15.1 EC2 Web Server Test

The Apache web server was tested on the EC2 instances to confirm that the webpage files were being served correctly.

## 15.2 Target Group Test

The EC2 instances were registered with the target group and their health status was monitored.

The configured health checks used:

```text
200
301
302
```

as successful response codes.

## 15.3 Load Balancer Test

After the ALB became **Active**, its DNS name was copied into a web browser.

The webpage was successfully displayed through the ALB.

This confirmed that:

```text
Internet
   ↓
ALB
   ↓
Target Group
   ↓
EC2 Web Server
   ↓
Apache
   ↓
Static Webpage
```

was functioning correctly.

## 15.4 Multi-AZ Test

The web servers were distributed across:

* `us-east-1a`
* `us-east-1b`

This configuration provides redundancy across Availability Zones.

If one Availability Zone experiences an issue, the other Availability Zone can continue serving the application, provided healthy targets remain available.

---

# 16. Challenges Encountered

## 16.1 Deploying Web Servers in Private Subnets

One important networking consideration was that the EC2 instances were placed in private subnets and therefore did not have direct internet access.

### Solution

NAT Gateways were deployed in the public subnets and private route tables were configured to route outbound traffic through the appropriate NAT Gateway.

This allowed the EC2 instances to download the required application files and packages without exposing them directly to inbound internet traffic.

### What I Learned

I gained a better understanding of the difference between **public and private subnets**, and how NAT Gateways provide outbound connectivity for private resources.

---

## 16.2 Configuring ALB-to-EC2 Communication

The web servers needed to receive traffic from the Application Load Balancer without being directly exposed to the internet.

### Solution

Separate Security Groups were created for the ALB and web servers.

The web server Security Group allowed HTTP/HTTPS traffic from the ALB Security Group.

### What I Learned

This demonstrated how Security Groups can be used to control communication between different layers of an AWS architecture.

---

## 16.3 Automating Web Server Deployment

Manually installing Apache and copying the webpage files on multiple instances would make the deployment process repetitive.

### Solution

A Bash script was created to automate system updates, package installation, application download, file deployment, and Apache configuration.

### What I Learned

I gained practical experience using Bash to automate server configuration and application deployment.

---

# 17. Cost Considerations

Several AWS resources used in this project can generate charges depending on usage and AWS pricing.

These include:

* Amazon EC2
* Application Load Balancer
* NAT Gateways
* Elastic IP addresses
* Amazon Route 53

Particular attention should be paid to **NAT Gateways**, as they can generate charges while running.

For learning environments, resources that are no longer required should be stopped or deleted where appropriate.

> **Important:** Before deleting resources, verify their dependencies to avoid leaving unnecessary resources running or disrupting other components.

---

# 18. Results

The static webpage was successfully deployed on AWS using a multi-AZ architecture.

The final implementation achieved the following:

* A dedicated VPC was created.
* Public and private subnets were distributed across two Availability Zones.
* EC2 web servers were deployed in private subnets.
* Apache was installed and configured to serve the webpage.
* An Application Load Balancer distributed traffic to the web servers.
* NAT Gateways provided outbound internet connectivity for private instances.
* Auto Scaling was configured to manage EC2 capacity.
* Route 53 was configured for DNS resolution.
* Security Groups controlled communication between the application components.

The project successfully demonstrated how a simple static webpage can be deployed using a more resilient cloud architecture rather than relying on a single publicly exposed EC2 instance.

---

# 19. Lessons Learned

## 19.1 AWS Networking

I gained practical experience with:

* Amazon VPC
* CIDR blocks
* Public and private subnets
* Route tables
* Internet Gateways
* NAT Gateways
* Availability Zones

## 19.2 AWS Compute

I learned how to:

* Launch Amazon EC2 instances.
* Configure Amazon Linux 2023.
* Install and configure Apache.
* Deploy application files to a Linux web server.
* Use Auto Scaling to manage EC2 capacity.

## 19.3 Load Balancing

I gained practical experience with:

* Creating an Application Load Balancer.
* Creating and configuring target groups.
* Registering EC2 instances as targets.
* Configuring health checks.
* Routing traffic to healthy instances.

## 19.4 Security

I gained a better understanding of:

* Security Groups
* Private subnet architecture
* Controlled communication between AWS resources
* Separating administrative SSH access from application traffic

## 19.5 Automation

The Bash deployment script provided practical experience automating:

* Package installation
* Web server configuration
* Application downloads
* File deployment
* Service management

---

# 20. Future Improvements

The following improvements could be made to further strengthen the architecture:

* Configure **HTTPS** using AWS Certificate Manager.
* Add **Amazon CloudFront** for content delivery and caching.
* Implement **AWS WAF** for additional web application protection.
* Improve monitoring using **Amazon CloudWatch alarms**.
* Implement centralized logging.
* Use **Infrastructure as Code** with Terraform or AWS CloudFormation.
* Integrate a **CI/CD pipeline** for automated application deployment.
* Improve the Auto Scaling configuration with more advanced scaling policies.
* Implement automated testing as part of the deployment process.

These improvements were not part of the current implementation but represent possible steps toward a more production-ready architecture.

---

# 21. AWS Services Used

| AWS Service               | Purpose                                                       |
| ------------------------- | ------------------------------------------------------------- |
| Amazon VPC                | Provides the isolated networking environment                  |
| Amazon EC2                | Hosts the static webpage                                      |
| Application Load Balancer | Distributes incoming traffic                                  |
| Auto Scaling              | Dynamically manages EC2 capacity                              |
| Internet Gateway          | Provides internet connectivity to public resources            |
| NAT Gateway               | Provides outbound internet connectivity for private instances |
| Amazon Route 53           | Provides DNS resolution                                       |
| Security Groups           | Controls network traffic to AWS resources                     |
| Elastic IP                | Provides static public IP addresses for NAT Gateways          |

---

# 22. Final Architecture Summary

```text
                              INTERNET
                                  │
                                  ▼
                           Amazon Route 53
                                  │
                                  ▼
                     Application Load Balancer
                          /               \
                         /                 \
                        ▼                   ▼
               Public Subnet AZ1    Public Subnet AZ2
                        │                   │
                        │                   │
                  NAT Gateway AZ1      NAT Gateway AZ2
                        │                   │
                        ▼                   ▼
               Private Subnet AZ1   Private Subnet AZ2
                        │                   │
                        ▼                   ▼
                   EC2 Web Server       EC2 Web Server
                        │                   │
                        └─────────┬─────────┘
                                  │
                         Auto Scaling Group
                                  │
                                  ▼
                         Static Web Application
```

---

# 23. Conclusion

This project provided practical experience in designing, deploying, testing, and documenting AWS infrastructure for a static web application.

Rather than deploying the webpage on a single publicly accessible EC2 instance, the architecture used **multiple Availability Zones, private application subnets, an Application Load Balancer, NAT Gateways, Auto Scaling, Security Groups, and Route 53**.

The project strengthened my understanding of **AWS networking, EC2, Linux server administration, Apache, load balancing, Auto Scaling, DNS, security, and cloud architecture design**.

It also demonstrated how individual AWS services can be combined to create an infrastructure that is more **available, scalable, redundant, and controlled** than a basic single-server deployment.
