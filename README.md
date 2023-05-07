# ansible-conf-mangt-P11
**ANSIBLE CONFIGURATION MANAGEMENT – AUTOMATE PROJECT 7 TO 10**

### Ansible Client as a Jump Server (Bastion Host)

- A Jump Server (sometimes also referred as Bastion Host) is an intermediary server through which access to internal network can be provided. For instance, the current architecture i am  working on, ideally, the webservers would be inside a secured network which cannot be reached directly from the Internet. That means, even DevOps engineers cannot SSH into the Web servers directly and can only access it through a Jump Server – it provide better security and reduces attack surface.

- On the diagram below the Virtual Private Network (VPC) is divided into two subnets – Public subnet has public IP addresses and Private subnet is only reachable by private IP addresses.

![VPC Arch](./images/VPC%20Arch.png)

## TASK

1.  Install and configure Ansible client to act as a Jump Server/Bastion Host.

2. Create a simple Ansible playbook to automate servers configuration



### STEP1: INSTALL AND CONFIGURE ANSIBLE ON EC2 INSTANCE

1. Update Name tag on your Jenkins EC2 Instance to `Jenkins-Ansible`. We will use this server to run playbooks.

2. In your GitHub account create a new repository and name it ansible-config-mgt.

3. Install Ansible

```
sudo apt update

sudo apt install ansible
```

- Check your Ansible version by running `ansible --version`

4. Configure Jenkins build job to save your repository content every time you change it.

- Create a new Freestyle project `ansible` in Jenkins and point it to your ‘ansible-config-mgt’ repository.
- Configure `Webhook` in GitHub and set webhook to trigger ansible build.
- Configure a Post-build job to save all (**) files.

5. Test your setup by making some change in README.MD file in main branch and make sure that builds starts automatically and Jenkins saves the files (build artifacts) in following folder
`ls /var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/`

- Now the setup looks like this;

![Architecture1](./images/Architecture.png)

Note; Every time you stop/start your Jenkins-Ansible server – you have to reconfigure GitHub webhook to a new IP address, in order to avoid it, it makes sense to allocate an Elastic IP to your Jenkins-Ansible server.(Refer to LB server in Project 10).

## STEP 2 – Prepare your development environment using Visual Studio Code

1. Intall Visual Studio Code, configure it to connect to your newly created GitHub repository.

2. Clone down your ansible-config-mgt repo to your Jenkins-Ansible instance
git clone <ansible-config-mgt repo link>


3. In your ansible-config-mgt GitHub repository, create a new branch that will be used for development of a new feature.

4. Checkout the newly created feature branch to your local machine and start building your code and directory structure there.

`git checkout -b Prj-11a`

5. Create a directory and name it playbooks – it will be used to store all your playbook files.

`mkdir palybooks`

6. Create a directory and name it inventory – it will be used to keep your hosts organised.

`mkdir inventory`

7. Within the playbooks folder, create your first playbook, and name it common.yml

`cd playbooks && touch common.yml`

8. Within the inventory folder, create an inventory file (.yml) for each environment (Development, Staging Testing and Production) dev, staging, uat, and prod respectively.


`cd inventory && touch dev.ym staging.yml hat.yml prod.yml`


## STEP 3 – Set up an Ansible Inventory

- An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate.

- Save below inventory structure in the `inventory/dev.yml` file to start configuring your development servers. Ensure to replace the IP addresses according to your own setup.


- **Note:** Ansible uses TCP port 22 by default, which means it needs to ssh into target servers from Jenkins-Ansible host – for this can implement the concept of ssh-agent. 

You can enable this  SSH configuration file snippet for connecting to a remote server  with the Public IP address of "3.132.218.7" on your **IDE**

```
Host jenkins-ansible
    Hostname 3.132.218.7
    User ubuntu
    IdentityFile <path to key.pem file>(/Users/mozart/downloads/)MayorFaj-EC2.pem
    ForwardAgent yes
    ControlPath /tmp/ansible-ssh-%h-%p-%r
    ControlMaster auto
    ControlPersist 10m 
```
### **Here's what each line means:**

