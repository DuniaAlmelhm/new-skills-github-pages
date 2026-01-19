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

1. IaC (Infrastructure as Code) tools allow provisioning the infrastructure with configuration files, such as Terraform files. Thus, we avoid 
   1. using the UI of the cloud provider to create and manage the services needed such as S3, EC2, and ECR to deploy our desired tool
   2. complex scaling and automation 
   3. configuration drift which is the inconsistency in infrastructure across various data centers and regions on which the tool is deployed on 
2. Terraform is an example of an IaC tool. It uses the language HCL (HashiCorp Configuration Language) and follows declarative approach, which is defining a desired state then transforming the current state to the desired state
3. There are two ways to manage infrastructure:
   1. on prem (on premise): I could own (if I was rich enough) a physical server in a Data Center. Data Center is basically a storage of 100s of servers 
   2. on cloud: I rent a virtual machine, which runs on a physical server in a cloud provider's data center, from a cloud service provider such as AWS, Google Cloud, Microsoft Azure...etc
4. Localhost means the tool is running on my computer. My computer is a server duh
5. The .tf configuration files provide the mean to transform the current state of the infrastructure to a desired state 
6. The provider in terraform is the middle man that communicates between the config .tf files and the API of the cloud provider. It is a terraform plugin
7. Terraform registry is the place to check for the various providers
8. Cloud specific or dependent means the use of only one cloud service provider while Cloud Agnostic means the use of various cloud providers to provision desired infrastructure. Check this link for more information https://atlan.com/cloud-dependent-vs-cloud-agnostic/
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
### Theory 
Docker, as everyone knows, is a platform that allows us to deploy, run, and manage applications in a containerized way. These containers contain the code, libraries, and dependencies needed to run the application as expected. The most cool thing about Docker is the ability to run different services on the same host due to this robust feature of isolated environment (container) for each service
- There are two docker objects: 
  1. Image: A template containing instructions for creating the docker container. This image is built from the dockerfile, which is a text file containing all the commands needed to build the image
  2. Container: It is the running instance of the image. I can create many containers from the same image. 
- Docker registry: storage of docker images 
- Docker client: It is docker CLI that enables us to execute docker commands and interact with Docker API 
- Docker API or Docker Host: It is the middleman between docker client and docker daemon 
- Docker daemon: It is the part of docker that does the heavy lifting, such as building images and running containers

Please refer to their official website https://docs.docker.com/get-started/docker-overview/ for more information 

### Application
At the root of my project, I created a dockerfile that does the following:
1. Specify the Python version 
2. Set up the desired working directory for my project inside the container
3. Execute a command to install the necessary libraries defined in requirements.txt file 
4. The Streamlit app inside the container is listening on port 8501 
5. Configure the container to run the following command 
    ``` commandline 
    streamlit run main.py
    ```
I ran the following commands locally to make sure I can build the image from the dockerfile, and I can run a container from this image
1. Build the image with name stocks_app/intro. The tag would be latest since it is not defined in the command. The . is the path to the dockerfile in the project 
    ```commandline
    docker build -t stocks_app/intro
    ```
2. Run the container with flag -p to publish the port. The service runs on host port 8081, and it is listening on port 8501 inside the container. I can choose whatever host port I want. See the documentation here https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/ for more information 
    ```commandline
    docker run -p 8081:8501  stocks_app/intro 
    ```
3. Run the container in detached mode while publishing the port 8501 as both host and container port. Detached mode means the application inside the container keeps on running in the background (even when terminal session is closed)
    ```commandline
    docker run -d -p 8501:8501  stocks_app/intro 
    ```

Docker part is done, and we are ready for next part 🚀

## Automate the deployment with Ansible 
### Theory
Ansible is an automation engine to automate provisioning of resources, deployment of services and orchestration. I am following the official documentation of ansible throughout my theory and application. Their documentation can be found here https://www.redhat.com/en/ansible-collaborative

### Application 
#### Installation 
I never worked with ansible before, so I had to install it. I am working on Windows. 
```commandline
pip install ansible
```

When I tried to install it on Windows, I ended up with this error 
```markdown
AttributeError: module 'os' has no attribute 'get_blocking'
```

According to this issue on Stack Overflow https://stackoverflow.com/questions/74701206/install-ansible-windows-machine, installing ansible on Windows without WSL is not supported. Thus, I am installing it on Ubuntu-22.04 running under WSL
```commandline
pip install ansible 
ansible --version 
```
#### Start ansible in my project
I executed the following commands at the root of my project: 
1. I am creating a directory called ansible.
2. I am storing in this directory the inventory file, which lists all servers (hosts) that I want to use to deploy my services on and be managed by ansible. I am listing the IPv4 of my Hetzner server. 
3. I then confirm my inventory 
4. I am also storing ansible playbooks. A playbook is a config yaml file that orchestrate the steps of a process on the host. For example, I am creating my first playbook file, this playbook will orchestrate the steps to install docker on my Hetzner server

