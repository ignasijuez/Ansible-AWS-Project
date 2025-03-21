0️⃣ Base

create aws free tier account

install wsl for windows

install awscli
    $ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    $ unzip awscliv2.zip
    $ sudo ./aws/install
    $ aws --version
    output: aws-cli/2.x.x Python/3.x.x Linux/x86_64

install ansible

configure vscode to open on ubuntu


1️⃣ Set Up AWS CLI & Credentials
    In the context of setting up Ansible to work with AWS, you use AWS CLI to configure the credentials that will allow Ansible to interact with your AWS account and perform tasks like managing EC2 instances.
    AWS CLI is a command-line tool for interacting with AWS services from your terminal.
    You use it to configure and authenticate your IAM user’s credentials and region, allowing both AWS CLI and tools like Ansible to interact with your AWS account.
    After configuration, Ansible can automatically pick up the credentials set via the AWS CLI to manage your AWS resources like EC2 instances.


2️⃣ Set Up IAM Permissions for Ansible

configure ansible user in the IAM
    Go to AWS IAM Console → https://console.aws.amazon.com/iam
    Click "Users" → "Create User"
    Set username (e.g., ansible-user)
    Set permissions (choose one option):
        Option 1 (Recommended): Attach AdministratorAccess (Full AWS control)
        Option 2 (Safer): Attach only the required policies:
            AmazonEC2FullAccess (for managing VMs)
            IAMReadOnlyAccess (to read users/roles)
            AmazonS3ReadOnlyAccess (if using S3)
            CloudWatchFullAccess (for monitoring)
    Click "Create User"

Create Access Key for ansible-user
    For Access key type, choose CLI access (this is for the AWS CLI).
    Copy Access Key & Secret Key

Configure AWS CLI with IAM User
    $aws configure
    And enter the new IAM user's credentials, not the root account.
    check with: $aws sts get-caller-identity or $aws ec2 describe-instances


3️⃣ Install Required Ansible Collections
    In this step, you're going to install the Ansible AWS collection. This collection includes all the modules that allow you to interact with AWS resources (like EC2 instances, S3 buckets, etc.) using Ansible

Install Ansible AWS Collection
    $ansible-galaxy collection install amazon.aws
    check wiht: $ansible-doc -t module -l | grep aws
                $ansible-doc -l | grep aws


    When you installed the amazon.aws collection, Ansible gained the following functionality:

    1 - It can now interact with EC2 instances (e.g., create, delete, and manage instances).
    2 - It can manage S3 buckets, RDS instances, IAM roles/policies, and other AWS resources.
    3 - You can run Ansible playbooks to automate tasks like creating instances, managing security groups, uploading files to S3, etc.


4️⃣ Set Up Ansible Inventory for AWS

Instead of a static inventory.yml, we'll use dynamic inventory.
    Install Boto3 & Botocore (Required for AWS Modules)
        $sudo apt install python3-pip
        $sudo apt install python3.12-venv
        $python3 -m venv ansible-env
        $source ansible-env/bin/activate
        $pip install boto3 botocore
        $

    Configure Dynamic Inventory
        in ~/ansible/aws_inventory
        ""
        plugin: amazon.aws.aws_ec2
        regions:
        - us-east-1  # Change to your region
        filters:
        instance-state-name: running
        keyed_groups:
        - key: tags.Name
            prefix: tag

        ""

    make sure it uses the python from the environment

    checks:
        $ansible-inventory -i aws_ec2.yml --list
        $aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,Placement.AvailabilityZone]" --output table

        i created one instance manually and then it worked


5️⃣ Deploy Your EC2 Instances Using Ansible
    
Now, let's create the 3 servers:
1 for DB (MySQL)
2 for Web (Flask)
📌 Create create_ec2.yml Playbook

