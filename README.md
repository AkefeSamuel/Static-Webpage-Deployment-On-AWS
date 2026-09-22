## How I deployed a static website on AWS

### Introduction 
What is a static website? A static website is a type of website that serves pre-built HTML files directly to users without any server-side processing. In this project we deployed a static webpage built by Azeez Salu on AWS and utilized a key number of services in bringing this project to life. More insights on how we built this project would be listed below.

 <img width="723" height="638" alt="image" src="https://github.com/user-attachments/assets/6f2a9ea9-0f7b-4369-8e4a-6ea77c4056c3" />

Key AWS services that we used
1.	Virtual Private Cloud (VPC) with Public and Private Subnets.
2.	Security Groups
3.	Elastic Compute Cloud (EC2) 
4.	Network Address Translation (NAT) Gateways
5.	Application Load Balancer (ALB)
6.	Route53
7.	Certificate Manager
8.	Autoscaling groups.

How were these services used?
1.	Virtual Private Cloud (VPC) with Public and Private Subnets: The AWS Virtual Private Cloud enabled you to provision a logically isolated section of the AWS Cloud where we could launch AWS resources in a virtual network that we defined. The VPC was made up of two tiers. The first tier contained a public subnet and it hosted resources such our NAT gateways and our application load balancer. The second tier contained private subnets that housed our webservers. The subnets were duplicated across multiple availability zones in order to ensure high availability and fault tolerance. We also created an internet gateway with route tables in order to allow the resources in our VPC have access to the internet.
2.	Security groups: The Security Groups acted as virtual firewalls that protect resources such as our EC2 instances (webservers). They filter incoming traffic into our AWS resources.
3.	Elastic Compute Cloud (EC2): These are virtual servers on which our website was installed in.
4.	Network Address Translation (NAT) Gateways: These were placed in the public subnets in order to allow outbound internet access from our webservers whenever they needed to download updates, packages and also pull zip files from GitHub.
5.	Application Load Balancer: The Application Load Balancer was used to dynamically distribute traffic across EC2 instances in order to prevent a single server from being overwhelmed.
6.	Route 53: This was used to create a record set in order to point our domain name to the Application load balancer.
7.	Certificate Manager: This was used to create a Secure Sockets Layer (SSL) certificate in order to encrypt all communication between our web browser and our web servers.
8.	Autoscaling groups: The Autoscaling groups were used to dynamically create and scale our EC2 instances in the private app subnets.
How did I build this?
1.	Create a custom VPC using the reference architecture.
a.	On the AWS Management, select the N.Virginia region and in the search box, type vpc and select it.
 


 

b.	Select the Create VPC icon and select it.

 
c.	Select VPC only and give it a name tag. I am naming mine dev-vpc. Give it a CIDR range also, I would be using 10.0.0.0./16. Leave everything else as default and create your VPC.
 


 
d.	Next select the actions tab then select edit VPC settings.

 

e.	Tap enable DNS hostnames and save your configuration.
 

2.	Create an Internet Gateway. The Internet Gateway enables your VPC to communicate with the internet.
a.	On the lefthand side of the management console, select internet gateway from the list of options and Create Internet Gateway.
 

b.	Name it dev-igw, then create it.
 



c.	Next, select Attach to a VPC.

 

d.	Select your VPC from the list of options and attach.
 





3.	Create your public subnets AZ1.
a.	On the lefthand side of your management console, select subnets and then Create Subnet.

 
b.	Select your dev-vpc
 




c.	Edit your subnet settings.
 

d.	Create Subnet.
 






e.	Next, select the subnet you just created and click on the actions icon then edit subnet settings.
 

f.	Enable auto-assign public IPv4 address. This is done so that all resources created in this subnet are automatically assigned a public ipv4 address.
 





g.	Save changes.
 

4.	Create your public subnet AZ2 using the same steps we used to create the public subnet AZ1. However, give it the following values.
a.	Name public-subnet-AZ2
b.	Availability Zone: us-east-1b
c.	IPv4 CIDR block: 10.0.1.0/24.
Afterwards, enable auto-assign public IPv4 address from the part edit subnet settings like we did earlier.

5.	Create a public route table in order to route traffic from our public subnet to the internet through the internet gateway.
a.	On the lefthand part of the management console, select route tables and create route table.
 
b.	Next, give your route table a name and select your VPC.
 