```commandline
mkdir ansible
cd ansible
touch inventory.ini
ansible-inventory -i inventory.ini --list
touch playbook_install_docker.yaml
```

#### Ping
To verify there is a connection between ansible and my server and the presence of a python interpreter, I have to ping the host using the command 
```commandline
ansible myhosts -m ping -i inventory.ini
```
Breakdown of the command:
- ansible: tool I am using to execute the command 
- myhosts: list of all IP addresses defined in the file inventory
- -m ping: specifically calling module ping. The module communicates with the listed hosts under myhosts. If a successful connection is established, it will return **pong** for the host is reachable
- -i inventory.ini: it specifies the file to be used as a source for hosts 

This command resulted in this error 
<img src="/images/ping_error.png" >

This command has two issues: 
1. the username needs to be specified since it is different between the managed node (Hetzner server) and the control node (local system), so I added the flag -u
    ```commandline
    ansible myhosts -m ping -i inventory.ini -u root
    ```
2. Permission is denied: I am adding a flag --ask-pass, so I am prompted to enter ssh password 
    ```commandline
    ansible myhosts -m ping -i inventory.ini -u root --ask-pass
    ```
   This resulted in the following error 
    ```markdown
    "msg": "to use the 'ssh' connection type with passwords or pkcs11_provider, you must install the sshpass program"
    ```
    I ran the following to install sshpass on Ubuntu https://www.firsttiger.com/blogs/install-sshpass-program-on-macos/
    ```commandline
    sudo apt install sshpass
    ```

re-running this command 
```commandline
ansible myhosts -m ping -i inventory.ini -u root --ask-pass
```
and passing the ssh password, resulted in **SUCCESS and the path to python interpreter as well as PONG** In other words, the inventory has been successfully created 🎉🎉🎉

Checkout their documentation here https://docs.ansible.com/projects/ansible/latest/getting_started/get_started_inventory.html for building the inventory 

#### Playbook_install_docker 
In this playbook, I am attempting to install docker on my Hetzner server. Checkout the playbook since I am commenting on and adding relevant resources to almost every line for more details. However, I am going to highlight important concepts here:

- apt is package manager for Ubuntu. It is like pip for Python. It accesses repositories via HTTP protocol to download desired packages
- Docker uses HTTPS, which is HTTP Secure Protocol (HTTP over TLS). Thus, apt can't access docker repository. The following packages need to be installed to access docker repository via apt
  1. apt-transport-https is a package that enables apt to download via HTTPS 
  2. ca-certificates are files that verify the legitimacy of SSL/TLS certificates
  3. curl is a tool for making requests (HTTP or HTTPS) without the need of web browser 
- gnupg is a tool that creates private and public keys to secure data. It uses many encryption algorithms, such as RSA and SHA-2
- Installing docker engine can be done through 2 different packages: 
  1. docker.io: The vendor is the operating system. Thus, it is straight forward to install docker engine from it. However,I might not end up with most recent releases of dependencies and libraries since the provider is the operating system (ubuntu in my case)
  2. dokcer-ce: The vendor is Docker. It requires adding the Docker's GPG key via gnupg tool to apt keyrings and enabling apt to access HTTPS, so it requires more steps to install docker engine. However, I will have new releases of docker engine available sooner than with docker.io
- A GPG key consists of both private and public keys. The docker-ce packages will be verified via the public key, which is stored in the apt keyrings 
- To access docker repository via apt, the repository must be added to the **/etc/apt/sources.list.d** directory on apt  
- containerd.io is the repository for containerd, which is a container runtime daemon to run containers, such as docker and kubernetes containers 
- containerd is used by docker daemon (dockerd) to execute container commands. 
- To simplify the flow, the user interacts with docker CLI. Docker CLI (provided by the docker-ce-cli package) communicates with docker daemon socket. The docker daemon socket communicates with dockerd (provided by the docker-ce package). Dockerd communicates with containerd (provided by the containerd.io repository). containerd runs and manages the container. Checkout this documentation for more information on containerd and dockerd https://www.docker.com/blog/containerd-vs-docker/

I have used the following resource extensively to create the playbook and acquire more theory on docker https://community.hetzner.com/tutorials/howto-docker-install
I also got inspired by this author to automate the installation of docker with ansible https://alexhernandez.info/articles/infrastructure/how-to-install-docker-using-ansible/











