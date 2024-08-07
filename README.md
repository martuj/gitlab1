### PRE-REQUISITES
For this project, we will be using the Google Cloud Platform for our Infrastructure. 
* Service account credential
* GitLab Personal Access Token — the service account credential gotten above as well as the GitLab personal access token must be stored as GitLab pipeline variables. Note, the credential variable should be of type file.

### Steps

#### Step1
After generating the token add the Credential and Token as variables by performing the following steps:

Go to the GitLab project you created for this project
Click to ‘Settings’ the last option in the sidebar of the project
Click on ‘CI/CD’, then expand the variables section of the page
Click ‘Add Variable’, and add the variable for CREDENTIAL and GITLAB_ACCESS _TOKEN. The CREDENTIAL variable must be of type File

#### Step2: TERRAFORM
Below is a diagram that describes the architecture and connection of the terraform files and operations.

First, we populate the modules/debian_vm folder with the module files main.tf and variables.tf

modules/debian_vm/main.tf
```
# Create Network (VPC)
resource "google_compute_network" "vpc_network" {
  name                    = "${var.environment}-network"
  auto_create_subnetworks = false
}

# Create Subnetwork
resource "google_compute_subnetwork" "tf_subnet" {
  name          = "${var.environment}-subnetwork"
  ip_cidr_range = "10.128.10.0/24"
  region        = "us-central1"
  network       = google_compute_network.vpc_network.name
}

# Create Virtual Machine
resource "google_compute_instance" "vm_instance" {
  name                      = var.environment
  machine_type              = "e2-small"
  tags                      = ["${var.environment}"]
  allow_stopping_for_update = true

  boot_disk {
    # auto_delete = false
    initialize_params {
      image = "debian-cloud/debian-11"
    }
  }

  # Adding SSH keys
  metadata = {
    ssh-keys = "${var.ssh_user}:${file(var.pubkey_file)}"
  }

  network_interface {
    network    = google_compute_network.vpc_network.name
    subnetwork = google_compute_subnetwork.tf_subnet.name
    access_config {
    }
  }
}

# Create firewall rules
resource "google_compute_firewall" "rules" {
  name        = "fwr-${var.environment}"
  network     = google_compute_network.vpc_network.name
  description = "Creates firewall rule targeting tagged instances for terraform infrastructure"

  allow {
    protocol = "tcp"
    ports    = ["80", "8080", "22", "1000-2000"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["${var.environment}"]
}

# Copy the IP address into a txt file
resource "local_file" "vm_ip" {
  content  = google_compute_instance.vm_instance.network_interface[0].access_config[0].nat_ip
  filename = "../${var.environment}_vm_ip.txt"
}
```

modules/debian_vm/variable.tf
```
variable "ssh_user" {
  type        = string
  description = "Username of attached to the SSH key"
  default     = "meharuser"
}

variable "pubkey_file" {
  type        = string
  description = "SSH public key"
  default     = "../.keys/vm_keys.pub"
}

variable "environment" {
  type        = string
  description = "dev, staging or prod"
  default     = "dev"
}
```
The structure of your directory should be

iac-cicd
    └── modules
        └── debian_vm
            ├── main.tf
            └── variables.tf
Next we create the files for the environments: dev

dev/providers.tf
```
# Authenticate with GCP Service account
provider "google" {
  credentials = file("CREDENTIAL")

  project = "deloitte-team2"
  region  = "us-central1"
  zone    = "us-central1-c"
}
```

dev/main.tf
``
# Use the debian module to provision 
module "debian_vm" {
  source = "../modules/debian_vm"
  
