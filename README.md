## How I Deployed an HTML Website on AWS

I recently deployed an HTML website on AWS by following a course by Azeez Salu and rather than just following the tutorial without asking questionds, I took time to understand the purpose of each AWS service used and how the individual components worked together to deliver a highly available and secure architecture. Let's explore this together.

### Reference Architecture

The architecture used for this project is based on the reference architecture below:
<img width="463" height="408" alt="image" src="https://github.com/user-attachments/assets/c8cfbd53-5e3f-4079-b45b-d2f3784a840e" />


### AWS Services Used

1. **Amazon VPC**
2. **Public and Private Subnets**
3. **Security Groups**
4. **Amazon EC2**
5. **NAT Gateway**
6. **Application Load Balancer (ALB)**
7. **Amazon Route 53**
8. **AWS Certificate Manager (ACM)**
9. **EC2 Auto Scaling**

### How the Components Worked Together

The website was deployed using a **two-tier VPC architecture** consisting of public and private application subnets distributed across multiple Availability Zones. This design provides network isolation and improves availability and fault tolerance by avoiding dependence on a single Availability Zone.

### 1. VPC and Subnets

The VPC was divided into public and private application subnets across multiple Availability Zones.

A key distinction between a **public subnet** and a **private subnet** is how their route tables provide connectivity to the internet:

* **Public subnet:** A public subnet has a route in its route table connected to an **Internet Gateway (IGW)**. Resources deployed in a public subnet can communicate directly with the internet when they have the necessary public addressing and security-group rules. For example, the Application Load Balancer and NAT Gateways were deployed in the public subnets.

* **Private subnet:** A private subnet does not have a direct route to an Internet Gateway. Instead, its route table can send outbound internet traffic to a **NAT Gateway**, which is deployed in a public subnet. Resources in the private subnet do not have public IP addresses and cannot receive unsolicited inbound connections directly from the internet. However, they can initiate outbound connections through the NAT Gateway, such as downloading packages, pulling files from GitHub, or accessing external APIs.

In this project:

* **Public subnets** hosted the **Application Load Balancer and NAT Gateways**.
* **Private application subnets** hosted the **EC2 web servers**.

This separation ensured that the web servers were not directly exposed to the internet, while still allowing them to access external resources when required.

The subnets were distributed across multiple Availability Zones to improve availability and provide redundancy.

#### 2. Internet Gateway

An **Internet Gateway (IGW)** was attached to the VPC to provide internet connectivity for resources in the public subnets.

#### 3. Route Tables

Route tables controlled how traffic was routed within and outside the VPC.

* The **public route table** was associated with the public subnets and contained a route to the Internet Gateway.
* The **private route table** was associated with the private application subnets and contained a route to the NAT Gateway.

This allowed the private EC2 instances to initiate outbound internet connections without exposing them directly to inbound internet traffic.

#### 4. Security Groups

Security Groups acted as **stateful virtual firewalls** controlling inbound and outbound traffic to the AWS resources.

The security groups were layered to control which components could communicate with one another:

* The **ALB security group** allowed inbound HTTP/HTTPS traffic from the internet.
* An **SSH security group** was configured to allow SSH access from my IP address.
* The **web server security group** allowed application traffic and SSH traffic from the **ALB security group** and **SSH security group**, respectively, rather than allowing direct application traffic from the internet.

This layered approach restricted direct access to the web servers while allowing controlled administrative access when required.

#### 5. EC2 Web Servers

The HTML website was hosted on **Amazon EC2 instances** located in the private application subnets.

Because these instances did not have public IP addresses, they could not be accessed directly from the internet. Instead, they received application traffic from the Application Load Balancer.

The website files were maintained in a GitHub repository and downloaded onto the EC2 instances during deployment.

#### 6. NAT Gateway

The **NAT Gateways** were deployed in the public subnets and referenced by the private subnet route tables.

They allowed the private EC2 instances to initiate outbound connections to the internet, such as downloading packages, updates, or the website files from GitHub, while preventing unsolicited inbound connections from the internet.

#### 7. Auto Scaling Group

An **Auto Scaling Group (ASG)** was used to dynamically create and scale our EC2 instances in the private app subnets based on incoming traffic to the application load balancer.

The ASG launched instances using a predefined launch template and distributed them across multiple Availability Zones. It helped to:

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

### Here is how we built the project

We started the building process by:

#### 1. Creating a custom 2-tier VPC with public and private subnets. The first tier contained the public subnet while the second tier contained the private app subnet which was duplicated across multiple availability zones in order to promote high availability and fault tolerance in case any availability zone experiences any outage.

