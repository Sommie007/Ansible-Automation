## **ANSIBLE CONFIGURATION MANAGEMENT**
--- 

Ansible Client as a Jump Server (Bastion Host)

A Jump Server (sometimes also referred as Bastion Host) is an intermediary server through which access to internal network can be provided. If you think about the current architecture you are working on, ideally, the webservers would be inside a secured network which cannot be reached directly from the Internet. That means, even DevOps engineers cannot SSH into the Web servers directly and can only access it through a Jump Server – it provide better security and reduces attack surface.

On the diagram below the Virtual Private Network (VPC) is divided into two subnets – Public subnet has public IP addresses and Private subnet is only reachable by private IP addresses.

![Alt text](Ansible-automation/projectdisplay.png)

## **Step 1 - INSTALL AND CONFIGURE ANSIBLE ON EC2 INSTANCE**


1. Update Name tag on your Jenkins EC2 Instance to `Jenkins-Ansible`. We will use this server to run playbooks.

2. In your GitHub account create a new repository and name it `ansible-config-mgt`.

![Alt text](Ansible-automation/newrepo.png)

3. Install Ansible

**`sudo apt update`**

**`sudo apt install ansible -y`**

![Alt text](Ansible-automation/ansible-install.png)


Check your Ansible version by running **`ansible --version`**

![Alt text](Ansible-automation/ansible-version.png)

Install jekins dependent openjdk using this command **`sudo apt-get install fontconfig openjdk-17-jre -y`** first, then install jenkins using this command 

```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
```
confirm that jekins is up and running using this command **`sudo systemctl status jenkins.service`**

![Alt text](Ansible-automation/jenkins-status.png)

jekins default port is 8080, remember to permit it on the security group for ingress traffic

![Alt text](Ansible-automation/jenkinspermission.png)

then browse the public Ip using port 8080 that was permmited, page below will be displayed.

![Alt text](Ansible-automation/browseIPjenkins.png)

cat the directory **`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`**
 displayed to access admin password

then sign into jekins.


4. Configure Jenkins build job to save your repository content every time you change it – this will solidify your Jenkins configuration skills.


Create a new Freestyle project ansible in Jenkins and point it to your ‘ansible-config-mgt’ repository.

Configure Webhook in GitHub and set webhook to trigger ansible build.

![Alt text](Ansible-automation/webhook.png)



Configure a Post-build job to save all (**) files.


![Alt text](Ansible-automation/Jenkinswebhookconfig.png)



5. Test your setup by making some change in README.MD file in master branch and make sure that builds starts automatically and Jenkins saves the files (build artifacts) in following folder

```
ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/
```
![Alt text](Ansible-automation/checkbuildnumber.png)

Note: Trigger Jenkins project execution only for /main (main) branch.

Now your setup will look like this:


![Alt text](Ansible-automation/projectsetup.png)


**Tip: Every time you stop/start your `Jenkins-Ansible server` – you have to reconfigure GitHub webhook to a new IP address, in order to avoid it, it makes sense to allocate an Elastic IP to your `Jenkins-Ansible server` (you have done it before to your LB server in Project 10). Note that Elastic IP is free only when it is being allocated to an EC2 Instance, so do not forget to release Elastic IP once you terminate your EC2 Instance.**


## **Step 2 – Prepare your development environment using Visual Studio Code**

1. First part of ‘DevOps’ is ‘Dev’, which means you will require to write some codes and you shall have proper tools that will make your coding and debugging comfortable – you need an Integrated development environment (IDE) or Source-code Editor. There is a plethora of different IDEs and Source-code Editors for different languages with their own advantages and drawbacks, you can choose whichever you are comfortable with, but we recommend one free and universal editor that will fully satisfy your needs – Visual Studio Code (VSC).

2. After you have successfully installed VSC, configure it to connect to your newly created GitHub repository.

3. Clone down your ansible-config-mgt repo to your Jenkins-Ansible instance

```
git clone <ansible-config-mgt repo link>
```

![Alt text](<Ansible-automation/git clone.png>)


## **Step 3 - BEGIN ANSIBLE DEVELOPMENT**


1. In your ansible-config-mgt GitHub repository, create a new branch that will be used for development of a new feature.

**Tip: Give your branches descriptive and comprehensive names, for example, if you use Jira or Trello as a project management tool – include ticket number (e.g. PRJ-145) in the name of your branch and add a topic and a brief description what this branch is about – a bugfix, hotfix, feature, release (e.g. feature/prj-145-lvm)**


