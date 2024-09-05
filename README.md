### Pre-Requisites
For this project, we will be using the Google Cloud Platform for creating the Infrastructure. 
* GCP-Service account credential
* GitLab Personal Access Token 

### Steps

#### Step 1: Generate the Service Account Credentials
We need to configure a Service Account with Editor Permission that the pipelines will use to deploy the resources.
1. Go to the IAM in your project, and create a Service Account
2. Choose a name
3. Give the Editor Permission and create
4. Now, let’s create a key for the Service Account. Please select the service account, go to the Key section, and add a key in JSON format. Save it for we use in Gitlab CI
5. Go to the GitLab project you created for this project
6. Click on ‘Settings’ the last option in the sidebar of the project
7. Click on ‘CI/CD’, then expand the variables section of the page
8. Click ‘Add Variable’, and add the variable for CREDENTIAL. The CREDENTIAL variable must be of type File

#### Step 2: Create a personal access token in GitLab
We will generate the Personal Access Token
1. On the left sidebar, select your avatar
2. Select Edit profile
3. On the left sidebar, select Access tokens
4. Select Add new token
5. Enter a name and expiry date for the token
6. Select the desired scopes
7. Select Create personal access token
8. Save the personal access token somewhere safe. After you leave the page, you no longer have access to the token
9. Go to the GitLab project you created for this project
10. Click on ‘Settings’ the last option in the sidebar of the project
11. Click on ‘CI/CD’, then expand the variables section of the page
12. Click ‘Add Variable’, and add the variable GITLAB_ACCESS_TOKEN

    ![image](https://github.com/user-attachments/assets/91a1a551-5004-4686-9f7c-aeb02d08b617)


#### Step 3 : Terraform
The structure of your directory should be
![image](https://github.com/user-attachments/assets/06eb022c-38a5-45ae-b1ac-a20167e13788)

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

modules/debian_vm/variables.tf
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

dev/provider.tf
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
```
# Use the debian module to provision 
module "debian_vm" {
  source = "../modules/debian_vm"
  
# Input variables
  environment = "dev"
}
```


### Step 4: ANSIBLE
Create a folder named ansible_files in the project and ensure that the permission is such that is not a world-writable directory (else, the playbook won’t run. More on that here). Add install_nginx.yml file which is responsible for performing the configuration itself. Next add the ansible.cfg which sets the ansible configuration and so the default configuration file /etc/ansible/ansible.cfg will not be used.

ansible_files/ansible.cfg
```
[defaults]

inventory = ../vm_ip.txt
private_key_file = ../.keys/vm_keys
host_key_checking = False
remote_user = meharuser
inventory sets the path to the file that contains the list of IPs that should be SSHed into
private_key_file sets the path to the private SSH key file
host_key_checking sets the SSH option StrictHostKeyChecking=no
remote_user sets the username to authenticate with the VM.
```

ansible_files/install_nginx.yml
```
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
```

![image](https://github.com/user-attachments/assets/f3936813-5d4a-472d-86ed-9bf7edac6951)

### Step 5: GitLab PIPELINE

```
stages:
  - ssh_gen
  - tf_plan
  - tf_apply
  - ans_config
  - tf_destroy

variables:
  PROJECT_ID: 60651700
  TF_USERNAME: meharnafis
  TF_PASSWORD: ${GITLAB_ACCESS_TOKEN}
  TERRAFORM_VERSION: 1.5.0

gen_ssh_keys:
  image: bash:latest
  stage: ssh_gen
  script:
    - apk update && apk add openssh-client
    - mkdir .keys && cd ./.keys
    - ssh-keygen -f vm_keys_${ENVIRONMENT} -q -t rsa -N "" && echo "Keys successfully generated"
  parallel:
    matrix: 
      - ENVIRONMENT: [dev]
  artifacts:
    name: ssh-keys
    expire_in: "1 day"
    paths: 
      - .keys

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
      - ENVIRONMENT: [dev]

actual_provision:
  image: alpine:3.18.2
  stage: tf_apply
  allow_failure: true
  only:
    variables:
      - $CI_COMMIT_REF_NAME == "main"
  script:
    - !reference [.provision, script]
    - terraform apply -auto-approve -lock=false
    - sleep 20
  parallel:
    matrix: 
      - ENVIRONMENT: [dev]
  artifacts:
    name: dev_vm_ip
    expire_in: "1 day"
    paths:
      - '*_vm_ip.txt'

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
    - ansible-playbook install_nginx.yml
  parallel:
    matrix: 
      - ENVIRONMENT: [dev]

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
      - ENVIRONMENT: [dev]

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
      - ENVIRONMENT: [dev]

```


Your repository should look like if you followed well up to this point.

![image](https://github.com/user-attachments/assets/c78d4021-567d-4054-9038-72797579d8ed)

Below is the screenshot of the repo
![image](https://github.com/user-attachments/assets/e1fb39cb-f0d2-4ff7-b710-d3326be38476)


Below is the screenshot of the successful pipeline
![image](https://github.com/user-attachments/assets/86b886ad-a50b-4373-b200-2e7890608090)