In creating the VPC, we followed these steps:
##### a. On the AWS Management Console, click on the search bar and type VPC then select it
<img width="1012" height="827" alt="Screenshot (410)" src="https://github.com/user-attachments/assets/db044f27-9e9d-4594-baf8-20925193cc58" />

Then, select **Create VPC**.

<img width="1261" height="780" alt="Screenshot (411)" src="https://github.com/user-attachments/assets/62ddb04f-a107-4035-8117-0c41be5ce5e2" />

Select **VPC only** and provide a name tag. For this project, I named the VPC **dev-vpc**.

Enter the same CIDR block used in the reference architecture: **10.0.0.0/16**. Leave the remaining settings as default and select **Create VPC**.

<img width="1244" height="763" alt="Screenshot (412)" src="https://github.com/user-attachments/assets/27233207-6ba9-4f1d-aaab-2b55fda22a6f" />
<img width="1246" height="770" alt="Screenshot (413)" src="https://github.com/user-attachments/assets/b9bbb3ce-0a69-4f36-b21c-61d4fe0d9a3b" />

Next, select the **Actions** tab and choose **Edit VPC settings**.

<img width="1276" height="763" alt="Screenshot (425)" src="https://github.com/user-attachments/assets/a905556e-594f-45aa-b0d0-3a223a54b3a7" />

Enable **DNS hostnames** and select **Save changes**.

<img width="1249" height="755" alt="Screenshot (416)" src="https://github.com/user-attachments/assets/cb68d331-0a09-4753-b9e0-75c45d7fbc3c" />

## 2. Create an Internet Gateway

On the left-hand side of the VPC dashboard, select **Internet Gateways**, then select **Create Internet Gateway**.

<img width="1268" height="765" alt="Screenshot (417)" src="https://github.com/user-attachments/assets/87480726-34fb-4e4d-a8df-59266977c21f" />
<img width="1243" height="765" alt="Screenshot (418)" src="https://github.com/user-attachments/assets/9b356824-42a0-4e50-88d4-0a811ca0482f" />

Give the Internet Gateway a name and select **Create Internet Gateway**.

<img width="1249" height="771" alt="Screenshot (420)" src="https://github.com/user-attachments/assets/ee0deed4-6f21-4f62-a708-f97edf895f5a" />

Attach the Internet Gateway to the VPC created earlier.

<img width="1249" height="771" alt="Screenshot (420)" src="https://github.com/user-attachments/assets/ee0deed4-6f21-4f62-a708-f97edf895f5a" />

## 3. Create Two Public Subnets

<img width="1265" height="766" alt="Screenshot (421)" src="https://github.com/user-attachments/assets/900eb2e9-ca25-454e-8147-c65c13d1c421" />

Select **Filter by VPC**, then select the VPC you created. This ensures that only the subnets associated with your VPC are displayed and helps prevent configuration errors.

<img width="1260" height="762" alt="Screenshot (422)" src="https://github.com/user-attachments/assets/4b661c7a-6d55-4e4c-b6da-015cad85345f" />

Select your VPC under the **VPC ID** section and enter the values specified in the reference architecture.

For the first public subnet:

* **Name:** public-subnet-az1
* **Availability Zone:** us-east-1a
* **IPv4 CIDR block:** 10.0.0.0/24

<img width="1251" height="765" alt="Screenshot (423)" src="https://github.com/user-attachments/assets/a0f8d95a-707f-4488-93f9-40d09a85e3aa" />

Leave the remaining settings as default and select **Create subnet**.

<img width="1250" height="763" alt="Screenshot (424)" src="https://github.com/user-attachments/assets/5262dab4-84ea-4d61-ab56-e17d4787a30b" />

After creating the subnet, select it, click the **Actions** tab, and select **Edit subnet settings**.

<img width="1276" height="763" alt="Screenshot (425)" src="https://github.com/user-attachments/assets/a905556e-594f-45aa-b0d0-3a223a54b3a7" />

Enable **Auto-assign IPv4 address** and save the changes.

<img width="1247" height="760" alt="Screenshot (426)" src="https://github.com/user-attachments/assets/dccf2027-64dd-4262-ba07-48f1cdbde450" />

Create the second public subnet using the same process:

* **Name:** public-subnet-az2
* **Availability Zone:** us-east-1b
* **IPv4 CIDR block:** 10.0.1.0/24

## 4. Create a Public Route Table

The public route table will be used to connect the public subnets to the internet.

On the left-hand side of the VPC dashboard, select **Route Tables**, then select **Create route table**.

<img width="1269" height="757" alt="Screenshot (428)" src="https://github.com/user-attachments/assets/b336a4f2-3907-4f3f-acf2-8d9c8f898588" />

