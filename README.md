## Optimizing Scalability and Reliability for a Car Rental WordPress Application.

## **Problem Statement**

In response to the operational challenges faced by our car rental business, we have deployed a WordPress-based application on Amazon Web Services (AWS) EC2 instances. This application serves as a vital tool for optimizing the booking and management of rental vehicles. The current infrastructure utilizes a 3-tier architecture, encompassing a load balancer, Route 53 for DNS management, target groups, and various supporting resources. Despite its functionality, our application faces scalability, reliability, and performance issues. Furthermore, we are exploring the integration of Amazon Elastic File System (EFS) to elevate our data storage and sharing capabilities. In light of these challenges, we hereby request our cloud engineer to redesign our architecture, aligning it with the AWS Well-Architected Framework and its six key pillars.


## **Challenges and Objectives:**

  - Operational Excellence: Our primary challenge lies in ensuring the application's continuous availability, reducing downtime, and optimizing resource utilization. To achieve operational excellence, we need to enhance automation, monitoring, and incident response mechanisms.
  
  - Security: As a critical aspect, security needs to be a top priority. We must bolster our application's security posture by implementing AWS Identity and Access Management (IAM) policies, encryption mechanisms, and threat detection systems.
  
- Reliability: To address the reliability issues faced by our application, we should focus on reducing single points of failure, implementing backup and recovery strategies, and enhancing the overall system's fault tolerance.
  
- Performance Efficiency: Improving the performance of our application is vital for delivering a seamless user experience. This entails optimizing resource allocation, leveraging auto-scaling capabilities, and tuning database and application configurations for maximum efficiency.
  
- Cost Optimization: Cost management is crucial to ensure the efficient use of AWS resources. By right-sizing instances, implementing cost monitoring, and adopting serverless architecture where feasible, we can achieve substantial cost savings.
  
- Scalability: To accommodate increased demand, our architecture should seamlessly scale both horizontally and vertically. Implementing auto-scaling groups and distributing traffic effectively are key aspects of achieving this scalability.


## **Architecture Overview**

  - Presentation Layer: This layer serves as the interface between users and the application, functioning as the primary entry point for user interactions. To enhance reliability and performance, it's supported by an Application Load Balancer (ALB). The ALB efficiently distributes incoming traffic across multiple web server instances, ensuring consistent access to the application.

  - Application Layer: This layer plays a role in hosting the web servers responsible for delivering the static components of the application. Using EC2 instances, I deploy these web servers for optimal performance and scalability. To safeguard against unexpected traffic spikes or server failures, I employ an Auto Scaling Group, which dynamically adjusts the number of EC2 instances based on demand.
    In same vein, as our architecture presents, If you need a shared file system for multiple instances running your WordPress application, you can use Amazon EFS to store WordPress core files or plugins/themes that need to be accessed by multiple instances.

  - RDS is used to host the WordPress database. It provides a managed MySQL database, which makes it easier to set up, operate, and scale a relational database. This is where the WordPress content, settings, and user data are stored. Alternatively, if the need arises, we can employ Amazon Simple Storage Service (S3) to store static files securely. This adaptability ensures that the application can evolve to meet changing requirements without major architectural modifications.

## **Architecture**

<p align="center">
<img src="https://i.imgur.com/7b8RQNO.png" height="65%" width="75%" alt="RDP event fail logs to iP Geographic information"/>
</p>

## **Replicating the Architecture: Deployment Steps**

1. Identity and Access Management (IAM):

    - Create IAM Users: Create IAM users for your team members with appropriate permissions.
  
    - Implement MFA: Enforce Multi-Factor Authentication (MFA) for IAM users for enhanced security
  
    - Define IAM Policies: Create custom IAM policies that grant necessary permissions for EC2, VPC, S3, EFS, and other resources.
  
    - IAM Password Policies: Configure password policies to enforce strong password requirements.
   
2. Virtual Private Cloud (VPC)

    - Create VPCs and configure the necessary subnets. Also, configure the NAT and Internet Gateways and associate them with their respective route tables.
       
  
3. Security Group Configuration:

    - ALB Security Group: Create a Security Group for the Application Load Balancer (ALB) to allow inbound HTTP and HTTPS traffic from 0.0.0.0/0.

    - Web Server Security Group: Configure a Security Group for your EC2 instances to allow inbound HTTP and HTTPS traffic from the ALB Security Group as well as SSH from "My IP".

    - Database Security Group: Allow inbound traffic for MYSQL/Aurora on port 3306 from the webserver security group.

    - Elastic File System Security Group: Allow inbound traffic of NFS protocol on port 2049 from webserver security group and EFS security group (tip: ensure to create your security first, then go back to the SG and attach the EFS SG). Finally, add SSH from SSH SG.

