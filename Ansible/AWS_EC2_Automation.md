## Project Overview
1. **IAM Setup** for secure AWS access  
2. **Ansible Vault** for credential encryption  
3. **EC2 Instance Creation** (Ubuntu & Amazon Linux)  
4. **Password-less SSH Setup**  
5. **Targeted Instance Control** (Stop Ubuntu instances)  


## **Local Setup**
### **Install Ansible & Dependencies**
# Install Ansible
sudo apt update && sudo apt install -y ansible

# Install boto3 (AWS SDK for Python) and AWS Ansible collection
sudo apt install python3-boto3
ansible-galaxy collection install amazon.aws


### **Verify Installation**
ansible --version  # Should show >= 2.9
python3 -c "import boto3; print(boto3.__version__)"  # Should show version

## **AWS IAM Configuration**
### **IAM User Creation**
 Created IAM user `ansible-user` with:  
  -  Access key + Secret key  
  - Attached policies:  
     - AmazonEC2FullAccess


### **Understanding `ansible.cfg` Settings**
[defaults]
inventory = ./inventory.ini         # Path to your inventory file

private_key_file = ~/Downloads/ubuntu.pem  # Default SSH private key

host_key_checking = False           # Disables SSH host key verification

### **1. `inventory = ./inventory.ini`**
- Specifies the location of your inventory file.    
  - Ansible uses this file to determine which hosts to manage.  

### **2. `private_key_file = ~/Downloads/ubuntu.pem`**
- Sets the default SSH private key for connecting to instances.    
  - The key must match the one assigned to your EC2 instances.  
  - Strict permissions are critical:  
    chmod 400 ~/Downloads/ubuntu.pem  # Prevents unauthorized access
    
- **Why Avoid Hardcoding in Playbooks**:  
  Centralizing the key path in `ansible.cfg` makes playbooks portable across environments.


### **Inventory File Structure**
[linux_nodes]

amazon-linux-1 ansible_host=13.233.132.246 ansible_user=ec2-user

[ubuntu_nodes]

ubuntu-node-1 ansible_host=13.235.94.92 ansible_user=ubuntu
ubuntu-node-2 ansible_host=15.206.68.242 ansible_user=ubuntu

[all:vars]

ansible_ssh_private_key_file=~/Downloads/ubuntu.pem
ansible_python_interpreter=/usr/bin/python3


### **1. Host Group Definitions**
#### **`[linux_nodes]` and `[ubuntu_nodes]`**
-  Organizes hosts into logical groups for targeted operations.  
- **Example**:  
  ansible ubuntu_nodes -m ping  # Only targets Ubuntu instances

  `ansible_host` == Public/IPv4 address of the instance
  `ansible_user` == SSH user (OS-specific: `ec2-user` for Amazon Linux, `ubuntu` for Ubuntu) 

### **2. Group Variables (`[all:vars]`)**
Applies to **all hosts** in the inventory.

#### **`ansible_ssh_private_key_file=~/Downloads/ubuntu.pem`**
- Specifies the default SSH private key for authentication.    
  - Avoids repeating `--private-key` in CLI commands.  
  - Ensure strict permissions:  
    chmod 400 ~/Downloads/ubuntu.pem

#### **`ansible_python_interpreter=/usr/bin/python3`**
- Sets the Python interpreter path on remote hosts.  
- **Why**:  
  - Ansible requires Python on target hosts to execute modules.  
  - Prevents failures if the default `python` points to Python 2 (deprecated).  
- **OS Compatibility**:  
  - Amazon Linux 2/2023 and Ubuntu 20.04+ use `/usr/bin/python3`.  
  - Older systems may need `/usr/bin/python` (Python 2).  



### **3. `host_key_checking = False`**
- Disables SSH host key verification.  
  -  Avoids manual confirmation when connecting to new hosts.  
  -  But Reduces security (vulnerable to MITM attacks).  
- **When to Use??**:  
  - Recommended for **development/testing** (not production).  
  - For production, use:  
    host_key_checking = True



## **3. Ansible Vault Setup**
### **Generate Vault Password**
openssl rand -base64 2048 > vault.pass
chmod 600 vault.pass  # Restrict access

### **Store AWS Credentials Securely**
ansible-vault create secret.yml --vault-password-file vault.pass

**Content of `secret.yml`:**
ec2_access_key: "AKIAXXXXXXXXXXXXXXXX"
ec2_secret_key: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