c.	Create your route table.
 
d.	Next, we will be adding a public route in order to route our traffic to the Internet. On the page you are redirected to after creating your route table, select the route section and edit route.
 

e.	Click on Add route and select 0.0.0.0/0. This is the destination address for the internet. Then select Internet Gateway as the target, and your dev internet gateway under it then save changes.
 
 

f.	Next, we will be editing our subnet associations in order to attach both of our public subnets to the route table. On the lefthand side of the management console, select route tables then edit your subnet associations.
 






g.	Select both public subnets then save association.
 

6.	Create your private subnets.
a.	Similar to how we did earlier, on the leftmost hand side of your AWS Management Console, select subnets and create subnet.
 
b.	Select your dev VPC, give it the following configurations then create subnet.
Name: private-app-subnet-az1
Availability Zone: us-east-1a
IPv4 subnet CIDR block: 10.0.2.0/24.
c.	Create your second private app subnet. Similar to how we created the subnets above, we would select subnets on the leftmost hand side of our AWS management console and select create subnet.
 

d. Select your dev VPC, give it the following configurations then create subnet.
Name: private-app-subnet-az2
Availability Zone: us-east-1b
IPv4 subnet CIDR block: 10.0.3.0/24.
Note that the configurations here are different from the private app subnet AZ1.
We won’t also enable auto-assign public IPv4 address because it is a private subnet and we don’t want our resources exposed to the public internet. We would be deploying our webservers in the private subnets.

7.	Create NAT gateways and private route tables in the first and second Availability zones.
a.	On the leftmost hand side of the AWS management console, select NAT gateways and create your NAT gateway.
 

b.	Give it a name. We would be naming ours nat-gateway-az1 and under availability mode, we would be selecting zonal. Lastly under subnet we would be selecting public subnet az1 as the NAT gateways need to be in a public subnet in order get access to the internet. They serve to allow outbound internet access from private subnets in cases where packages and updates have to be downloaded from the resources there.
 
c.	Select allocate Elastic IP, wait till you see an elastic IP address in the allocation textbox then leave other settings as default then create NAT gateway AZ1. It is called NAT Gateway AZ1 because it is deployed in the first availability zone (us-east-1a).
 
 
d.	Create private route az1 to route traffic from the resources in the private subnet az1 to the NAT gateway AZ1. On the leftmost part of the management console, select Route tables and create route table.
 
e.	Give it a name, select your VPC and create your route table.
 

 






f.	Next, we will be editing the routes of the private-route-table-az1 to route outbound traffic to the internet through the NAT gateway AZ1. On the management console, select routes and edit the route of your private-route-table-az1.
 
g.	Similar to how we did it earlier, select add route then select 0.0.0.0/0.
 






h.	Select NAT Gateway and select NAT Gateway AZ1 then select save changes.
 
i.	Next, we would be editing subnet associations for the private-route-table-az1.
 






j.	Click edit subnet settings and select private-app-subnet-az1and save associations. Remember, this route table is for our availability zone us-east-1a.
 
Now we have added a route to our NAT gateway and have also associated our private-app-subnet-az1 to our private-route-table-az1. Next we would be doing the same for our second availability zone.
k.	Using the same steps above, create NAT gateway AZ2 in the us-east-1b availability zone then create a route table and create a public route to the NAT gateway AZ2 you crated then associate your private-app-subnet-az2 to it from the subnet associations section under the private-route-table-az2 configurations.

8.	Next, we would be creating our security groups. Security groups are stateless firewalls that filter incoming traffic at your resource level. i.e. the filter the traffic that come into your instances. On the lefthand side of your management console, select security groups and create security group.
 
a.	First, we would be creating the Application Load Balancer Security group. We would be opening port 80 and 443 in order to allow http and https access from the internet respectively. Give the security group a name, description and select your dev vpc.
 
b.	Add inbound rules http and https on port 80 and 443 respectively and let their source be the internet (0.0.0.0/0).
 






c.	Leave other settings as default and select create security group.
 
d.	Next, we would be creating our ssh security group in order to allow us ssh into our ec2 instances. Similar to how we created the first security group, go to the create security group page. Give it a name, description and select your VPC.
 





e.	Next, add inbound rules. Create an inbound rule of type SSH and make your IP address the source then create your SSH security group.
 
 







f.	Up next, we will be creating our webserver security group. Give it a name, description and select your dev vpc.
 
