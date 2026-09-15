## How I Deployed an HTML Website on AWS

I recently deployed an HTML website on AWS by following a course by Azeez Salu but rather than just following the tutorial and creating my own version of the project, I took time to understand the purpose of each AWS service used and how the individual components worked together to deliver a highly available and secure architecture. Let's explore this together.

### Reference Architecture

The architecture used for this project is based on the reference architecture below:
<img width="463" height="408" alt="image" src="https://github.com/user-attachments/assets/c8cfbd53-5e3f-4079-b45b-d2f3784a840e" />


### AWS Services Used

* **Amazon VPC** — Network isolation and subnet organization
* **Public and Private Subnets** — Separation of internet-facing and application resources
* **Security Groups** — Control of inbound and outbound traffic
* **Amazon EC2** — Hosting the web servers
* **NAT Gateway** — Providing outbound internet access to private instances
* **Application Load Balancer (ALB)** — Distributing incoming traffic across web servers
* **Amazon Route 53** — DNS management and routing the domain to the ALB
* **AWS Certificate Manager (ACM)** — Provisioning the SSL/TLS certificate
* **EC2 Auto Scaling** — Maintaining and scaling the web server fleet

### How the Components Worked Together

The website was deployed using a **two-tier VPC architecture** consisting of public and private application subnets distributed across multiple Availability Zones. This design provides network isolation and improves availability by avoiding dependence on a single Availability Zone.

### 1. VPC and Subnets

The VPC was divided into public and private application subnets across multiple Availability Zones.

A key distinction between a **public subnet** and a **private subnet** is how their route tables provide connectivity to the internet:

* **Public subnet:** A public subnet has a route in its route table to an **Internet Gateway (IGW)**. Resources deployed in a public subnet can communicate directly with the internet when they have the necessary public addressing and security-group rules. For example, the Application Load Balancer and NAT Gateways were deployed in the public subnets.

* **Private subnet:** A private subnet does not have a direct route to an Internet Gateway. Instead, its route table can send outbound internet traffic to a **NAT Gateway**, which is deployed in a public subnet. Resources in the private subnet do not have public IP addresses and cannot receive unsolicited inbound connections directly from the internet. However, they can initiate outbound connections through the NAT Gateway, such as downloading packages, pulling files from GitHub, or accessing external APIs.

In this project:

* **Public subnets** hosted the **Application Load Balancer and NAT Gateways**.
* **Private application subnets** hosted the **EC2 web servers**.

This separation ensured that the web servers were not directly exposed to the internet, while still allowing them to access external resources when required.

The subnets were distributed across multiple Availability Zones to improve availability and provide redundancy.

#### 2. Internet Gateway

An **Internet Gateway (IGW)** was attached to the VPC to provide internet connectivity for resources in the public subnets.

The public route table contained a default route (`0.0.0.0/0`) pointing to the Internet Gateway, allowing resources such as the ALB and NAT Gateways to communicate with the internet.

#### 3. Route Tables

Route tables controlled how traffic was routed within and outside the VPC.

* The **public route table** was associated with the public subnets and contained a route to the Internet Gateway.
* The **private route table** was associated with the private application subnets and contained a default route to the NAT Gateway.

This allowed the private EC2 instances to initiate outbound internet connections without exposing them directly to inbound internet traffic.

#### 4. Security Groups

Security Groups acted as **stateful virtual firewalls** controlling inbound and outbound traffic to the AWS resources.

The security groups were layered to control which components could communicate with one another:

* The **ALB security group** allowed inbound HTTP/HTTPS traffic from the internet.
* The **web server security group** allowed application traffic only from the **ALB security group**, rather than allowing direct application traffic from the internet.
* An **SSH security group** was configured to allow SSH access from my IP address.
* The **web server security group** also allowed SSH traffic from the **SSH security group**. This allowed the SSH access rule to be managed through a security-group reference rather than directly adding my IP address to the web server security group.

This layered approach restricted direct access to the web servers while allowing controlled administrative access when required.

#### 5. EC2 Web Servers

The HTML website was hosted on **Amazon EC2 instances** located in the private application subnets.

Because these instances did not have public IP addresses, they could not be accessed directly from the internet. Instead, they received application traffic from the Application Load Balancer.

The website files were maintained in a GitHub repository and downloaded onto the EC2 instances during deployment.

#### 6. NAT Gateway

The **NAT Gateways** were deployed in the public subnets and referenced by the private subnet route tables.

They allowed the private EC2 instances to initiate outbound connections to the internet, such as downloading packages, updates, or the website files from GitHub, while preventing unsolicited inbound connections from the internet.

#### 7. Auto Scaling Group

An **Auto Scaling Group (ASG)** was used to manage the EC2 web server instances.

The ASG launched instances using a predefined launch template and distributed them across multiple Availability Zones. It could also:

* Maintain the desired number of instances
* Launch additional instances when scaling conditions were met
* Terminate instances when scaling down
* Replace unhealthy instances based on health checks

This helped improve the availability and resilience of the application by ensuring that the web server fleet was not dependent on a single EC2 instance.

#### 8. Application Load Balancer

The **Application Load Balancer (ALB)** was deployed across the public subnets in multiple Availability Zones.

It served as the public entry point for the website and distributed incoming requests across healthy EC2 instances registered in its target group.

This meant users did not need to connect directly to individual EC2 instances.

#### 9. Route 53

**Amazon Route 53** was used to manage the DNS record for my domain.

The domain was configured with a record that directed traffic to the Application Load Balancer. This allowed users to access the website using the domain name instead of the ALB's automatically generated DNS name.

#### 10. AWS Certificate Manager

**AWS Certificate Manager (ACM)** was used to provision an SSL/TLS certificate for the domain.

The certificate was attached to the ALB's HTTPS listener, allowing the ALB to terminate HTTPS connections from users.

> **Note:** In this implementation, HTTPS was terminated at the ALB. Traffic between the ALB and the EC2 instances used HTTP. Therefore, TLS encryption protected the connection between the user's browser and the ALB, but not the ALB-to-EC2 leg. End-to-end encryption would require configuring HTTPS between the ALB and the web servers as well.

### Traffic Flow

A typical request to the website followed this path:

**User → Route 53 → Application Load Balancer → EC2 Web Server**

For outbound requests from the private EC2 instances, the path was:

**EC2 Web Server → Private Route Table → NAT Gateway → Internet Gateway → Internet**

This architecture separates the public-facing entry point from the web servers while allowing the private instances to access external resources when required.