- **Host jenkins-ansible"** specifies the nickname or alias for the remote server that you want to connect to.
- **"Hostname 3.132.218.7"** specifies the IP address or domain name of the remote server.
- **"User ubuntu"** specifies the username that will be used to authenticate with the remote server.
- **"IdentityFile /Users/mozart/downloads/MayorFaj-EC2.pem"** specifies the path to the SSH private key that will be used to authenticate with the remote server.
**"ForwardAgent yes"** specifies that SSH agent forwarding should be enabled for this connection, allowing the user to use their local SSH keys to authenticate with other servers accessed from the remote server.
**"ControlPath /tmp/ansible-ssh-%h-%p-%r"** specifies the path to the control socket used for connection multiplexing.
**"ControlMaster auto"** specifies that connection multiplexing should be enabled for this connection.
**"ControlPersist 10m"** specifies that the master connection should remain active for up to 10 minutes after the last active session, allowing subsequent sessions to use the same master connection

- Now you need to import your key into ssh-agent.

```
eval `ssh-agent -s`
ssh-add <path-to-private-key>
```


- Confirm the key has been added with the command below, you should see the name of your key

`ssh-add -l`

- Now, ssh into your Jenkins-Ansible server using ssh-agent

`ssh -A ubuntu@public-ip`


- Update your inventory/dev.yml file with this snippet of code:

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

## STEP 4 – Create a Common Playbook

- It is time to start giving Ansible the instructions on what you needs to be performed on all servers listed in inventory/dev

- In common.yml playbook you will write configuration for repeatable, re-usable, and multi-machine tasks that is common to systems within the infrastructure

- Update your playbooks/common.yml file with following code:

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

# ---------------------------------------------------------------------

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
...
```

- ###  This playbook is divided into two parts, each of them is intended to perform the same task: install wireshark utility (or make sure it is updated to the latest version) on your RHEL 8 and Ubuntu servers. It uses root user to perform this task and respective package manager: yum for RHEL 8 and apt for Ubuntu.



## STEP 5 – Update GIT with the latest code

- Now all of your directories and files live on your machine and you need to push changes made locally to GitHub.

- We have a separate branch, there is need to  raise a `Pull Request` (PR), get your branch peer reviewed and merged to the main branch.

- Commit your code into GitHub:

1. Use git commands to add, commit and push your branch to GitHub.

```
git status

git add <selected files>

git commit -m "commit message"

```

2. Head to your github account, create a Pull request 

3. Review the code, and merge to the main branch.

4. Head back on your terminal, checkout from the feature branch into the main, and pull down the latest changes.

`git checkout main`

`git pull origin prj-11a`

- Once your code changes appear in main branch – Jenkins will do its job and save all the files (build artifacts) to `/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/` directory on `Jenkins-Ansible server`.



## STEP 7 – Run Ansible test

- Execute ansible-playbook command . On your jenkins-ansible server termianl, run

`ansible-playbook -i /var/lib/jenkins/jobs/ansible/builds/<buildnumber>/archive/inventory/dev.yml /var/lib/jenkins/jobs/ansible/builds/<buildnumber>/archive/playbooks/common.yml`

![Ansible-playbook comm result](./images/AnsiblePlaybook%20Comm.png)

- You can go to each of the servers and check if wireshark has been installed by running `which wireshark` or `wireshark --version`

![WebS1](/images/WS1%20wireshark.png)

![WebS2](./images/WS2%20wireshark.png)

![DB](./images/DB%20wireshark.png)

![NFS](./images/NFS%20wireshark.png)

![LB](./images/LB%20wireshark.png)


- The updated Ansible architecture now looks like this:

![Updated Architechture](./images/Updated%20architecture.png)


### Optional Step

- Update your ansible playbook with some new Ansible tasks and go through the full

`checkout -> change codes -> commit -> PR -> merge -> build -> ansible-playbook`

cycle again to see how easily you can manage a servers fleet of any size with just one command!

- Update this playbook with following tasks:

1. Create a directory and a file inside it
2. Change timezone on all servers
3. Run some shell script
4. ...


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

    - name: create directory and file
      file:
        path: /var/www/html/mydir/myfile.txt
        state: touch
        mode: '0644'
        owner: apache
        group: apache

    - name: change timezone
      timezone:
        name: America/New_York

    - name: run shell script
      shell: /path/to/my/script.sh
      args:
        chdir: /path/to/my/script/dir/
# ------------------------------------------------------------------
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

    - name: create directory and file
      file:
        path: /var/www/html/mydir/myfile.txt
        state: touch
        mode: '0644'
        owner: apache
        group: apache

    - name: change timezone
      timezone:
        name: America/New_York

    - name: run shell script
      shell: /path/to/my/script.sh
      args:
        chdir: /path/to/my/script/dir/
...


```

 


# **THANKS!!!**