---
- name: Launch AWS EC2 instances
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Launch DB Server
      amazon.aws.ec2_instance:
        name: "db-server"
        key_name: my-key                    # Your key pair from ec2 (see dashboard and create one)
        instance_type: t3.micro
        security_group: default
        image_id: ami-001b41487253ce7f9     # Replace with your region's AMI (see in EC2 -> AMIs)
        region: eu-south-2                  # Your region
        count: 1
        tags:
          Name: db-server

    - name: Launch Web Servers
      amazon.aws.ec2_instance:
        name: "web-server"
        key_name: my-key                    # Your key pair from ec2 (see dashboard and create one)
        instance_type: t3.micro
        security_group: default
        image_id: ami-001b41487253ce7f9     # Replace with your region's AMI (see in EC2 -> AMIs)
        region: eu-south-2                  # Your region
        count: 2
        tags:
          Name: web-server

    create the key-pair, download the .pem, add it to .ssh in your windows and copy it to your home in wsl
    In WSL, you can access the Windows file system through /mnt/c/. So, if your .pem file is located in C:\Users\<YourUserName>\aws-keys\my-key.pem, you can access it in WSL as /mnt/c/Users/<YourUserName>/aws-keys/my-key.pem.
        $cp /mnt/c/Users/<YourUserName>/aws-keys/my-key.pem ~/my-key.pem
        $chmod 400 ~/my-key.pem
    
    Add the key to the SSH agent
        $ssh-add -l
            2048 SHA256:6Q+EoywAIOWjZGX7JQ7q4MOtNi++uAzFUJBqQI/rPAk /home/fish3418/my-key.pem (RSA)
        $eval $(ssh-agent -s)
        $ssh-add ~/my-key.pem

    Check
        $ssh -i ~/my-key.pem <admin(using debian instance)>@<INSTANCE_IP>

        if you can't access may the inbound rule be the problem:
            go to security groups
            select the group that you are working with
            edit inbound rules and add one for the ssh with custom and 0.0.0.0/0

    Now back with it:
        $ansible-playbook create_ec2.yml -vvv

    Check:
        $aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,Placement.AvailabilityZone]" --output table


6️⃣ Playbook

    Check:
        $ansible all -i aws_ec2.yml -m ping
        add on ansible.cfg:
            interpreter_python = auto_silent
            inventory = aws_ec2.yml
            host_key_checking = False
            private_key_file = /home/fish3418/my-key.pem
            remote_user = admin

    Check inventory:
        $ansible-inventory -i aws_ec2.yml --graph

    Create group_vars and use tag_names as file name to use the vars:
        $group_vars/tag__db_server.yml
            db_name, db_user, db_password, etc

    Check ssh to the instance:
        sudo mysql -u root -p
        SELECT User, Host FROM mysql.user;
        SHOW DATABASES;

    Install flask on the other two instances
        Remember that now the db is in another instance so Remote Connections to MySQL needs to be allowed
            $sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
                bind-addr : 0.0.0.0
                and user with all privileges

    Modify the app.py to be used correctly with ec2 instances
    Check inbound rules to accept traffic to port 5000
        Edit the dafault and add inbound rule as TCP 5000 anywhere
    
    Working!
    Welcome on each instance:
        instance1 ip:5000 -> Welcome!
        instance2 ip:5000 -> Welcome!

    Delete all instances, automate the mysql configuration in the playbook.
    $ansible-playbook create_ec2.yml -vvv
    $ansible-playbook -i aws_ec2.yml playbook.yml
    check:
        instance1 ip:5000 -> Welcome!
        instance2 ip:5000 -> Welcome!

    AFTER DONE:
        automate the creation of table and the users
            need to update the bind-addr and then restart the server
        automate the update of the code of the app.py for new creation of db instances that have a dif ip

    Working!


7️⃣ Configure Load Balancing
Option 1: Use Nginx as a load balancer on a separate VM.
Option 2: Use AWS ALB, GCP Load Balancer, or Azure Load Balancer.

    We will use AWS CLB:
    Create setup_clb.yml
        CLB will automatically distribute incoming traffic between the two web server instances. 
        The load balancer will route requests to the instances based on the configured listener and health check settings.

    Check:
        $aws elb describe-instance-health --load-balancer-name my-clb --region eu-south-2
            check the state: In-Service!

        check via web: access http://<your-clb-DNS name>

solved (did not restart db after doing the bind-addr so it wasnt working)
CHECK connection to db:
    logs from the flask app that doesn't connect to the db:
    $sudo journalctl -u flask_app.service -f


8️⃣ Automate Notification
After deployment, send an email notification using:
community.general.mail module
SMTP (e.g., Gmail, AWS SES)
Include server details (IP, ports, credentials)

    Install Required Dependencies
    $ansible-galaxy collection install community.general

    If using Gmail, also install ssmtp (for Debian/Ubuntu) or mailx (for RHEL-based):
    sudo apt install ssmtp -y  # Debian/Ubuntu
    sudo yum install mailx -y  # RHEL/CentOS

    Configure Ansible Playbook and credentials using vault
    $vim vault.yml (put credentials there, stmp user and password from previous step)
    $sudo su
    $ansible-vault encrypt vault.yml

    $ansible-playbook notify.yml --ask-vault-pass
        (you should have access read to the file if not it wont work i did chown)

    working!