Give the route table a name and select your VPC, then select **Create route table**.

<img width="1280" height="766" alt="Screenshot (429)" src="https://github.com/user-attachments/assets/973c2ecd-b3e4-4cab-b941-21241c2d54e3" />

Next, edit the routes to create a route that connects the route table to the internet through the Internet Gateway.

Add the following route:

* **Destination:** 0.0.0.0/0
* **Target:** Internet Gateway
* **Gateway:** Your Internet Gateway

<img width="1247" height="766" alt="Screenshot (431)" src="https://github.com/user-attachments/assets/dc4473ca-e50f-4e09-8187-739efd6b49c3" />

Next, select **Subnet associations** and edit the subnet associations to attach both public subnets to the route table.

<img width="1281" height="754" alt="Screenshot (432)" src="https://github.com/user-attachments/assets/db9f0a1b-00bc-4d69-8706-6c19e597cda3" />

Select the two public subnets and save the association.

<img width="1269" height="745" alt="Screenshot (433)" src="https://github.com/user-attachments/assets/a3bf9625-ad05-422e-8f6f-49ab3e306277" />

## 5. Create the Private Subnets

Next, create the private subnets.

For the first private subnet:

* **Name:** private-app-subnet-az1
* **Availability Zone:** us-east-1a
* **IPv4 CIDR block:** 10.0.2.0/24

<img width="1225" height="770" alt="Screenshot (435)" src="https://github.com/user-attachments/assets/c3c7a388-405e-43c0-80d7-6bb179baa648" />

Create the second private subnet using the following configuration:

* **Name:** private-app-subnet-az2
* **Availability Zone:** us-east-1b
* **IPv4 CIDR block:** 10.0.3.0/24

## 6. Create NAT Gateways

Next, create the NAT Gateways.

<img width="1248" height="760" alt="Screenshot (436)" src="https://github.com/user-attachments/assets/d482ea32-fb9f-4536-b811-00c90e2cf41b" />

Give the NAT Gateway a name and select the appropriate Availability Zone. For the first NAT Gateway, select **public-subnet-az1** and allocate an **Elastic IP address** to it.

<img width="1227" height="760" alt="Screenshot (438)" src="https://github.com/user-attachments/assets/65a52702-df5f-4e0d-8715-06ac2492e6e2" />

Repeat the same process for **NAT Gateway AZ2**, creating it in **public-subnet-az2**.

## 7. Create Private Route Tables

Next, create private route tables for the private subnets.

First, create a private route table for the private application subnet in **us-east-1a**.

<img width="1269" height="740" alt="Screenshot (439)" src="https://github.com/user-attachments/assets/734e565f-9b74-4fda-b9ad-f849afe50475" />

Edit the routes and configure the default route to use **NAT Gateway AZ1**, then save the changes.

<img width="1269" height="765" alt="Screenshot (440)" src="https://github.com/user-attachments/assets/4a67c6f6-791d-446b-bfa9-9cce76b0dfe4" />

Next, edit the **Subnet associations** and associate the route table with **private-app-subnet-az1**.

<img width="1252" height="748" alt="Screenshot (441)" src="https://github.com/user-attachments/assets/a94c34f5-088c-4453-bca7-5a78375ccb4a" />

Repeat the same steps to create a route table for **private-app-subnet-az2** and configure it to route outbound traffic through **NAT Gateway AZ2**.

## 8. Create Security Groups

Security groups are virtual firewalls attached to AWS resources that control inbound and outbound traffic at the resource level.

First, create the **ALB security group**.

<img width="1261" height="765" alt="Screenshot (443)" src="https://github.com/user-attachments/assets/16861885-9a97-4c68-a818-975fd4f96cd1" />

Next, create the **SSH security group**.

<img width="1267" height="776" alt="Screenshot (444)" src="https://github.com/user-attachments/assets/69f603f7-8015-4b2b-baf3-b6126479a7e3" />

Finally, create the **web server security group**. Configure HTTP and HTTPS access so that the source is the **ALB security group**. Configure SSH access so that the source is the **SSH security group**.

<img width="1239" height="723" alt="Screenshot (446)" src="https://github.com/user-attachments/assets/68cac0c1-9cc2-409d-a0b2-e1a77080474f" />

## 9. Launch EC2 Instances and Install the Application

Next, launch the EC2 instances and install the application using a Bash script.

In the search bar, type **EC2** and select **Launch instance**.

<img width="1037" height="829" alt="Screenshot (448)" src="https://github.com/user-attachments/assets/48cdec69-8f00-405e-97f9-971d0c26d193" />