## **4. EC2 Provisioning Playbook
### **Playbook: `ec2_create.yml`**
---
- name: Provisioning EC2 Instances
  hosts: localhost
  connection: local

  vars_files:
    - secrets.yml

  vars:
    ec2_access_key: "{{ vault_ec2_access_key }}"
    ec2_secret_key: "{{ vault_ec2_secret_key }}"

  tasks:
  - name: Create EC2 instances
    amazon.aws.ec2_instance:
      name: "{{ item.name }}"
      key_name: "ubuntu"
      instance_type: t2.micro
      security_group: "default"
      region: ap-south-1
      aws_access_key: "{{ec2_access_key}}"
      aws_secret_key: "{{ec2_secret_key}}"
      network:
        assign_public_ip: true
      image_id: "{{ item.image }}"
    
    loop:
      - { image: "ami-0b09627181c8d5778", name: "manage-node-1" }
      - { image: "ami-0f918f7e67a3323f0", name: "manage-node-2" }
      - { image: "ami-0f918f7e67a3323f0", name: "manage-node-3" }


### **Run Playbook**
ansible-playbook ec2_create.yml --vault-password-file vault.pass
![image](https://github.com/user-attachments/assets/1ca230ee-b935-478d-a307-2b023bfccfcf)
![image](https://github.com/user-attachments/assets/77a15f21-0dbc-46e2-b7e2-e3c3bb4307b1)

## **5. Password-less SSH Setup**
Check if SSH is enabled:

![image](https://github.com/user-attachments/assets/bd26a03f-8bae-4278-9ee3-6871d4510eba)


### **Test SSH Access**
ssh -i ~/Downloads/ubuntu.pem ubuntu@<PUBLIC_IP>  # Should log in without password
![image](https://github.com/user-attachments/assets/d3da962e-040e-4d7e-93e6-7001921734bb)


## **6. Stop Ubuntu Instances Playbook**
### **Playbook: `stop_ubuntu.yml`**
---
- name: Stop the Ec2 instance
  hosts: all
  become: true
  gather_facts: true # required to detect OS family
  
  tasks:
    - name: Shutdown Ubuntu Instances only
      ansible.builtin.command: /sbin/shutdown -t now
      when:
        ansible_facts['os_family'] == "Debian"

      ignore_errors: true # continues even if some instances fail


### **Run Playbook**
ansible-playbook stop_ubuntu.yml --vault-password-file vault.pass
![image](https://github.com/user-attachments/assets/e4a23201-1054-40d2-a295-d54d51c32878)
![image](https://github.com/user-attachments/assets/a08a7eb8-ff99-4ff7-9663-0ac6d0fe5c42)

## Learnings:

### **1. IAM Best Practices** 
  Created a dedicated IAM user (`ansible-user`) with only `AmazonEC2FullAccess` instead of using root credentials.  
  -  Prevents accidental deletion/modification of non-EC2 resources (e.g., S3, RDS).
  -  Root credentials can compromise the entire AWS account if leaked. IAM users provide audit trails via CloudTrail.


### **2. Secure Credential Management with Ansible Vault**
  Stored AWS keys in `secret.yml` encrypted with `ansible-vault`, using a randomly generated password (`vault.pass`).  
  - `chmod 600 vault.pass` restricts access to the password file.  

### **3. Infrastructure Provisioning**  
  Used `amazon.aws.ec2_instance` with `loop` to deploy multiple instances automically.  

### **4. Password-less SSH for Automation**
- **Key-Based Authentication**:  
  Deployed public keys via `ssh-copy-id` to enable Ansible’s SSH connections without passwords.
  Private key (`ubuntu.pem`) must have `400` permissions.  
- **Cross-OS Compatibility**:  
  Configured SSH for both Ubuntu (`ubuntu` user) and Amazon Linux (`ec2-user`).

### **5. Targeted Instance Control**
- **OS-Specific Operations**:  
  Used `ansible_facts['os_family']` to selectively stop Ubuntu instances (`Debian` family) while ignoring others.    
- **Graceful Shutdown**:  
  `/sbin/shutdown -h now` ensures services terminate cleanly (better than AWS’s forced stop).

### **6. Debugging and Error Handling**
- **Playbook Resilience**:  
  Added `ignore_errors: true` to continue execution even if some instances fail to stop.  
- **Validation Steps**:  
  - Checked security group rules for SSH access.  

### **7. AWS Integration**
- **Boto3 Dependency**:  
  Installed `python3-boto3` for Ansible’s AWS module compatibility.  
- **Collection Management**:  
  Used `ansible-galaxy collection install amazon.aws` to access modern AWS modules.
