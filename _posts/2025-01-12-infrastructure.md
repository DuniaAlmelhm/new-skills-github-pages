---
title: "Infrastructure first draft"
date: 2025-01-12
---

"Infrastructure as Code (IaC) 
-	Infrastructure includes servers, network configuration (any services that talking to each other internally/externally like API addresses, routing tables, DNS “Domain Name system”), storage (like snowflake connection), monitoring  
-	Localhost means the application is running on my computer as a server 
-	Deploying the application means running it on a cloud server 
-	Manage infrastructure physically or on-prem such as Data Center which are 100s of physical servers. Problems: expensive maintenance, scalability issues, low flexibility for re-configuration 
-	Manage infrastructure on cloud: Intelcom rents servers from AWS to run applications/services. 
-	On cloud: visit the UI of AWS and configure services needed to run the application, such as the EC2 instances/virtual machines (virtualized servers running on physical servers in AWS data center) you need. Scale up and down on demand 
-	AWS region means the AWS data center available in the area 
-	Distributing the deployment on different regions is a good idea so the service keeps on running if one of the data centers is upgrading or is down => reliable 
-	“Pay-as-you-go”
-	AWS cloud watch for monitoring every component of the infrastructure 
-	When we add more regions with the same infrastructure, we have to manually configure each region 
-	AWS RDS to run MySQL  
-	EC2 instances act individually. Thus, we need to configure each instance to communicate properly with the other components of the infrastructure, such as RDS instances or S3 buckets 
-	Configuration drift: when the infrastructure is inconsistent in the various data centers/regions on which the app/service is distributed 
-	IaC allows managing the infrastructure with configuration files (Terraform) instead of using UI (AWS UI) to avoid manual intervention, configuration drift, and complex scaling 
-	IaC: code to transform from current state to desired state of infrastructure => consistency, automation (save time/effort), code is written in configuration files that can be shared through version control 
-	Provision is creation of resources and components 
-	Cloud Specific vs Cloud Agnostic 
-	Cloud Agnostic: infrastructure on various cloud providers such as Terraform/OpenTofu

Resource: https://www.youtube.com/watch?v=bEXfPzoB4RE
"