Give the instance a name and select the **Amazon Linux AMI**, using **Amazon Linux 2023**.

<img width="1269" height="786" alt="Screenshot (449)" src="https://github.com/user-attachments/assets/ea46587f-e8f6-48cf-9316-959b6c40288f" />

<img width="829" height="399" alt="Screenshot (450)" src="https://github.com/user-attachments/assets/db70476f-c284-44eb-9632-1c6ef586e8e7" />

Under the network settings, select **dev-vpc**, choose **private-app-subnet-az1**, and select the **webserver security group**.

<img width="1255" height="766" alt="Screenshot (451)" src="https://github.com/user-attachments/assets/75e74ab0-5d56-407e-a4c6-d9d8e41d62db" />

Leave the storage settings as default. Under **Advanced details**, upload the Bash script.

<img width="938" height="94" alt="Screenshot (452)" src="https://github.com/user-attachments/assets/a782a335-7291-415f-887f-3c1306df7b14" />

The Bash script used to install and configure the web server is:

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

Repeat the same process for the second web server, creating it in **private-app-subnet-az2**.

## 10. Create an Application Load Balancer and Target Group

Next, create an **Application Load Balancer** and a **target group** to attach the EC2 instances to the load balancer.

### Create a Target Group

From the EC2 console, select **Target Groups** from the left-hand navigation menu.

<img width="1263" height="756" alt="Screenshot (453)" src="https://github.com/user-attachments/assets/11d7b3bd-3561-49d1-bab7-27a208b5a5ad" />

Select **Create target group** and provide a name for the target group.

<img width="1241" height="760" alt="Screenshot (454)" src="https://github.com/user-attachments/assets/272c53d4-b683-4083-b02b-3796dc9b636f" />

Leave the other settings as default and configure the health check success codes to include **200, 301, and 302**.

<img width="1238" height="766" alt="Screenshot (455)" src="https://github.com/user-attachments/assets/24bf1f45-586e-4f09-a18e-0887533969f7" />

Select **Next**.

<img width="1240" height="750" alt="Screenshot (456)" src="https://github.com/user-attachments/assets/1913ef6e-e485-49d5-8de2-5a54afcac471" />

Select the two EC2 instances and choose **Include as pending below**.

<img width="1258" height="771" alt="Screenshot (458)" src="https://github.com/user-attachments/assets/20086f45-b0f2-4dba-96d1-23e487009b18" />

Review the settings and select **Create target group**.

## 11. Create the Application Load Balancer

From the left-hand side of the EC2 console, select **Load Balancers** and create a new load balancer.

<img width="1277" height="769" alt="Screenshot (459)" src="https://github.com/user-attachments/assets/a15ad038-1fa2-45c0-9be8-a3bd90f85047" />

Select **Application Load Balancer**.

<img width="3727" height="1534" alt="Screenshot (461)" src="https://github.com/user-attachments/assets/be834aff-3e22-496d-a8ca-1229e9b7d3c1" />

Select **dev-vpc** and choose the two public subnets:

* **public-subnet-az1** — us-east-1a
* **public-subnet-az2** — us-east-1b

<img width="1277" height="769" alt="Screenshot (465)" src="https://github.com/user-attachments/assets/d26084c0-a90f-49a9-837d-bc99482c865e" />

Select the **ALB security group**.

<img width="1244" height="757" alt="Screenshot (466)" src="https://github.com/user-attachments/assets/a6353b09-f81d-4490-9a71-0e1ffac26bf5" />

Configure the **HTTP listener** and select the target group created earlier.

<img width="1247" height="767" alt="Screenshot (467)" src="https://github.com/user-attachments/assets/48519a35-5bc5-4ae5-be46-a724c175c99d" />

Leave the remaining settings as default and select **Create load balancer**.

Wait until the Application Load Balancer's status changes to **Active**. Then copy the ALB's **DNS name** and paste it into your browser.

At this stage, use **HTTP** rather than **HTTPS**, since an HTTPS listener has not yet been configured.

<img width="1248" height="754" alt="Screenshot (468)" src="https://github.com/user-attachments/assets/cb215eb4-8798-4561-9565-f214ff0e50c0" />

The webpage should then be displayed in your browser.

<img width="1245" height="847" alt="Screenshot (469)" src="https://github.com/user-attachments/assets/5a0d9e4a-ace0-4827-a904-ff26041f4c7b" />

## Conclusion

This project provided practical experience in designing and deploying a **secure, highly available, and scalable AWS infrastructure**. Rather than relying on a single EC2 instance, the architecture used multiple Availability Zones, private subnets, an Application Load Balancer, NAT Gateways, and other AWS services to improve the reliability and security of the application.
