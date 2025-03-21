
### `docs/SETUP.md` (Detailed Documentation)

# Setup and Deployment Guide

This guide details the step-by-step process to set up and deploy the Ansible-AWS project from a Windows OS.


## 0️⃣ Base Setup

**Purpose**: Build the foundation.

In this step, you'll install and configure all the foundational tools required for this project, including AWS CLI, Ansible, WSL, and VSCode.

- **AWS Free Tier Account**: Sign up at [aws.amazon.com](https://aws.amazon.com).
- **Install WSL on Windows**: Enable WSL and install Ubuntu via the Microsoft Store.
- **Install AWS CLI on WSL**:
  ```bash
  $ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  $ unzip awscliv2.zip
  $ sudo ./aws/install
  $ aws --version
  # Expected output: aws-cli/2.x.x Python/3.x.x Linux/x86_64
  ```
- **Install Ansible**: Follow the official guide for your OS.
- **Configure VSCode for WSL**: Install the "Remote-WSL" extension.


### 1️⃣ Set Up AWS CLI & Credentials

**Purpose**: Authenticate Ansible with AWS.

In this step, you'll configure the AWS CLI with the necessary IAM credentials, allowing Ansible to manage AWS resources programmatically.


#### Steps:
1. In AWS IAM Console:
   - Navigate to "Users" → "Create User" → Name: `ansible-user`.
   - Assign Permissions:
     - **Option 1**: `AdministratorAccess` (full control, recommended for testing).
     - **Option 2**: `AmazonEC2FullAccess`, `IAMReadOnlyAccess`, `AmazonS3ReadOnlyAccess`, `CloudWatchFullAccess`.
   - Create CLI access key and copy credentials.

2. Configure AWS CLI:
   ```bash
   $ aws configure
   ```
   - Use IAM credentials and region (e.g., `eu-south-2`).

3. Verify setup:
   ```bash
   $ aws sts get-caller-identity
   ```

### 2️⃣ Install Ansible Collections

**Purpose**: Enable AWS interaction.

In this step, you're going to install the Ansible AWS collection which includes all the modules that allow you to interact with AWS resources using Ansible.

#### Steps:
1. Install required collection:
   ```bash
   $ ansible-galaxy collection install amazon.aws
   ```
2. Verify installation:
   ```bash
   $ ansible-doc -t module -l | grep aws
   ```

When you installed the amazon.aws collection, Ansible gained the following functionalities:
1. It can now interact with EC2 instances (e.g., create, delete and manage instances)
2. It can manage s3 buckets, RDS instances, IAM roles/policies, and other AWS resources
3. You can run Ansible playbooks to automate tasks like creatin instances, managing security groups, uploading files to S3, etc


### 3️⃣ Set Up Dynamic Inventory

**Purpose**: Manage EC2 instances dynamically.

Here, you'll configure Ansible to use AWS dynamic inventory, ensuring that your playbooks always work with the latest EC2 instances without manual updates.

#### Steps:
1. Install dependencies:
   ```bash
   $ sudo apt install python3-pip python3.12-venv
   $ python3 -m venv ansible-env
   $ source ansible-env/bin/activate
   $ pip install boto3 botocore
   ```

   Make sure that it uses the python from the environment.

2. Create `aws_ec2.yml` in `~/ansible/aws_inventory`:
   ```yaml
   plugin: amazon.aws.aws_ec2
   regions:
     - eu-south-2
   aws_access_key_id: <your_key_id>
   aws_secret_access_key: <your_secret_access_key>
   filters:
     instance-state-name: running
   keyed_groups:
     - key: tags.Name
       prefix: tag
    hostnames:
      - dns-name
    compose:
      ansible_user: admin 
   ```

   The ansible_user is set to ‘admin’ as the instance is running Debian, and this is the standard method for connecting to it.

3. Verify setup:

    Create one instance manually and run the following commands:
   ```bash
   $ ansible-inventory -i aws_ec2.yml --list
   $ aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,Placement.AvailabilityZone]" --output table
   ```

### 4️⃣ Setup Security Group

**Purpose**: Create a new SG.

This step covers ho to use Ansible to create a new SG with custom rules.

#### Steps:
1. Create `setup_sg` role.
    ```bash
    $ ansible-galaxy init setup_clb
    # vim /roles/setup_clb/tasks/main.yml # tasks are defined here
    ```
2. Append it to the playbook and run it
    ```bash
    $ ansible-playbook -i aws_ec2.yml playbook.yml -vvv
    ```
3. Verify:
   ```bash
   $ aws ec2 describe-security-groups --region eu-south-2
   ```
   Or via the UI visiting the security group tab


### 5️⃣ Deploy EC2 Instances

**Purpose**: Launch servers.

This step covers how to use Ansible to provision EC2 instances, automating the setup of database and web servers on AWS.

#### Steps:
1. Create `create_ec2.yml`:
   ```yaml
   - name: Launch AWS EC2 instances
     hosts: localhost
     gather_facts: no
     tasks:
       - name: Launch DB Server
         amazon.aws.ec2_instance:
           name: "db-server"
           key_name: my-key
           instance_type: t3.micro
           security_group: ansible-sg
           image_id: <your_region's AMI>
           region: eu-south-2
           count: 1
           tags:
             Name: db-server
       - name: Launch Web Servers
         amazon.aws.ec2_instance:
           name: "web-server"
           key_name: my-key
           instance_type: t3.micro
           security_group: ansible-sg
           image_id: <your_region's AMI>
           region: eu-south-2
           count: 2
           tags:
             Name: web-server
   ```

2. Set up SSH key:

    Create the key-pair in the AWS UI, download the .pem, add it to .ssh in your windows and copy it to your home in wsl.

    In WSL, you can access the Windows file system through /mnt/c/. So, if your .pem file is located in C:\Users\<YourUserName>\aws-keys\my-key.pem, you can access it in WSL as /mnt/c/Users/YourUserName/aws-keys/my-key.pem.

   ```bash
   cp /mnt/c/Users/<YourUserName>/aws-keys/my-key.pem ~/my-key.pem
   chmod 400 ~/my-key.pem
   eval $(ssh-agent -s)
   ssh-add ~/my-key.pem
   ```
3. Check connectivity
   ```bash
   $ ssh -i ~/my-key.pem <admin>@<INSTANCE_IP>
   ```
    If you can't access may the inbound rule be the problem. To solve this, add a new inbound rule in the security group you are working with and add one rule to enable ssh.

4. Run the playbook:
   ```bash
   $ ansible-playbook create_ec2.yml -vvv
   ```
   The option '-vvv' is a verbose option to monitor and detect failures.

5. Check instances
   ```bash
   $ aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,Placement.AvailabilityZone]" --output table
   ```

### 6️⃣ Playbook

**Purpose**: Deploy MySQL and Flask.

Now, you'll configure and deploy a MySQL database and Flask application on the provisioned servers, ensuring they are ready to handle requests.

#### Steps:
1. Check the ansible connectivity to the instances:
    ```bash
    $ ansible all -i aws_ec2.yml - ping
    ```

2. Edit `ansible.cfg`:
   ```ini
   [defaults]
   interpreter_python = auto_silent
   inventory = aws_ec2.yml
   host_key_checking = False
   private_key_file = /home/<your-user>/my-key.pem
   remote_user = admin
   ```

3. Create `playbook.yml` for MySQL and Flask deployment.
    
    Explore all the files in the repository. Special attention on roles, there is where all the code is!!

    Remember that now the db is in another instance so Remote Connections to MySQL needs to be allowed, for this you need to /etc/mysql/mariadb.conf.d/50-server.cnf and modify this line like this: bin-addr : 0.0.0.0

4. Run:
   ```bash
   ansible-playbook -i aws_ec2.yml playbook.yml -vvv
   ```

### 7️⃣ Set Up Load Balancing

**Purpose**: Distribute traffic with CLB.

In this step, you'll set up an AWS Classic Load Balancer (CLB) to evenly distribute traffic across multiple web servers, enhasncing availability and reliability. Another solution would be to use nginx as a load balancer on a separate instance.

#### Steps:
1. Create `setup_clb` role.
    ```bash
    $ ansible-galaxy init setup_clb
    # vim /roles/setup_clb/tasks/main.yml # tasks are defined here
    ```
2. Append it to the playbook and run it
    ```bash
    $ ansible-playbook -i aws_ec2.yml playbook.yml -vvv
    ```
3. Verify:
   ```bash
   $ aws elb describe-instance-health --load-balancer-name my-clb --region eu-south-2
   ```
   Or via web visiting: http://<your-clb-DNS-name>:5000

### 8️⃣ Automate Notification

**Purpose**: Send email post-deployment.

You'll configure Ansible to send email notifications upon successful deployment, providing automated updates on the deployment status.

#### Steps:
1. Install required collection:
   ```bash
   $ ansible-galaxy collection install community.general
   ```
   If using Gmail, also install ssmtp (for Debian/Ubuntu) or mailx (for RHEL-based)
   ```bash
   $ sudo apt install ssmtp -y # Debian/Ubuntu
   $ sudo yum install mailx -y # RHEL/CentOS
3. Create `setup_notify` role.
    ```bash
    $ ansible-galaxy init setup_notify
    # vim /roles/setup_notify/tasks/main.yml # tasks are defined here
    ```
4. Append it to the playbook and run it
    ```bash
    $ ansible-playbook -i aws_ec2.yml playbook.yml -vvv
    ```
5. Verify:

   You have to receive an email with all the information about the deployment.


### 9️⃣ Secure the Setup

**Purpose**: Protect credentials and access.

#### Steps:
1. Create `vault.yml`:
   ```bash
   ansible-vault create vault.yml
   ```
   Add all the variables that need to be private.

2. Restrict security groups to necessary ports.

### 🔟 Final Automation

**Purpose**: Single playbook deployment.

#### Steps:
1. End the playbook with all the roles and add another role to wait after the creation of the instances before going on with the inventory refresh.
2. Run:
   ```bash
   ansible-playbook -i aws_ec2.yml playbook2.yml --ask-vault-pass
   ```

## Changes to be made to work
- **Modify the notify role**: Use your emails.
- **Modify vault**: Update the vault secrets and then encrypt it.
- **Modify ansible.cfg**: Update the path of the private key file.


## Key Notes

- **Inventory Refresh**: Use `meta: refresh_inventory` with `gather_facts: yes`.

When automating everything in a single playbook, a major issue arose: after creating the AWS instances, Ansible’s dynamic inventory was not updating automatically, causing all subsequent tasks to fail. This happens due to a key Ansible behavior—inventory is only loaded once at the start of a playbook run and does not refresh on its own.

To resolve this, there were three possible solutions. The first was to use meta: refresh_inventory, but this required gather_facts to be enabled since Ansible only updates its inventory when it collects host facts. The second option was to add a wait time before proceeding with the deployment, allowing the inventory to reflect the changes. The third alternative was to manually force an inventory update by running a specific command from localhost.

The chosen solution was the first one, ensuring that gather_facts was enabled. Without this setting, even if meta: refresh_inventory was executed, the inventory would not update correctly because Ansible would not have gathered the necessary information. As a result, when trying to execute tasks on the newly created servers, Ansible would fail to recognize them and return an error stating that the “hosts” field was not set.

- **Security Groups**: `ec2_security_group` is declarative—specify all rules per task.

An important detail in automating security groups was that the ec2_security_group module is declarative, meaning each task overwrites previous rules instead of adding new ones. To avoid this, the security group was first created without rules, and then all rules were added in a single task.