9️⃣ Secure Everything
Store credentials in Ansible Vault.
Use firewall rules / security groups to allow traffic only on necessary ports.

All credentials in ansible vault
    now edit it with: $ansible-vault edit vault.yml
    we put; smtp_user, smtp_pass, smtp_server, smtp_port, db_name, db_user, db_password

Security group rules have been added during the development and only for the purpose of that time so its
all what needed and not more, always go for give the minimum permisions and then allow
instead of give all and restric some


🔟 Final automatization:
All in one file
    problem: didn't update the dynamic inventory after the creation of the instances so all failed
    key Ansible behavior: inventory is only loaded once at the start of a playbook run.
    so we need to use "meta: refresh_inventory"

    we used: 
        import_playbook: create_ec2.yml
        and also
        made the setup_clb and setup_notify as roles, well all are roles
        and added a small pause until refresh inventory to wait ip assignation

    $ansible-playbook -i aws_ec2.yml playbook.yml --ask-vault-pass -vvv
        (this does the create ec2, deploy db and web, set up clb, notify via gmail)

    important note:
        to automate it all in one playbook i had problems understanding this:
            - El problema es que el inventario dinámico no se actualiza automáticamente dentro de un mismo playbook.
            - soluciones:
                - Opción 1: Usar meta: refresh_inventory
                    Pero OJO: meta: refresh_inventory solo funciona si tienes gather_facts: true.
                - Opción 2: Agregar un tiempo de espera antes del deploy
                - Opción 3: Hacer un nuevo llamado explícito al inventario dinámico
                    - name: Force inventory refresh
                        command: ansible-inventory -i aws_ec2.yml --list
                        delegate_to: localhost
                        run_once: true

            Usamos la opción 1:
            - La razón por la que gather_facts: yes es necesaria para que meta: refresh_inventory funcione es porque Ansible solo actualiza el inventario dinámico cuando recolecta facts.
            Si gather_facts: no, Ansible no recolecta información de los hosts y no actualiza el inventario dinámico.
            Esto significa que cuando ejecutas meta: refresh_inventory, Ansible no lo aplica correctamente, porque depende de los facts para actualizar los hosts.

            📌 Ejemplo de problema sin gather_facts:
                Creas las instancias EC2 en el playbook.
                Ejecutas meta: refresh_inventory, pero el inventario no cambia porque no se recopilan facts.
                Cuando intentas hacer tareas en los nuevos servidores (tag__web_server), Ansible dice:
                ❌ "the field 'hosts' is required but was not set"
                Porque aún no reconoce que esas instancias existen.

    final thing added:
        automation of the creation of the security group
        important note:
            The ec2_security_group module is declarative: it sets the security group’s rules to exactly what you specify in each task. If you don’t include the previous rules in the second task, they’re removed.
            Solution:
                - task1: create the security group with no rules
                - task2: add all the rules
            (ec2_security_group_rule was not available in the ansible documentation eventhough ia told me it was the latest version and could be used the module) 

🔢:
    $ansible-playbook -i aws_ec2.yml playbook2.yml --ask-vault-pass
    check notification mail, access dns name of the clb
    and check welcome page, dnsname/how are you, dnsname/read from database

    working!


TODO NEXT:
Set up the Https for the web-servers and clb

role ssl_certificate



CHANGE for use:
notify role: the emails + add your credentials in the vault:
smtp_user: "your email"
smtp_pass: "xxxx"  # App Password de Google
smtp_server: "smtp.gmail.com"
smtp_port: 587


------------------------------------------------------------------------------------------------------------

0 clear about the billing, what the free tier includes as free
all the time you have to be atento a veure si no et cobraran a lhora de fer algo



(ansible-env) fish3418@fish:~/ansible/aws_inventory$ ssh-add -l
2048 SHA256:6Q+EoywAIOWjZGX7JQ7q4MOtNi++uAzFUJBqQI/rPAk /home/fish3418/my-key.pem (RSA)
(ansible-env) fish3418@fish:~/ansible/aws_inventory$ aws ec2 describe-instances --query 'Reservations[*].Instances[*].PublicIpAddress'
[
    [],
    [
        "51.92.170.27"
    ]
]
(ansible-env) fish3418@fish:~/ansible/aws_inventory$ ssh -i ~/my-key.pem admin@51.92.170.27
Linux ip-172-31-9-132 6.1.0-23-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.99-1 (2024-07-15) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
admin@ip-172-31-9-132:~$ exit
logout
Connection to 51.92.170.27 closed.
(ansible-env) fish3418@fish:~/ansible/aws_inventory$