2. Checkout the newly created feature branch to your local machine and start building your code and directory structure

3. Create a directory and name it `playbooks` – it will be used to store all your playbook files.

4. Create a directory and name it `inventory` – it will be used to keep your hosts organised.


5. Within the playbooks folder, create your first playbook, and name it common.yml

6. Within the inventory folder, create an inventory file (.yml) for each environment (Development, Staging Testing and Production) `dev`, `staging`, `uat`, and `prod` respectively.


## **Step 4 – Set up an Ansible Inventory**


An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate. Since our intention is to execute Linux commands on remote hosts, and ensure that it is the intended configuration on a particular server that occurs. It is important to have a way to organize our hosts in such an Inventory.


Save below inventory structure in the `inventory/dev` file to start configuring your development servers. Ensure to replace the IP addresses according to your own setup.


Note: Ansible uses TCP port 22 by default, which means it needs to `ssh` into target servers from `Jenkins-Ansible` host – for this you can implement the concept of ssh-agent. Now you need to import your key into `ssh-agent`:

[set up open ssh on Windows]  https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse?tabs=powershell

or 

If you want to use the ssh-keygen, then run it using **`ssh-keygen -t ed255519`** to generate a public key, then add it using **`echo "(sshkey)" >> ~/.ssh/authorized_key `**, Do this for all the servers you want ansible to connect to automatically

```
`eval "$(ssh-agent -s)"`

ssh-add <path-to-private-key>
```
![Alt text](Ansible-automation/sshconfig.png)

Confirm the key has been added with the command below, you should see the name of your key

```
ssh-add -l
```
Now, ssh into your `Jenkins-Ansible` server using ssh-agent

```
ssh -A ubuntu@public-ip
```

Also note, that your Load Balancer user is `ubuntu` and user for RHEL-based servers is `ec2-user`.

Update your `inventory/dev.yml` file with this snippet of code:

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


## CREATE A COMMON PLAYBOOK


## **Step 5 – Create a Common Playbook**


It is time to start giving Ansible the instructions on what you needs to be performed on all servers listed in `inventory/dev`.

In `common.yml` playbook you will write configuration for repeatable, re-usable, and multi-machine tasks that is common to systems within the infrastructure.

Update your `playbooks/common.yml` file with following code:

```
---
- name: update web, nfs and db servers
  hosts: webservers, nfs
  remote_user: ec2-user
  become: yes
  become_user: root
  tasks:
    - name: ensure wireshark is at the latest version
      yum:
        name: wireshark
        state: latest

- name: update LB server
  hosts: lb, db
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

Examine the code above and try to make sense out of it. This playbook is divided into two parts, each of them is intended to perform the same task: install **wireshark** utility (or make sure it is updated to the latest version) on your RHEL 8 and Ubuntu servers. It uses `root` user to perform this task and respective package manager: `yum` for RHEL 8 and `apt` for Ubuntu.


## **Step 6 – Update GIT with the latest code**

Push all the changes made to the directories and files from local machine to Github.

Commit your code into GitHub:

1. use git commands to add, commit and push your branch to GitHub.
```
git status

git add <selected files>

git commit -m "commit message"
```
2. Create a Pull request (PR)

3. Wear a hat of another developer for a second, and act as a reviewer.

4. If the reviewer is happy with your new feature development, merge the code to the master branch.

5. Head back on your terminal, checkout from the feature branch into the master, and pull down the latest changes.

Once the code changes appear in master branch – Jenkins will do its job and save all the files (build artifacts) to /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/ directory on Jenkins-Ansible server as shown below.

![Alt text](Ansible-automation/playbookdisplay.png)



## RUN FIRST ANSIBLE TEST


## **Step 7 – Run first Ansible test**


Now, it is time to execute ansible-playbook command and verify if your playbook actually works:

Connect to your jenkins-ansible server via VScode (configure the .ssh/config file with your jenkins server information)

**`cd ansible-config-mgt`**

**`ansible-playbook -i inventory/dev.yml playbooks/common.yml`**


![Alt text](<Ansible-automation/ansible run.png>)

You can go to each of the servers to run the command **`wireshark --version`** or **`which wireshark`** to confirm that wireshark was successfully installed.

![Alt text](<Ansible-automation/wireshark version.png>)


At the end of this project we have implemented a solution that is shown below

![Alt text](<Ansible-automation/anisble architecture.png>)
