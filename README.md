Building a Scalable Multi-Tier Web Application on AWS

Goal: Designed for scalability and high availability, this application uses an autoscaling group of EC2 instances to serve web requests, with a dedicated backend infrastructure for queue management, database storage, and caching. 

Requirements
Before diving into the implementation, let’s start with the requirements:

Web Application Server: Hosted on Tomcat 10, listening on port 8080 on EC2 instances.
Web Application Source Code: Build one or fork from github. Also, all the EC2 userdata scripts are in the GitHub repository to reduce the length of this post.
Load Balancing and Autoscaling: Application Load Balancer (ALB) in front of an EC2 Autoscaling Group to handle web traffic.

Backend Servers:
RabbitMQ for message queuing services.
MySQL for user data and web app credentials.
Memcached for caching content to improve performance.
Domain and SSL: A custom domain with HTTPS enabled using AWS Certificate Manager (ACM).
DNS Management: Route 53 to manage DNS for both public and private traffic.
AWS CLI & Maven: AWS CLI to interact with AWS environment & Maven to build code

Architecture Design
This is a classic three-tier architecture:

Presentation Layer (Web App Frontend): Autoscaling EC2 instances running Tomcat.
Application Layer (Service Backend): Consists of RabbitMQ, MySQL, and Memcached instances.
Data Layer (Database and Caching): MySQL for persistence and Memcached for caching.

Implementation Steps
1. Creating Key Pairs and Security Groups
Key Pair: Create a key pair to access all EC2 instances via SSH.
Security Groups: Configured three security groups:
Web Application Security Group: Allow inbound traffic on port 8080 from the ALB.
Backend Security Group: Allow traffic from the web application for communication with RabbitMQ (port 5672), MySQL (port 3306), and Memcached (port 11211). Also, add a rule to the backend security group that allow traffic from itself. This ensures the backend instances can communicate with themselves internally.
Application Load Balancer Security Group: Allow inbound traffic on ports 80 (HTTP) and 443 (HTTPS) for the ALB. Also, allow SSH on port 22 into all your instances and pick the source as MYIP so only your permitted IP can access instances.
inbound rules for backend security group

2. Launching EC2 Instances with User Data Scripts
Database Instance: Launch MySQL instance and include a user data script to automatically install the MySQL server upon boot. It will also create a user to access your webapp. SSH into the MYSQL instance from your local system then confirm the database is running.
ssh -i "keypair.pem" ubuntu@ec2-public-ip
systemclt status mariadb
Image description

Login with your created user details and check your database

mysql -u username -p accounts
show tables;
Image description

Memcached Instance: Launch a separate instance for Memcached, using a user data script to install and configure the service.
Image description

Confirm the service is running

systemctl status memcached
RabbitMQ Instance: Also, using the user data script, launch and configure Rabbitmq instance for message queue handling.
Confirm the service is running

systemctl status rabbitmq-server
Image description

Web Application EC2: Launch a Tomcat 10 server instance to serve the web application using Ubuntu AMI. Create EC2 role that grants read and write access to s3 and attach it to the EC2.
3. Configuring Route 53 for DNS
Domain Registration and SSL: Register a domain with Route 53, request an SSL certificate through ACM, and create a public hosted zone.
Public DNS: Add a CNAME record pointing the ALB’s endpoint to the domain for HTTPS access.
Private DNS: Create a private hosted zone with A records for the backend services’ private IP addresses to allow internal communication by name. Ensure you update the route53 internal record in your application.properties file.
4. Building and Deploying the Application Code
S3 Bucket Creation: Create an S3 bucket to store the application artifact.
IAM Role: Assign an IAM role to the web application EC2 instance to allow S3 access.
Application Build: Build the application using Maven on the local machine and uploaded the artifact to S3.
Code Deployment: SSH into the web application EC2, downloaded the artifact from S3, and deployed it to the Tomcat server. Each step is defined below.
create a bucket that will hold the web app artifact

aws s3api create-bucket --bucket fozwebapp-artifact --region eu-west-2 --create-bucket-configuration LocationConstraint=eu-west-2
Run Maven from the directory that holds the source code in your local system

mvn install
Copy the artifact from your local system to the s3 bucket. Then, SSH into the instance, install AWS CLI and copy the artifact from s3 into the EC2 instance.

aws s3 cp target/vprofile-artifact s3://fozwebapp-artifact
Image description

Install AWS CLI and copy artifact from s3 to /tmp/ in EC2

snap install aws-cli --classic
aws s3 cp s3://fozwebapp-artifact/vprofile-v2.war /tmp/
Image description

Now, stop tomcat10 on the EC2, remove the default directory of tomcat10, and replace it with the artifact the restart. Run each of the commands one after the other. Lastly, you can confirm tomcat10 is running

systemctl daemon-reload
systemctl stop tomcat10
rm -rf /var/lib/tomcat10/webapps/ROOT
cp /tmp/vprofile-v2.war /var/lib/tomcat10/webapps/ROOT.war
systemctl start tomcat10
tomcat10 running

5. Setting Up the Application Load Balancer
Target Group Creation: Configure an ALB target group for the autoscaling EC2 instances, setting listener and health checks on port 8080 to ensure only healthy instances serve traffic.
HTTPS Configuration: Configure the ALB to use the SSL certificate from ACM.
DNS Mapping: Map the domain’s CNAME record in Route 53 to the ALB for secure, HTTPS-enabled access.
Target group

Application Loadbalance Endpoint mapped to app domain name
loadbalancer endpoint

6. Connecting to the Web Application and Verifying
Access Verification: Access the web app through the custom domain and verify load balancer forwarding by inspecting responses from the Tomcat server.
Service Checks: SSH into each backend instance to ensure that RabbitMQ, MySQL, and Memcached services were running and accessible by the web application server.
7. Configuring Auto Scaling for the Web Application EC2 Instances
Autoscaling Policies: Set up scaling policies to adjust the number of EC2 instances in response to application load.
Monitoring and Logging: Enabled CloudWatch metrics to monitor CPU and request metrics, triggering autoscaling when defined thresholds were met.
webapp running and reachable on https

How the Web Application Works
User Access: Users access the application via the custom domain. The ALB handles the HTTPS requests and forwards them to Tomcat servers running in the autoscaling group.
Backend Processing: When a web request involves queuing or caching, the application server communicates with RabbitMQ and Memcached instances. For user data requests, it connects to MySQL.
Scaling and Availability: The autoscaling group dynamically adjusts based on load, ensuring efficient handling of fluctuating traffic.
Possible Improvements for the Architecture
Containerization of applications and deployment with kubernetes Enhanced scalability, automated management, and flexibility for microservices deployment.

Employ AWS serverless solutions like Lamba, API Gateway, AWS RDS in place of MYSQL, ElastiCache, and MQ in place of Memcached & Rabbitmq

Elastic Beanstalk to simplify deployment, scaling, and monitor without direct server management.

Front load balance with Cloudfront or AWS Global Accelerator depending on the type of content the app is serving.