#LIFT AND SHIFT WEB APPLICATION WORKLOAD 

The objective of this projects is to train how to run application workload on AWS cloud by using lift and shift strategy and simulate real world client compliance 

We have, application services, which are running on physical or virtual machines, in order to solve problems like complex managemenent, scale up/Down complexity, upfront capEX and regular operation the app wil be shift to cloud, with this we will have flexibility, payasUgo, automation and easy Infra Management.

**AWS SERVICES** 
- EC2 INSTANCES > VM for tomcat, rabbitmq, memcache, mysql 
- ELB > NGINX lb replacement 
- AUTOSCALING > automation for vm scaling 
- S3/EFS Storage > shared storage 
- ROUTE 53 > private dns server 
- ACM > certificate manager for https 

*COMPLIANCE** 
- FLexible Infra
- No upfront cost
- Mordenize Effectively 


**SETUP** 
> security group & Key Pairs: 
So first we are gonna setup Security Groups and rules, in total is going to be 3 SG 
1 ELB-SG - Allowing https trafic from anywhere on port 80 and 433
2 APP-SG - Allowing TCP traffic from our ELB-SG on port 8080 and SSH on port 22 from our private IP
3 BACKEND-SG - TCP on por 11211 allowing traffic from APP-SG to MemcacheD, TCP on 5672 allowing traffic from APP-SG to RabbitmQ, TCP on 3306 allowing traffic from APP-SG to MySQL and allow backend internal traffic for our backend server can communicate with earch other

> Setup of backend instances and validation
In total is gonna be 4 instances db01, mc01, rmq01, app01 all them is gonna be launch passing user data for automactally setup of the services and after atached to the right security group, wen the instance is ready we can checkout the runned user data by ussing the command ``curl http://169.254.169.254/latest/user-data`` this should output the script for service setup, also make sure that all services is running using systemctl status <service> and the internal traffic is happing ping all the machines, if get timeou check if the ports are open in systemd firewall using ``ss -tunpl | grep 1121``

The only script that didn't work was the rabbitmq and was solved using the bellow commands: 
rabbitmq install amzlinux 2
yum-config-manager --save --setopt=erlang-solutions.skip_if_unavailable=true
yum install https://github.com/rabbitmq/erlang-rpm/releases/download/v23.3.4.11/erlang-23.3.4.11-1.el7.x86_64.rpm -y
yum install https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.9.13/rabbitmq-server-3.9.13-1.el7.noarch.rpm -y
sudo rabbitmq-plugins enable rabbitmq_management

> Rout 53 setup
The next thing is update private IP of these three instances in Route 53 private DNS zones.

db01 172.31.55.103
mc01 172.31.61.55 
rmq01 172.31.14.184

So it's a simple A record name to IP mapping.Which will be used by our Tomcat ec2 instances, in our application.properties file, we will mention these names rather than mentioning IP addresses. So even if we replace the backend servers and we give the same name.Then we really don't need to make any change in our application server.

Now the next step is lunch and config our Tomcat ec2 instance, we're gonna use for OS ubuntu server 18 since it is more simplified to setup the web server, as per a user-data script, we are using Tomcat 8, if we use ubuntu 20 or higher its need to install Tomcat 9

>Artifact setup 
So now we are gonna build our app artifact, for this is needed JDC and Maven. in our application.properties files is needed some updates, the first change is in the hostnames that we have setup on rout 53 entries. Validate if the dns records are pointing for the right ips, and if ports, hostnames are the same in application.properties as our dns records, with all correct we can build the artifact runing mvn install. Once the artifact is builded we need to upload for a s3 bucket, for best practices we will creat a iam user and attach s3FullAccess which will be used for authentication and upload of the artifact. after uploaded, in order to download the artifact from our ec2 instance we will create a role on ami, and attach to the app01 instance, then we ssh on the app01 check if the tomcat service is running and delete root directory which will have the default application, download our artifact and cp for /var/lib/tomcat8/webapps/ROOT.war then start tomcat service and check /ROOT/WEB-INF/classes/application.properties. Now lets check netowrk flow with telnet runing telnet db01.viniciussa.in 3306 we can also validate others backends servers with telnet 

> Load Balancer and DNS 
We are gonna creat a LB but first lets creat a target group, first TG is the app we are gonna use port 8080 and helth check in path /login in advanced setting override port for 8080 and threshold 2. Now create a ALB internet-facing, select the SG created earlier alowwing 80 and 443 from anywhere, add listener HTTPS 443 point for app-TG, once created copy de DNS name go to DNS provider and add a cname record, also make sure that you LB are in the same AZ of your instances 

> Configuring Auto Scaling 
Firs thing to do is create an AMI  of the app instance, then creat a Launch Configuration for the auto scaling group, remeber to select the sames SG, instance type, keys and iam role. 
Now lets create auto escaling group selecting the launch configuration, same subnets and attach to the load balancer target group, so our instances will be automatically updated in this TG, select the healtch to be ELB so EC2 Auto Scaling automatically replaces instances that fail health checks, now lets specify the size of the Auto Scaling group by changing the desired minimum and maximum capacity, and for target tracking scaling policy i will be using average CPU utilization with the target value 80 so wen our server pass the target value it will be automatically add another instance to andle the demand, also we can setup notification using SNS Topic wen an instance is launched, terminated, or fail to launc or terminate, review and creat 

> Now lets review ou web server work flow 
 As a user access the app through a URL that points to the load balancer endpoint with https connection, you access your application load balancer endpoint the certificate is in ACM, application load balancer is in a security group that allows only 443 https traffic which then forwards the request to Tomcat ec2 instance on port 8080, which is in another Security Group.
 For the backends servers that had access with name, tt's private IP mapping is given in the private dns zone, our backend servers are in one security group Memcache, Rabbitt MQ and MySQL.
 And now, whenever we want, we can upload a new artifact to S3 bucket and download it to our Tomcat ec2 instances, that's not such an efficient way of deploying artifact, the right way of deploying artifact is in a completely automated process, using CI/CD pipeline wich i will configure in futher process.
 If we want to creat auto scaling group for Memcache, Rabbitt MQ and MySQL we can just select our instance and attached to auto scaling group, but there are really better ways to do these things in AWS, instead of using auto scaling group and ec2 instances, we can use some PASS & SAAS services, which i'm going to do in the next project, refactoring our application stack migrating for aws managing services.
 
 ![Captura de tela 2023-02-23 171312](https://user-images.githubusercontent.com/95035624/221019672-58c77548-67f0-4cba-8a08-cc7120a7f3d2.png)

![Captura de tela 2023-02-23 151112](https://user-images.githubusercontent.com/95035624/221019074-ea58fcb8-a3fa-4994-a23d-e1285bf65087.png)

 ![Captura de tela 202![Captura de tela 2023-02-23 151248](https://user-images.githubusercontent.com/95035624/221019104-d801b922-58ad-4af4-8e93-9de09ee0620c.png)
 
3-02-23 151146](https://user-images.githubusercontent.com/95035624/221019090-4efa9516-498b-450e-9324-87f935665b89.png)
 
![Captura de tela 2023-02-23 160940](https://user-images.githubusercontent.com/95035624/221019117-b9fa4076-7280-418a-a3b9-ffa57b0b4e41.png)
 
![Captura de tela 2023-02-23 161329](https://user-images.githubusercontent.com/95035624/221019122-0723f8f9-fe07-4882-9f44-cd416ce5d466.png)
