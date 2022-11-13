# Designing Network and Compute architecture for a Highly Available Web Application.

## Lab 01 – Part 01 of 03

Private networks in AWS are created by the service called Amazon Virtual Private Cloud (VPC). We can create a VPC within any AWS region of our choice and within that VPC we define subnets to manage related sets of servers or other AWS resources. VPC service lets us define rules for how network traffic for our subnets is routed. We can also decide whether our private network (VPC) should be connected to the Internet, to corporate networks or to keep the network completely private.

In this lab we are going to design the network for a highly available two tier web application. The load balancer will be deployed in two public subnets across two availability zones having Internet connectivity via Internet Gateway. Rest all web/app/db servers will be deployed in two private subnets across two availability zones having outbound only internet connectivity via network address translation (NAT) service. The incoming and outgoing traffic will be filtered by virtual firewalls; by NACL at Subnet level and by Security Groups at instance/resource level.

The network after building, will look similer to the picture below

![](https://github.com/nvaws/labs/blob/main/images/NetworkDiagram.png)

### Activity 01 – Creating a VPC and associated resources

Login to your AWS account and find VPC under Networking & Content Delivery category.  
Click on VPC Dashboard in the side bar and then click on Launch VPC Wizard.  

Most of the below fields will be prefilled for you please verify, if not then update the below values.

* Resources to create: VPC, subnets, etc.
* Name tag auto-generation: "Lab"
* IPv4 CIDR block: 10.0.0.0/16
* IPv4 CIDR block: No IPv6 CIDR Block
* Tenancy: default
* Availability Zones (AZs): 2
* Number of public subnets: 2
* Number of private subnets: 2
* NAT gateways ($): None (this is chargable resource, let's not create it)
* VPC endpoints: S3 Gateway
* Keep both the options under DNS Options checked and click on Create VPC.

Click on Your VPCs in the side bar and see the new VPC.

_Did you notice that a VPC (default VPC) was already created? Find out what other resources were automatically created for you in VPC and why._

:key: In the Subnets section, select one of your public subnets and Actions dropdown; go to Edit Subnet Settings and check Enable auto-assign public IPv4 address box.  Click on Save.

Repeat the same step for the other Public Subnet as well but do not enable this setting for Private Subnets.

_Why have we created two private and public in different subnets? Should we not create both Public subnets in one AZ and both Private in another AZ?_

Please explore the three Route Tables that have got created for you. Look at their Routes and Subnet Associations. 

Creating Security Groups

Let us now create two different 'Security Groups' for Web server and load balancer. We would leverage them in coming steps.
In the navigation pane find and click on 'Security Groups'

* Click on 'Create Security Group'
  * Security group name: My-Web-SG
  * Description: This SG is to be used for web application servers.
  * VPC: Lab-vpc
* Click on Create

Similarly create another security group for your Loadbalancer layer.

Select either of the Security Group now and click on 'Inbound Rules' tab.
Click on 'Edit Rules' and add rules for incoming traffic on the security groups like mentioned below.

#### My-Web-SG

| Type  | Protocol | Port Range | Source |           |
| :---: | :------: | :--------: | :----: | :-------: |
| HTTP  |   TCP    |     80     | Anywhere-IPv4 | 0.0.0.0/0 |
| HTTPS |   TCP    |    443     | Anywhere-IPv4 | 0.0.0.0/0 |
|  SSH  |   TCP    |     22     | Anywhere-IPv4 | 0.0.0.0/0 |

#### My-ALB-SG

| Type  | Protocol | Port Range | Source |           |
| :---: | :------: | :--------: | :----: | :-------: |
| HTTP  |   TCP    |     80     | Anywhere-IPv4 | 0.0.0.0/0 |
| HTTPS |   TCP    |    443     | Anywhere-IPv4 | 0.0.0.0/0 |

For now, our VPC configuration is complete. The instances launched in our public subnets should have outbound access to Internet and the instances in our private subnet should not. We would verify the same in the next section.


### Activity 02 - Creating EC2 instances

We are now going to create an EC2 instance as MyWebServer. Let us switch to EC2 Dashboard now and click on Launch Instance.

Creating the Web Server

* Name: MyWebServer
* Amazon Machine Image: "Amazon Linux 2" (this should be already selected)
* Instance Type: t2.micro
* Key Pair: Create a new Key Pair -> Key pair name: mykey, leave the rest as default and click on Create Key pair.

In the Network Settings section, edit and fill the below values
* VPC: Lab-vpc
* Subnet: Select one of your Public Subnets
* Firewall (security groups): Select existing security group -> My-Web-SG

Configure storage: Leave everything as it is.


**Expand the Advance Details section and paste the following script in the user data section at the bottom.**

```bash
#!/bin/bash
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
cd /var/www/html
wget https://raw.githubusercontent.com/nvaws/labs/main/index.txt
INSTANCEID=`curl http://169.254.169.254/latest/meta-data/instance-id`
INSTANCETYPE=`curl http://169.254.169.254/latest/meta-data/instance-type`
PRIVATEIP=`curl http://169.254.169.254/latest/meta-data/local-ipv4`
sed -e "s/INSTANCEID/$INSTANCEID/" -e "s/INSTANCETYPE/$INSTANCETYPE/" -e "s/PRIVATEIP/$PRIVATEIP/" index.txt > index.html
```

This script will –

1 - Install an Apache web server and create a Hello World! page.  
2 - Activate the Web server and configure it to automatically start on reboots.  
3 - Read some EC2 instance related data from EC2 metadata and show it on the webpage.  

Click on Launch Instance.

We have just launched an EC2 instance in our public subnet as MyWebServer. 

Try browsing the public DNS of the web server, does it open? Yes, because your MyWebServer is in a public subnet, has a public IP and the firewall allows the traffic on port 80 from anywhere. In an ideal scenario, only the Load Balancer should be in public subnet and the web servers should be in Private Subnets. The firewall of EC2 should allow traffic coming from the load balancer and not from anywhere, we will do that in next section. The instance details you see on the webpage is presented by reading the meta-data of the EC2 instance.

Go through the various options under Action drop down. 

After an instance is configured, we can build an image from it for our future usage so we dont have to start from scrach if we want to launch multiple EC2 instances. It can be done going to the Action dropdown, save your Image and create image. You don't have to do it right now.

Terminate the instance.

### Activity 03 - Creating an Auto Scaling Group

You will now be launching this application using a Launch Template into an Auto Scaling Group, the ASG will automatically grow and shrink the number of your servers based on the user defined threshold. The requests to your application will be distributed by Application Load Balancer.

Click on Launch Templates in the side bar and click on Create Launch Template

- Launch template name: MyWebServer_LT
- Application and OS Images (Amazon Machine Image): Quick Start -> Amazon Linux 2 (Free tier eligible)
- Instance type: t2.micro
- Key pair (login): Select the one you previously created

Leave all other setting as default and scroll down to Advanced details

**Expand the Advance Details section and paste the previously shared script in the user data section at the bottom.**

- Now click on Create launch template.

Your Launch template is created, let us now use this Launch template to create an auto scaling group. Select your newly created Launch template, click on action dropdown and scroll down to find Create Auto scaling group, click on it.

- Auto Scaling group name: MyWeb_ASG
- Launch template: it is already selected. Click Next.
- Network: Select Lab-vpc from the dropdown and then selct both of the public subnets from the Subnets dropdown. Click Next.
- Load balancing - optional: ignore everything on this page and click Next.
- Group size: Desired capacity 2, Minimum capacity 2 and Maximum capacity 4.
- Scaling policies: Select Target tracking scaling policy. Metric type: Average CPU utilization, Target Value: 50, Click Next.
- Click on Add notification
- Add Notification: Create Topic
- Send a notification to: MyASG_Topic
- With these recipients: 'your email ID'. Click Next. 
- Next Create a Tag with 'Key: Name' and 'Value: MyWebServer'
- Review: Create Auto Scaling group

Now go to the Auto Scaling Groups Dashboard. Explore the Activity History and other tabs.

Also check if you received an email from SNS topic, you need to confirm the subscription.

You have just launched our highly available web application in an Auto Scaling Group. You can browse the application by the instances' public IPs/DNS.

Let us create a Load balancer in public subnets, that will divert the traffic to both these instances in round robin method.

### Creating an Application Load Balancer

Go to the Load Balancing section of EC2 dashboard and click on Target Group

- Create Target Group
- Target group name: MyTG
- VPC: Lab-vpc
- Leave rest defaults and click Next.
- Leave everything on this page unchanged and click Create target group.

Let us register our instances in ASG with the MyTG target group. Go to Auto Scaling Groups, select your ASG and click on Edit button on top. Scroll down to find the Load balancing section, click on the first check box and select the target group you created from the dropdown. Go to the bottom of the page and click on Update.

Go to the Load Balancers page and click on Create Load Balancers.

- Load balancer types: Application Load Balancer -> Create
- Load balancer name: MyALB
- Scroll down to find the Network mapping section
- Select the VPC in which you have launched the ASG
- Select Public Subnets from both AZs. This is a critical step, reconfirm before going forward.
- Configure Security Settings – Ignore the warning, it is recommending to have SSL certificate.
- Security groups. Select My_ALB_SG from existing ones and remove any other if already selected.
- Listeners and routing: Leave default HTTP listener and select the MyTG Target Group from the dropdown.
- Leave rest defaults and Create load balancer

Once on the load balancer dashboard, you should see the DNS endpoint (A record) of your load balancer in Description Tab. ALB takes a little time to come up. Refresh till you see the state as active.

Open the DNS address of your ALB in a browser and notice what it shows. It is now diverting the traffic to both your instances. You can see the behavior of load balancer while you refresh the page and notice the instance ID/IP.


### Modify the Security Groups to ensure security on incoming traffic

Update the **My-Web-SG** security group settings as shown below.

#### My-Web-SG

| Type  | Protocol | Port Range | Source |           |
| :---: | :------: | :--------: | :----: | :-------: |
| HTTP  |   TCP    |     80     | Custom | My_ALB_SG |
| HTTPS |   TCP    |    443     | Custom | My_ALB_SG |
|  SSH  |   TCP    |     22     | Custom | 0.0.0.0/0 |


You can now try deleting one/more server in order to verify whether the auto scaling feature is able to spin up instances in response. You can also simulate the CPU load on servers by some stress test tool to see scale out action.

### Clean up steps –

Delete the resources in the below order

 1: ALB  
 2: Auto Scaling Group (takes a little time to delete, find out why in the activity history tab)  
 3: Target Group  
 4: Launch Template 


***All the services used in this lab are eligible and covered within the free tier account. There should be no charges if you delete all the resources within free tier monthly limits.***

✔️ Lab Complete!