4. Create a Database

    - Here, ensure to use the latest MYSQL engine, burstable database instance storage (check to include previous generation i.e., db.t2.micro), and pay attention to your database username and password. In subsequent configurations in ec2 server (wp-config.php): you will revert to your database configurations to get information like:

    - DB name: wordpress_Database_v1 DB 

    - instance ID: wordpress-db-1 

    - Endpoint: wordpress-db-1.clfp8kgim5mc.eu-central-1.rds.amazonaws.com


5. Create an EFS and Mount Target

    - Create an EFS and customize it, then configure the mount target;
    
    - Select the VPC where your EC2 instances are located.
    
    - Select the Availability Zone (AZ) for the first mount target (Private App Subnet AZ 1 & Private App Subnet AZ 2).
    
    - Provide a Security Group that allows inbound NFS traffic from the EC2 instances.
  
    - Optionally, you can specify an IP address range for inbound access, or you can leave it as "0.0.0.0/0" to allow all incoming connections.
  
    - Click the "Create mount target" button.
  
    - Once created, click attach and copy the mount target DNS info. This will used in our ec2 setup server.

6. Create an EC2 setup server. Then ssh into the instance and include the following

    ```
    # 1. Mount EFS (Make sure EFS is properly configured)
    sudo su
    yum update -y
    mkdir -p /var/www/html
    sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0d845ea19fc5c7c9d.efs.eu-central-1.amazonaws.com:/ /var/www/html
    
    # 2. Install Apache
    sudo yum install -y httpd httpd-tools mod_ssl
    sudo systemctl enable httpd
    sudo systemctl start httpd
    
    # 3. Install PHP 7.4
    sudo amazon-linux-extras enable php7.4
    sudo yum clean metadata
    sudo yum install php php-common php-pear -y
    sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
    
    # 4. Install MySQL (consider using a more recent version)
    sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
    sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
    sudo yum install mysql-community-server -y
    sudo systemctl enable mysqld
    sudo systemctl start mysqld
    
    # 5. Set permissions
    sudo usermod -a -G apache ec2-user
    sudo chown -R ec2-user:apache /var/www
    sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
    sudo find /var/www -type f -exec sudo chmod 0664 {} \;
    sudo chown apache:apache -R /var/www/html
    
    # 6. Download WordPress files
    wget https://wordpress.org/latest.tar.gz
    tar -xzf latest.tar.gz
    cp -r wordpress/* /var/www/html/
    
    # 7. Create the wp-config.php file
    cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
    
    # 8. vim the wp-config.php file
    vim /var/www/html/wp-config.php
    
    # 9. Restart Apache
    sudo systemctl restart httpd

    ```

Purpose of the Setup Server Above:

The setup server serves as the core configuration hub for our AWS-hosted WordPress site. It orchestrates the installation of essential components like Apache, PHP, and MySQL for database operations. Moreover, it creates an Amazon EFS volume, enabling shared access to WordPress content among multiple EC2 instances in our chosen Availability Zones (AZ 1 and AZ 2).

Key Functions:

    - Installs and configures Apache, PHP, and MySQL for seamless WordPress operation.
      
    - Manages database connections and updates configurations in wp-config.php.
      
    - Establishes an Amazon EFS volume mounted on the HTML directory of web servers.
      
    - Facilitates content sharing across EC2 instances in AZ 1 and AZ 2, mirroring the setup of Web Server 1 and Web Server 2. This design ensures high availability and scalability for our WordPress site.


7. Configure WordPress site with your information

    - Prior to the configuration of WordPress, use your ec2 public IP address to direct you to the WordPress site.

    - Configure WordPress with private information and make sure you access the WordPress admin page. 

<p align="center">
<img src="https://i.imgur.com/bHfKtvZ.png" height="65%" width="75%" alt="RDP event fail logs to iP Geographic information"/>
</p>

8. Webserver (1 & 2), Target Group, and Load Balancer

Now that we are sure that our set-up server is running, let's create two ec2 servers and position them in our private app subnet AZ 1 and private app subnet AZ 2 respectively. Then include the following user data scripts: 
Steps
    
  1)	Create a webserver in private app subnet AZ 1 and use the user data. 
    
  2)	Then create another webserver in private app subnet AZ 2 and use the user data.

