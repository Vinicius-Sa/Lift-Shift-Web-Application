#LIFT AND SHIFT WEB APPLICATION WORKLOAD 

The objective of this project is to train how to run application workload on AWS cloud by using lift and shift strategy and simulate real-world client compliance 

We have, application services, which are running on physical or virtual machines, to solve problems like complex management, scale-up/Down complexity, upfront capEX, and regular operation the app will be shifted to the cloud, and with this, we will have flexibility, payasUgo, automation, and easy Infra Management.

**AWS SERVICES** 
- EC2 INSTANCES > VM for tomcat, Rabbitmq, Memcache, MySQL 
- ELB > NGINX lb replacement 
- AUTOSCALING > automation for vm scaling 
- S3/EFS Storage > shared storage 
- ROUTE 53 > private DNS server 
- ACM > certificate manager for https 

*COMPLIANCE** 
- Flexible Infra
- No upfront cost
- Modernize Effectively 


**SETUP** 
> security group & Key Pairs: 
So first we are gonna set up Security Groups and rules, in total are going to be 3 SG 
1 ELB-SG - Allowing https traffic from anywhere on ports 80 and 433
2 APP-SG - Allowing TCP traffic from our ELB-SG on port 8080 and SSH on port 22 from our private IP
3 BACKEND-SG - TCP on port 11211 allows traffic from APP-SG to MemcacheD, TCP on 5672 allows traffic from APP-SG to RabbitmQ, TCP on 3306 allows traffic from APP-SG to MySQL and allows backend internal traffic for our backend server can communicate with each other

> Setup of backend instances and validation
In total is gonna be 4 instances db01, mc01, rmq01, and app01 all of them are gonna be launched by passing user data for the automatic setup of the services, and after being attached to the right security group, when the instance is ready we can check out the run user data by using the command ``curl http://169.254.169.254/latest/user-data`` this should output the script for service setup, also make sure that all services are running using systemctl status <service> and the internal traffic is happing ping all the machines, if get timeout to check if the ports are open in systemd firewall using ``ss -tunpl | grep 1121``

The only script that didn't work was the rabbitmq and was solved using the below commands: 
rabbitmq install amzlinux 2
yum-config-manager --save --setopt=erlang-solutions.skip_if_unavailable=true
yum install https://github.com/rabbitmq/erlang-rpm/releases/download/v23.3.4.11/erlang-23.3.4.11-1.el7.x86_64.rpm -y
yum install https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.9.13/rabbitmq-server-3.9.13-1.el7.noarch.rpm -y
sudo rabbitmq-plugins enable rabbitmq_management

> Rout 53 setup
The next thing is update the private IP of these three instances in Route 53 private DNS zones.

db01 172.31.55.103
mc01 172.31.61.55 
rmq01 172.31.14.184

So it's a simple A record name to IP mapping. Which will be used by our Tomcat ec2 instances, in our application.properties file, we will mention these names rather than mentioning IP addresses. So even if we replace the backend servers and give the same name. Then we don't need to make any changes to our application server.

Now the next step is to lunch and config our Tomcat ec2 instance, which we're gonna use for OS ubuntu server 18 since it is more simplified to setup the web server, as per a user-data script, we are using Tomcat 8, if we use ubuntu 20 or higher its need to install Tomcat 9

>Artifact setup 
So now we are gonna build our app artifact, for this is needed JDC and Maven. Our application.properties files are needed some updates, the first change is in the hostnames that we have set up on route 53 entries. Validate if the DNS records are pointing for the right ips, and if ports and hostnames are the same in application.properties as our DNS records, with all correct we can build the artifact running mvn install. Once the artifact is built we need to upload for an s3 bucket, for best practices, we will create an IAM user and attach s3FullAccess which will be used for authentication and upload of the artifact. after uploading, to download the artifact from our ec2 instance we will create a role on AMI, and attach it to the app01 instance, then we ssh on the app01 to check if the tomcat service is running and delete the ROOT directory which will have the default application, download our artifact and cp for /var/lib/tomcat8/webapps/ROOT.war then start tomcat service and check /ROOT/WEB-INF/classes/application.properties. Now let's check network flow with telnet running telnet db01.viniciussa.in 3306 we can also validate others backends servers with telnet 

> Load Balancer and DNS 
We are gonna create an LB but first, let's create a target group, first TG is the app we are gonna use port 8080, and health check in path /login in advanced setting override port for 8080 and threshold 2. Now create an ALB internet-facing, select the SG created earlier allowing 80 and 443 from anywhere, and add listener HTTPS 443 point for app-TG, once created copy de DNS name go to the DNS provider, and add a CNAME record, also make sure that your LB is in the same AZ of your instances 

> Configuring Auto Scaling 
The first thing to do is create an AMI  of the app instance, then create a Launch Configuration for the auto-scaling group, remember to select the same SG, instance type, keys, and IAM role. 
Now let's create an auto-scaling group selecting the launch configuration, and the same subnets, and attach it to the load balancer target group, so our instances will be automatically updated in this TG, select the health to be ELB so EC2 Auto Scaling automatically replaces instances that fail health checks, now let's specify the size of the Auto Scaling group by changing the desired minimum and maximum capacity, and for target tracking scaling policy I will be using average CPU utilization with the target value 80 so when our server passes the target value it will be automatically adding another instance to handle the demand, also we can setup notification using SNS Topic when an instance is launched, terminated, or fail to launch or terminate... review and create 

> Now let's review our web server workflow 
 As a user access the app through a URL that points to the load balancer endpoint with an HTTPS connection, you access your application load balancer endpoint the certificate is in ACM, and the application load balancer is in a security group that allows only 443 https traffic which then forwards the request to Tomcat ec2 instance on port 8080, which is in another Security Group.
 For the backend servers that had access with name, tt's private IP mapping is given in the private DNS zone, our backend servers are in one security group Memcache, Rabbitt MQ, and MySQL.
 And now, whenever we want, we can upload a new artifact to the S3 bucket and download it to our Tomcat ec2 instances, that's not such an efficient way of deploying an artifact, the right way of deploying an artifact is in a completely automated process, using CI/CD pipeline which I will configure in further process.
 If we want to create an auto-scaling group for Memcache, Rabbitt MQ, and MySQL we can just select our instance and attached it to the auto-scaling group, but there are better ways to do these things in AWS, instead of using an auto-scaling group and ec2 instances, we can use some PASS & SAAS services, which I'm going to do in the next project, refactoring our application stack migrating for AWS managing services.
 
 ![Captura de tela 2023-02-23 171312](https://user-images.githubusercontent.com/95035624/221019672-58c77548-67f0-4cba-8a08-cc7120a7f3d2.png)

![Captura de tela 2023-02-23 151112](https://user-images.githubusercontent.com/95035624/221019074-ea58fcb8-a3fa-4994-a23d-e1285bf65087.png)

 ![Captura de tela 202![Captura de tela 2023-02-23 151248](https://user-images.githubusercontent.com/95035624/221019104-d801b922-58ad-4af4-8e93-9de09ee0620c.png)
 
3-02-23 151146](https://user-images.githubusercontent.com/95035624/221019090-4efa9516-498b-450e-9324-87f935665b89.png)
 
![Captura de tela 2023-02-23 160940](https://user-images.githubusercontent.com/95035624/221019117-b9fa4076-7280-418a-a3b9-ffa57b0b4e41.png)
 
![Captura de tela 2023-02-23 161329](https://user-images.githubusercontent.com/95035624/221019122-0723f8f9-fe07-4882-9f44-cd416ce5d466.png)
