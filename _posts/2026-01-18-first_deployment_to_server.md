## A bit of Context 
Have you ever wondered what are the steps and tools needed to deploy a service to a cloud server? 
If yes, you are not alone. 

![Chihuahua stare](https://media.tenor.com/47KEJ7yaVtoAAAAM/chihuahua-stare-at.gif)
#### Here, I will be demonstrating the deployment of a simple streamlit app on a cloud server from Hetzner. I will be using Terraform to provision resources, Docker to containerize my web application, and ansible to automate the deployment. 

## Hetzner
1. The API token got generated through Hetzner User Interface to authenticate when a request made to the API through terraform provider
2. Install Hetzner CLI
   ```commandline
   sudo apt install hcloud-cli 
   hcloud version
   ```
3. Setting up the context (API token) is quite important to interact with Hetzner Cloud Services as shown here https://community.hetzner.com/tutorials/howto-hcloud-cli
	```commandline
    hcloud context --help 
    hcloud context create stocks-app
    ```
    You can change stocks-app for whatever name you want. This command will prompt to enter the API token. The output should show the context (stock-app in my case) is active 
4. To see all contexts created, 
    ```commandline
    hcloud context list
    ```
5. To choose the server type and image from Hetzner, I had to run the following command
    ```commandline
    hcloud image list | grep Ubuntu
    hcloud server-type list 
    ```

## Provisioning resources with Terraform
### Theory
I didn't know where to start but YouTube as usual is a great school. I found this YouTube video https://www.youtube.com/watch?v=bEXfPzoB4RE. The creator defines so many important concept in their video before even highlighting how to apply it. I didn't finish the video, but I got a good gist of what I need to know to start deploying a tool to a server. The concepts that I think everyone should know are the following: 
1. Localhost means the tool is running on my computer. My computer is a server duh
2. There are two ways to manage infrastructure:
   1. on prem (on premise): I could own (if I was rich enough) a physical server in a Data Center. Data Center is basically a storage of 100s of servers 
   2. on cloud: I rent a virtual machine, which runs on a physical server in a cloud provider's data center, from a cloud service provider such as AWS, Google Cloud, Microsoft Azure...etc
3. IaC (Infrastructure as Code) allows provisioning the infrastructure with configuration files, such as Terraform files. Thus, we avoid 
   1. using the UI of the cloud provider to create and manage the services needed such as S3, EC2, and ECR to deploy our desired tool
   2. complex scaling and automation 
   3. configuration drift which is the inconsistency in infrastructure across various data centers and regions on which the tool is deployed on 
4. The .tf configuration files provide the mean to transform the current state of the infrastructure to a desired state 
5. Cloud specific or dependent means the use of only one cloud service provider while Cloud Agnostic means the use of various cloud providers to provision desired infrastructure. Check this link for more information https://atlan.com/cloud-dependent-vs-cloud-agnostic/
6. Terraform uses the language HCL (HashiCorp Configuration Language) and follows declarative approach, which is defining a desired state then transforming the current state to the desired state 
7. The provider in terraform is the middle man that communicates between the config .tf files and the API of the cloud provider. It is a terraform plugin
8. Terraform registry is the place to check for the various providers
9. When executing 
    ```commandline
    terraform init
    ```
   The provider is downloaded from registry and terraform is started. The following metadata on the provider gets generated:
    1. .terraform directory
    2. .terraform.lock.hcl
10. Terraform workflow consists of:
    1. writing the config .tf files in declarative approach 
    2. planning, which is Terraform way of peer reviewing with us our desired state 
    3. applying the desired state to the current infrastructure state or destroying, which is deleting all the resources created
11. When executing: 
    ```commandline
    terraform apply 
    ```
    The resources will be provisioned as desired and the state .tf file will be updated to reflect the current state
12. The state .tf file is like a database created by terraform to keep track of all resources created, managed or deleted 
13. State drift is the mismatch between the current infrastructure and the .tf config file. For example, if a colleague deleted an EC2 instance from AWS UI and didn't update the config .tf file, we will end up with a state drift

### Application 
At the root of my project, I created a directory terraform. This directory contains: 
1. provider.tf which will communicate with cloud service provider Hetzner to provision the resources I need, such as the server. I added the API token as a sensitive variable. Setting sensitive to true will protect the variable from being exposed during logging, planning and applying or destroying. However, the API token will still appear in terraform.tfstate file. 
2. resources.tf contains the resources (server) I need from Hetzner
3. terraform.tfvars contains the Hetzner API token. I added this file to gitignore since it contains the secret https://developer.hashicorp.com/terraform/tutorials/configuration-language/sensitive-variables. Make sure terraform.tfvars is not tracked before adding it to gitignore by executing the following  
    ```commandline
    git rm --cached terraform/terraform.tfvars
    ```
**P.S. I should definitely consider a better way of securing the API token by using a backend or a vault** 

While using the project as the root path, execute the following
```commandline
cd terraform/
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply
```
#### Results 
Server on Hetzner is created and can be accessed by ssh to the root via IPv4 as shown below 
```commandline
ssh root@IPv4
```
To log out from the server locally, click on Ctrl+D 

#### Lock state 
I got myself in a lock state with terraform because I ran terraform plan without the flag -var-file. Instead of letting it finish, I promptly stopped the process by doing Ctrl+C. To overcome the lock state, I did the following:

1. force unlock and remove the lock state file generated due to lock state
    ```commandline
    terraform force-unlock the_lock_id
    rm -f .terraform.tfstate.lock.info 
    ```
2. I was still in a lock state, so I tried to kill processes with name terraform 
    ```commandline
    pkill -f terraform 
    ```
3. No luck. I tried to obtain detailed info especially the PID (process_ID) of all running process on the system related to terraform. I then performed immediate force kill process id 878 which maps to terraform plan in my case. I finally confirmed the process ID 878 is actually killed
    ```commandline
    ps aux | grep terraform 
    kill -9 878  
    ps aux | grep terraform 
    ```
It worked 🎉🎉🎉

## Containerize the tool with Docker 



docker build -t stocks_app/intro . (-t is tag and . is path to the docker file) 
docker run -p 8081:8501  stocks_app/intro (-p publish the port on host port 8081 and container port (the exposed one) 8501. I can choose whatever host port)
docker run -d -p 8501:8501  stocks_app/intro (-d detach and run the container in the background) 