g.	Next, add your inbound rules. HTTP, HTTPS and SSH respectively. For the http and https types, make their source, the alb security group while you make the ssh security group the source for the ssh rule type.
 





h.	Create your webserver security group.
 
9.	Create EC2 instances in your private subnets.
a.	On your AWS Management Console, select EC2 in the search bar.
 




b.	Click launch instance.
 
c.	Name your EC2 instance webserver az1 and select the amazon linux AMI.
 







d.	Leave the Amazon Linux 2023 as default. Same as the architecture. Leave at default.
 
e.	Pick t3.micro as your instance type as it is free tier eligible.
 







f.	Create an RSA keypair and select it.
 
g.	Next, edit your network settings. Select your dev-vpc, your private-app-subnet-az1and select existing security group then select your webserver security group.
 
 
h.	Leave the storage as default then select advanced details
 
 
i.	After selecting advanced details, scroll to the end and paste this code in the User data section then select Launch Instance.
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
 
j.	Create another EC2 instance in the private app subnet az2 using the same steps we took. Select the same steps we took above, however in the network settings, select private app subnet az2.


10.	Create an application load balancer to distribute incoming traffic between our provisioned EC2 instances.
a.	On the lefthand side of the EC2 console, select target groups then create target groups. The target groups specify the instances we are connecting our application load balancer to.
 
b.	Select instances
 





c.	Give it a name then leave these other settings as default.
 
d.	Select your dev vpc
 
e.	Leave other settings as default then select next.
 
f.	Select both instances you created and include as pending below.
 
g.	Select next.
 







h.	On the next page, scroll to the end then create target group.
 
i.	Next on the leftmost side of the EC2 console, select Application Load Balancer and Create Application Load Balancer.
 







j.	Select Application Load Balancer then Create.
 
k.	Give it a name and leave scheme and IP address type as default.
 







l.	Select your dev-vpc.
 
m.	Select your availability zones and ensure its in a public subnet(s) so it can connect to the internet.
 







n.	Remove the default security group and select your alb security group.
 
o.	Under listeners and routing, select the target group that you just created.
 
 
p.	Scroll to the end then create load balancer.
 
q.	Once the status of the Application Load Balancer is active, copy the DNS name paste into your web browser in this manner http://yourDNSname. Ensure it’s not pasting as https:// since we’ve not created an HTTPS listener yet.
 






r.	Here, our website is live.
 
11.	I would record a video on how I linked my third-party acquired domain name to route 53 and also how I used Certificate Manager to create a free SSL certificate.
12.	The last thing we would be doing is to create an autoscaling group to dynamically create and destroy EC2 instances based on traffic demands.
a.	Terminate the existing EC2 instances. On your AWS management console, delete the two instances you created earlier so we can create an autoscaling group to recreate the instances back. Select the two instances then select the instance state icon select terminate instance.
 

b.	After the instances have deleted, select launch templates on the leftmost side of the EC2 dashboard.
 
c.	Give your launch template a name and description then select autoscaling guidance.
 







d.	Select Quick Start and select your Amazon Linux AMI.
 
e.	Under Instance type, select t3.micro.
 







f.	Under network settings, select your webserver security group under existing security groups.
 
g.	Scroll to advanced details and paste your bash script just like we did earlier then create launch template.
 






h.	On the EC2 console, select Auto Scaling Groups then create auto scaling group.
 
i.	Give it a name and select launch template then click next.
 







j.	Select your VPC and availability zones then scroll and click next.
 
k.	Select attach to an existing load balancer and select your load balancer’s target group.
 







l.	Scroll down and enable Elastic Load Balancing health checks then click next.
 
m.	Select your desired capacities, then click next.
 







n.	I won’t be adding notifications. Click next.
 
o.	Give it a nametag then click next.
 








p.	Scroll down and create Autoscaling Group.
 
q.	Once it says at desired capacity, it means both of your webservers have launched.
 
r.	Go back and paste http://yourapplicationloadbalancerDNSname and tell me what you see.




13.	Now let’s delete our resources.
Delete them in this order:
a.	Autoscaling Groups
b.	Launch Template
c.	Application Load Balancer
d.	Target Groups
e.	Security groups. Delete the webserver sg first then the others.
f.	NAT gateways
g.	VPC
h.	Elastic IPs
Thank you for staying this long with me. I hope you had a great time.
