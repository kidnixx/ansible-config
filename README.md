# ANSIBLE CONFIGURATION MANAGEMENT
Ansible Client as a Jump Server (Bastion Host)
A Jump Server (sometimes also referred as Bastion Host) is an intermediary server through which access to internal network can be provided. 
If you think about the current architecture you are working on, ideally, the webservers would be inside a secured network which cannot be reached directly from the Internet. That means, even DevOps engineers cannot SSH into the Web servers directly and can only access it through a Jump Server – it provide better security and reduces attack surface.

On the diagram below the Virtual Private Network (VPC) is divided into two subnets – Public subnet has public IP addresses and Private subnet is only reachable by private IP addresses.
![6033](https://user-images.githubusercontent.com/85270361/210153615-ea6cf398-05d3-45d0-9ea4-6daffac7fa4c.PNG)

Steps:

SETUP ALL THE INSTANCES as shown below.
Jenkins-ansible has ubuntu image
Rest is RHEL based servers
![Screenshot 2023-03-22 105957](https://user-images.githubusercontent.com/110480181/227113900-651d1dc8-f56b-4612-9230-d86901c20ddf.jpg)
Every instance has a Security Group Inbound rule as shown below.
![Screenshot 2023-03-23 111618](https://user-images.githubusercontent.com/110480181/227114687-99a4b611-1099-4097-a5ea-5aeb6128fbf1.jpg)
Outbound is all IPV4 Traffic

INSTALL AND CONFIGURE ANSIBLE ON jenkins ansible EC2 INSTANCE as shown below.

```
sudo apt update

sudo apt install ansible
```

INSTALL AND CONFIGURE jenkins on jenkins ansible EC2 INSTANCE as shown below.

Install JDK (since Jenkins is a Java-based application)
```
sudo apt update
sudo apt install default-jdk-headless
```
Install Jenkins
```
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt-get install jenkins
```
Make sure Jenkins is up and running
```
sudo systemctl status jenkins
```
From your browser access jenkins using http://jenkins ansible server-public-ip:8080. This will bring you to the login page.

![Screenshot 2023-03-22 110105](https://user-images.githubusercontent.com/110480181/227116005-78fa49ef-784f-46df-a245-848ad9604b34.jpg)

Configure Jenkins build job to save the repository content every time you change it

- Create a new Freestyle project ansible in Jenkins and point it to the github repository.
- Configure Webhook in GitHub and set webhook to trigger ansible build.
- Configure a Post-build job to save all (**) files

![6035](https://user-images.githubusercontent.com/85270361/210153832-75e74f67-0654-4fc1-bcdd-08c3d5a8fa76.PNG)

Setup VSCODE Editor. Setup Remote Development Extension.
![Screenshot 2023-03-22 125344](https://user-images.githubusercontent.com/110480181/227116626-9ba8cac6-882e-47de-8d25-0f033e2bfdd1.jpg)

In VSCode editor, Clone down the github repo to the Jenkins-Ansible instance

```
git clone <github repo link>
```
SETTING UP ANSIBLE

1.In the GitHub repository, create a new branch that will be used for development of a new feature.
2. Checkout the newly created feature branch to the local machine and start building the code and directory structure
3. Create a directory and name it playbooks – it will be used to store all the playbook files.
4. Create a directory and name it inventory – it will be used to keep the hosts organised.
5. Within the playbooks folder, create the first playbook, and name it common.yml
6. Within the inventory folder, create an inventory file (.yml) for each environment (Development, Staging Testing and Production) as dev, staging, uat, and prod respectively.
7. Set up an Ansible Inventory
An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate.
ssh into the Jenkins-Ansible server using ssh-agent

```
ssh -A ubuntu@public-ip
```

Update the inventory/dev.yml file with this snippet of code:

```
[nfs]
<NFS-Server-Private-IP-Address> ansible_ssh_user='ec2-user'

[webservers]
<Web-Server1-Private-IP-Address> ansible_ssh_user='ec2-user'
<Web-Server2-Private-IP-Address> ansible_ssh_user='ec2-user'

[db]
<Database-Private-IP-Address> ansible_ssh_user='ec2-user' 

[lb]
<Load-Balancer-Private-IP-Address> ansible_ssh_user='ubuntu'
```
8.Create a Common Playbook
It is time to start giving Ansible the instructions on what you needs to be performed on all servers listed in inventory/dev.
Update the playbooks/common.yml file with following code:

```
---
- name: update web, nfs and db servers
  hosts: webservers, nfs, db
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

- name: update LB server
  hosts: lb
  remote_user: ubuntu
  become: yes
  become_user: root
  tasks:
    - name: Update apt repo
      apt: 
        update_cache: yes

    - name: ensure wireshark is at the latest version
      apt:
        name: wireshark
        state: latest
```
Update GIT with the latest code

Once the code changes appear in master branch – Jenkins will do its job and save all the files (build artifacts) to 
/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/ directory on Jenkins-Ansible server.
![Screenshot 2023-03-22 110046](https://user-images.githubusercontent.com/110480181/227116909-01c15521-6b23-4f9f-bad9-734ee1abc7a7.jpg)
![Screenshot 2023-03-22 105957](https://user-images.githubusercontent.com/110480181/227117170-d8e7186b-a079-4700-ba32-5fa1047cc90c.jpg)
![Screenshot 2023-03-22 123800](https://user-images.githubusercontent.com/110480181/227117307-52caf77e-c167-45cf-acde-cf76f8d45d96.jpg)

9.Run first Ansible test


Now, it is time to execute ansible-playbook command and verify if the playbook actually works:

```
cd ansible-config-mgt
```

```
ansible-playbook -i inventory/dev.yml playbooks/common.yml
```
![Screenshot 2023-03-23 103934](https://user-images.githubusercontent.com/110480181/227117418-7065143d-2a00-46bb-bde8-e6a9310e0b41.jpg)
![Screenshot 2023-03-23 104247](https://user-images.githubusercontent.com/110480181/227117443-5b68a69b-56d5-49e5-b745-ecc4811438f5.jpg)


The updated Ansible architecture now looks like this:
![6038](https://user-images.githubusercontent.com/85270361/210154593-092a4ee2-ab8b-4212-a260-8845c3f8693a.PNG)