  # Input variables
  environment = "dev"
}
```

If you have done the above successfully, your repository should now look like this

cicd
    ├── dev
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    └── modules
         └── debian_vm
              ├── main.tf
              └── variables.tf



### Step3: ANSIBLE
After terraform has successfully provisioned the VMs and their IP addresses have been created then Ansible takes the job from there. One thing to note is that the implementation of the inventory_file and private_ssh_key is done one by one. For example, dev-vm-ip and dev-ssh-key will be the inventory file and private_ssh_key respectively at a time when the playbook is executed before the next environment is configured. The pipeline file makes this clearer.

A better approach will be to join the IP addresses and there respective VMs but the outcome is very similar in structure. The image below describes the workflow of Ansible.

ansible workflow
Create a folder named ansible_files in the project and ensure that the permission is such that is not a world-writable directory (else, the playbook won’t run. More on that here). Add install_nginx.yml file which is responsible for performing the configuration itself. Next add the ansible.cfg which sets the ansible configuration and so the default configuration file /etc/ansible/ansible.cfg will not be used.

ansible_files/ansible.cfg

[defaults]

inventory = ../vm_ip.txt
private_key_file = ../.keys/vm_keys
host_key_checking = False
remote_user = ogabiprince
inventory sets the path to the file that contains the list of IPs that should be SSHed into
private_key_file sets the path to the private SSH key file
host_key_checking sets the SSH option StrictHostKeyChecking=no
remote_user sets the username to authenticate with the VM.

ansible_files/install_nginx.yml

---

- hosts: all
  become: true
  tasks:
    - name: Update Repository Index
      apt: 
        update_cache: yes
    - name: Install Latest nginx Server
      apt: 
        name: nginx # Docker not nginx
        state: latest
    - name: Start nginx
      service:
        name: nginx
        state: started
        enabled: yes
    # - name: Deploy website
    #   copy: 
    #     src: ./website_folder/*
    #     dest: /var/www/html
    #     mode: 0755
hosts key defines target IPs to which should log into which is all in our case.
become key allows us to perform the operations as a sudoer (root)
task key allows us to list the operations we will like to carry out… Read more here

If you have done the above successfully, your repository should now look like this

iac-cicd
    ├── dev
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    ├── staging
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    ├── prod
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    ├── modules
    │    └── debian_vm
    │         ├── main.tf
    │         └── variables.tf
    └── ansible_files
         ├── install_nginx.yml
         └── ansible.cfg
GITLAB-CI PIPELINE
This is my favourite part of the project, adding automation to the task we wish to perform. The image below gives a very high level overview of the pipeline jobs and artifacts.


Now, going into detail…

stages

stages:
  - ssh_gen
  - tf_plan
  - tf_apply
  - ans_config
  - tf_destroy
The .gitlab-ci.yml file contains 5 stages: ssh_gen, tf_plan, tf_apply, ansible and tf_destroy. Using stages is good practice and allows us to easily group jobs that can run in parallel.

variables

variables:
  TERRAFORM_VERSION: 1.5.0
  TF_ADDRESS: http://34.125.152.235/api/v4/projects/1/terraform/state
We define 2 variables on the .gitlab-ci.yml file for easy visibility. The TERRAFORM_VERSION variable is used to store the Terraform Version, TF_ADDRESS stores the partial address of our terraform state files (it is completed by the various environment names).

gen_ssh_key

gen_ssh_keys:
  image: bash:latest
  stage: ssh_gen
  script:
    - apk update && apk add openssh-client
    - mkdir .keys && cd ./.keys
    - ssh-keygen -f vm_keys_${ENVIRONMENT} -q -t rsa -N "" && echo "Keys successfully generated"
  parallel:
    matrix: 
      - ENVIRONMENT: [dev, staging, prod]
  artifacts:
    name: ssh-keys
    expire_in: "1 day"
    paths: 
      - .keys
This job creates SSH key pairs for each environment using a bash image. The names of the keys are dependent on the environment, this is achieved by using a parallel matrix which runs the same job based on the number of elements in an array (in this file, there will be a job creating a vm_keys_dev key pair, a vm_keys_staging key pair and a vm_keys_prod key pair ). Read more about the parallel matrix here or check the official documentation.

.provision

.provision:
  script:
    - apk update && apk upgrade
    - apk add unzip
    - wget -O terraform.zip https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_386.zip
    - unzip terraform.zip && mv ./terraform /bin
    - terraform -version
    - cd $ENVIRONMENT
    - sed -i 's#vm_keys.pub#vm_keys_'"${ENVIRONMENT}"'.pub#g' ../modules/debian_vm/variables.tf
    - sed -i 's#CREDENTIAL#'"${CREDENTIAL}"'#g' provider.tf
    - > 
        terraform init 
        -backend-config="address=${TF_ADDRESS}/${ENVIRONMENT}"
        -backend-config="lock_address=${TF_ADDRESS}/${ENVIRONMENT}/lock" 
        -backend-config="unlock_address=${TF_ADDRESS}/${ENVIRONMENT}/lock"
        -backend-config="username=root" 
        -backend-config="password=${GITLAB_ACCESS_TOKEN}" 
        -backend-config="lock_method=POST" 
        -backend-config="unlock_method=DELETE" 
        -backend-config="retry_wait_min=5"
In Gitlab-CI, jobs that begin with a period/full-stop ‘.’ are hidden and disabled jobs; they do not appear or run in the pipeline (more on that here). The .provision job is a disabled job but will be referenced in other parts of the .gitlab-ci.yml file. The first 5 commands in the script downloads Terraform binary and moves it into the executable PATH. Next, we move into the directory of the environment, rename the SSH keys file path in the modules/debian_vm/main.tf file using the sed command and initialise terraform. NOTE: All terraform jobs require that the terraform init command is run before any other terraform command. The options required to use an HTTP backend are passed here using the -backend-config option. More information about the options that can be configured for an HTTP backend can be found here.

dry_provision

dry_provision:
  image: alpine:3.18.2
  stage: tf_plan
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "gcp-terraform"
  script:
    - !reference [.provision, script]
    - terraform validate
    - terraform plan
  parallel:
    matrix: 
      - ENVIRONMENT: [dev, staging, prod]
In this project, there are 2 core branches; namely ‘gcp-terraform’ and ‘main’. This job will only run on the main branch — our working branch. The first command in the script !reference [.provision, script] will run all the commands in the .provision job by referencing it. This job will validate the terraform configuration using the terraform validate command and show us a list of changes that will occur after running the terraform plan command.

actual_provision

actual_provision:
  image: alpine:3.18.2
  stage: tf_apply
  allow_failure: true
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "main"
  script:
    - !reference [.provision, script]
    - terraform apply -auto-approve
    - sleep 20
  parallel:
    matrix: 
      - ENVIRONMENT: [dev, staging, prod]
  artifacts:
    name: vm_ip
    expire_in: "1 day"
    paths:
      - '*_vm_ip.txt'
This job does the actual provisioning of the resources in the Terraform configuration. By using -auto-approve with the terraform apply command, no further approval (typing of yes to approve) is required to make the changes. This job only runs on the main branch of the project to keep us from error when making changes to our pipeline. In other words, only confirmed changes/modifications should be merged into main. After applying the terraform configuration, we sleep for 20 seconds; this acts as a buffer time for infrastructure that is provisioned but not yet ready.

As a safety measure, we must ensure that the destroy_tf job (which destroys the resources) is available to run in every pipeline. We do this by adding a ‘allow_failure: true’ option to the actual_provision job and the ansible_conf job. This way, even if both jobs fail the destroy_tf job will still be accessible from the pipeline. After the job is complete the IP address of the VM is stored as an Artifact using the artifact keyword.

ansible_conf

ansible_conf:
  image: python:3.9.17-slim-bullseye
  stage: ans_config
  allow_failure: true
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "main"
  script:
    - cat ${ENVIRONMENT}_vm_ip.txt >> vm_ip.txt
    - chmod 750 ansible_files
    - cd ansible_files
    - sed -i 's#vm_keys#vm_keys_'"${ENVIRONMENT}"'#g' ansible.cfg
    - apt-get update && apt-get install ansible -y
    - ansible --version
    - ansible-playbook ansible-playbook.yml
  parallel:
    matrix: 
      - ENVIRONMENT: [dev, staging, prod]
This job configures each of the environments one by one which contain VMs in our case. We use a python image because ansible needs python to run (python:3.9.17-slim-bullseye). The first command renames the txt file containing the IP address of each environment to the filename in the ansible.cfg file. The next command changes the permissions on the ansible_files directory which is necessary for the ansible.cfg file in the PWD to be usable (It must not be a world-writable directory).

Then we change to the ansible_files directory and rename the SSH key to match that of each environment using the parallel:matrix: keyword. Finally, we install Ansible and run the Ansible Playbook using the ansible-playbook install_nginx.yml command.

dry_destroy_tf

dry_destroy_tf:
  image: alpine:3.18.2
  stage: tf_destroy
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "main"
  script:
    - !reference [.provision, script]
    - terraform plan -destroy
  when: manual
  parallel:
    matrix: 
      - ENVIRONMENT: [dev, staging, prod]
This job plans a “destroy” of the infrastructure that was provisioned by terraform. It is a manual job. The ‘when: manual’ keyword ensures the job never runs unless it is started manually. The first command references the .provision job to initialise and configure all that is necessary for terraform to run. By using the terraform plan -destroy command, a demo of the terraform destroy is run and the user can decide whether or not to go ahead to destroy the infrastructure using the manually-triggered actual_destroy_tf job.

actual_destroy_tf

actual_destroy_tf:
  image: alpine:3.18.2
  stage: tf_destroy
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "main"
  script:
    - !reference [.provision, script]
    - terraform destroy -auto-approve
  when: manual
  parallel:
    matrix: 
      - ENVIRONMENT: [dev, staging, prod]
This job performs the actual destruction of the infrastructure that was provisioned by terraform. It MUST be a manual job else the pipeline will create and destroy the infrastructure automatically. The ‘when: manual’ keyword ensures the job never runs unless it is started manually. The first command references the .provision job to initialise and configure all that is necessary for terraform to run. Finally, the terraform destroy command deletes/destroys the infrastructure.

If you followed well up to this point, your repository should look like.

iac-cicd
    ├── dev
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    ├── staging
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    ├── prod
    │    ├── provider.tf
    │    ├── main.tf
    │    └── backend.tf
    ├── modules
    │    └── debian_vm
    │         ├── main.tf
    │         └── variables.tf
    ├── .gitlab-ci.yml
    └── ansible_files
         ├── install_nginx.yml
         └── ansible.cfg
Below is a screenshot of a successful pipeline


ansible_conf passed logs
CONCLUSION
Congratulations on making it this far! I hope you had fun and did learn a lot. After pushing to the GitLab repository, the various environments should be provisioned and a Welcome to Nginx page should be seen… let me know in the comment section if you have any challenges and what not.

Thank you!

42


3


Prince Ogabi
Written by Prince Ogabi
42 Followers
DevOps Engineer

Follow

Recommended from Medium
Terraform Vs Terragrunt
DevOpsNinjaHub
DevOpsNinjaHub

Terraform Vs Terragrunt
Terraform and Terragrunt are both popular Infrastructure as Code (IaC) tools used for provisioning and managing infrastructure resources…

Feb 16
7 Open Source Tools You Should Be Using
C. L. Beard
C. L. Beard

in

OpenSourceScribes

7 Open Source Tools You Should Be Using
Remote access, password management, identity and network security

4d ago
91
Lists

Athletes jumping over a hurdle in a race

Staff Picks
706 stories
·
1199 saves



Stories to Help You Level-Up at Work
19 stories
·
728 saves



Self-Improvement 101
20 stories
·
2476 saves



Productivity 101
20 stories
·
2164 saves
Amazing CNCF Projects Worth Checking Out
Maryam Tavakkoli
Maryam Tavakkoli

in

Women in Technology

Amazing CNCF Projects Worth Checking Out
The Cloud Native Computing Foundation (CNCF) is home to some of the most important and useful projects in the tech industry. These projects…

Jul 6
34
Count(*) vs Count(1) in SQL.
Vishal Barvaliya
Vishal Barvaliya

in

Data Engineer

Count(*) vs Count(1) in SQL.
If you’ve spent any time writing SQL queries, you’ve probably seen both `COUNT(*)` and `COUNT(1)` used to count rows in a table. But what’s…

Mar 9
845
21
Creating custom VPC on AWS using OpenTofu
Vinod Kumar Nair
Vinod Kumar Nair

in

Level Up Coding

Creating custom VPC on AWS using OpenTofu
The OpenTofu is a Linux Foundation project which is a complete opensource Infrastructure as Code tool, an alternative to the popular…

May 5
52
Setting Up Grafana + InfluxDB + Telegraf Using Docker: A Step-by-Step Guide
Pau Santana
Pau Santana

Setting Up Grafana + InfluxDB + Telegraf Using Docker: A Step-by-Step Guide
In today’s data-driven world, monitoring system performance and collecting metrics is crucial for maintaining the health and efficiency of…

Mar 19
56
See more recommendations
Help

Status

About

Careers

Press

Blog

Privacy

Terms

Text to speech

Teams