Note: Ensure to update the mount target ID: fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com

  ```
  #!/bin/bash
  yum update -y
  sudo yum install -y httpd httpd-tools mod_ssl
  sudo systemctl enable httpd 
  sudo systemctl start httpd
  sudo amazon-linux-extras enable php7.4
  sudo yum clean metadata
  sudo yum install php php-common php-pear -y
  sudo yum install php-{cgi,curl,mbstring,gd,mysqlnd,gettext,json,xml,fpm,intl,zip} -y
  sudo rpm -Uvh https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
  sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2022
  sudo yum install mysql-community-server -y
  sudo systemctl enable mysqld
  sudo systemctl start mysqld
  echo "fs-03c9b3354880b36a6.efs.us-east-1.amazonaws.com:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
  mount -a
  chown apache:apache -R /var/www/html
  sudo service httpd restart
  ```

Target group

  - Create a target group and include the respective Webservers. Please do not include the setup server.

Application Load Balancer: 

  - ALB Setup: Configure the Application Load Balancer (ALB) and associate it with the ALB Security Group. Ensure it's connected to the appropriate public subnets and target group of our webservers. Once the ALB is finished provisioning, do the following:
  
  •	Paste the DNS in a web browser (wordpress-lb-1257114690.eu-central-1.elb.amazonaws.com)
    
  •	Add the following at the end of the DNS: wordpress-lb-1257114690.eu-central-1.elb.amazonaws.com/wp-admin
    
  •	It will take you to the wordpress admin page. In the admin page,
    
  •	Go to settings, update the already existing IP address to the ALB DNS address: http:// wordpress-lb-1257114690.eu-central-1.elb.amazonaws.com
    
  •	Then save and exit. 


3. DNS and SSL/TLS Certificate Setup:

    - Route 53 Configuration: Create a hosted zone on Route 53 to manage the DNS records for your application. Configure DNS records like A and CNAME records as needed.

    - SSL/TLS Certificate: Obtain an SSL/TLS certificate using AWS Certificate Manager for your domain or subdomain to enable secure HTTPS traffic. Ensure the certificate is associated with the ALB.
  
4. Application Load Balancer (ALB):

    - ALB Setup: Configure the Application Load Balancer (ALB) and associate it with the ALB Security Group. Ensure it's connected to the appropriate public subnets.

    - Listeners: Set up ALB listeners for HTTP (port 80) and HTTPS (port 443) to forward traffic to backend EC2 instances. Configure appropriate SSL policies for HTTPS.
  
5. Launch Template Creation:

    - Launch Template: Create a Launch Template specifying the required instance specifications for your EC2 instances. Include user data or user scripts for configuring instances upon launch.

      ```
      #!/bin/bash
      sudo su
      yum update -y
      yum install -y httpd
      cd /var/www/html
      wget https://github.com/Ahmednas211/jupiter-zip-file/raw/main/jupiter-main.zip
      unzip jupiter-main.zip
      cp -r jupiter-main/* /var/www/html
      rm -rf jupiter-main jupiter-main.zip
      systemctl start httpd
      systemctl enable httpd
      
      ```
  
6. Auto Scaling Group (ASG):

    - ASG Configuration: Create an Auto Scaling Group (ASG) using the Launch Template. Configure scaling policies, instance count, and health checks.
      
    - Target Group: Set up a Target Group and configure it as the target for the ASG. This ensures that traffic is distributed correctly among instances.
  
7. ALB Listener Configuration:

    - Listener Rules: Define rules in the ALB listener to route traffic based on hostnames, paths, or other criteria. Ensure that traffic is appropriately directed to backend instances.
  
8. DNS Configuration:

    - Route 53 Updates: Update DNS records in Route 53 to point to the ALB's DNS name. Implement health checks in Route 53 to monitor the availability of your application.

9. Testing and Monitoring:

    - Testing: After DNS records have propagated, thoroughly test your application using the domain name (HTTPS) to verify the deployment's success.
      
    - Monitoring: Implement cloud monitoring and logging solutions (e.g., AWS CloudWatch) to monitor the health and performance of your infrastructure and applications.
  
10. Backup and Disaster Recovery:

    - Implement backup and disaster recovery mechanisms to ensure data and application availability in case of failures.
   
11. Security and Compliance:

    - Follow AWS security best practices, implement IAM roles and policies, and consider compliance requirements specific to your application.
   
12. Documentation:

    - Maintain detailed documentation of the deployment process, infrastructure architecture, and any changes made over time.

## **A Screenshot of Landing Page Static Website**

<p align="center">
<img src="https://i.imgur.com/eUP7W7a.png" height="75%" width="95%" alt="RDP event fail logs to iP Geographic information"/>
</p>

<p align="center">
<img src="https://i.imgur.com/ZHRd67Q.png" height="75%" width="95%" alt="RDP event fail logs to iP Geographic information"/>
</p>

**Congratulations! You've Successfuly Hosted a Static Jupiter Website :)**
