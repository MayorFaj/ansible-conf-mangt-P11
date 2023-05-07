# ansible-conf-mangt-P11
ANSIBLE CONFIGURATION MANAGEMENT – AUTOMATE PROJECT 7 TO 10






- In your ansible-config-mgt GitHub repository, create a new branch that will be used for development of a new feature.

- Checkout the newly created feature branch to your local machine and start building your code and directory structure there

`git checkout -b Prj-11a`

- Create a directory and name it playbooks – it will be used to store all your playbook files.

`mkdir palybooks`

- Create a directory and name it inventory – it will be used to keep your hosts organised.

`mkdir inventory`

- Within the playbooks folder, create your first playbook, and name it common.yml

`cd playbooks && touch common.yml`

- Within the inventory folder, create an inventory file (.yml) for each environment (Development, Staging Testing and Production) dev, staging, uat, and prod respectively.


`cd inventory && touch dev.ym staging.yml hat.yml prod.yml`


## Step 4 – Set up an Ansible Inventory

- An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate.

- Save below inventory structure in the `inventory/dev` file to start configuring your development servers. Ensure to replace the IP addresses according to your own setup.


- **Note:** Ansible uses TCP port 22 by default, which means it needs to ssh into target servers from Jenkins-Ansible host – for this you can implement the concept of ssh-agent. Now you need to import your key into ssh-agent.

- Open another terminal, run

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

## Step 5 – Create a Common Playbook

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

- This playbook is divided into two parts, each of them is intended to perform the same task: install wireshark utility (or make sure it is updated to the latest version) on your RHEL 8 and Ubuntu servers. It uses root user to perform this task and respective package manager: yum for RHEL 8 and apt for Ubuntu.



## Step 6 – Update GIT with the latest code

- Now all of your directories and files live on your machine and you need to push changes made locally to GitHub.

- Now we have a separate branch, there is need to  raise a `Pull Request` (PR), get your branch peer reviewed and merged to the main branch.

- Commit your code into GitHub:

1. Use git commands to add, commit and push your branch to GitHub.

```
git status

git add <selected files>

git commit -m "commit message"

```

2. Create a Pull request 

3. Review the code, and merge to the main branch.

4. Head back on your terminal, checkout from the feature branch into the main, and pull down the latest changes.

`git checkout main`

`git pull origin prj-11a`

- Once your code changes appear in main branch – Jenkins will do its job and save all the files (build artifacts) to `/var/lib/jenkins/jobs/ansible/builds/<build_number>/archive/` directory on Jenkins-Ansible server.



## Step 7 – Run Ansible test

- on your jenkins-ansible server termianl, run

`ansible-playbook -i /var/lib/jenkins/jobs/ansible/builds/<buildnumber>/archive/inventory/dev.yml /var/lib/jenkins/jobs/ansible/builds/<buildnumber>/archive/playbooks/common.yml`

![Ansible-playbook comm result](./images/AnsiblePlaybook%20Comm.png)

- You can go to each of the servers and check if wireshark has been installed by running `which wireshark` or `wireshark --version`

![WebS1](/images/WS1%20wireshark.png)

![WebS2](./images/WS2%20wireshark.png)

![DB](./images/DB%20wireshark.png)

![NFS](./images/NFS%20wireshark.png)

![LB](./images/LB%20wireshark.png)


- The updated with Ansible architecture now looks like this:

![Updated Architechture](./images/Updated%20architecture.png)


### Optional Step

- Update your ansible playbook with some new Ansible tasks and go through the full

`checkout -> change codes -> commit -> PR -> merge -> build -> ansible-playbook`

cycle again to see how easily you can manage a servers fleet of any size with just one command!

- Update this playbook with following tasks:

Create a directory and a file inside it
Change timezone on all servers
Run some shell script
...


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

```

 


# **THANKS!!!**