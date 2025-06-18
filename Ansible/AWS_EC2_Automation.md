1. **IAM Setup** for secure AWS access  
2. **Ansible Vault** for credential encryption  
3. **EC2 Instance Creation** (Ubuntu & Amazon Linux)  
4. **Password-less SSH Setup**  
5. **Targeted Instance Control** (Stop Ubuntu instances)  


## **1. Local Setup**
### **Install Ansible & Dependencies**
```bash
# Install Ansible
sudo apt update && sudo apt install -y ansible

# Install boto3 (AWS SDK for Python) and AWS Ansible collection
pip install boto3
ansible-galaxy collection install amazon.aws
```

### **Verify Installation**
```bash
ansible --version  # Should show >= 2.9
python3 -c "import boto3; print(boto3.__version__)"  # Should show version
```

---

## **2. AWS IAM Configuration**
### **IAM User Creation**
1. Created IAM user `ansible-user` with:  
   - **Programmatic access** (Access key + Secret key)  
   - Attached policies:  
     - `AmazonEC2FullAccess`  
     - `AmazonSSMManagedInstanceCore` (for SSM fallback)  

### **Why Instances Were Created in Root Account**
- Your IAM user had `AmazonEC2FullAccess`, but the **default VPC/subnet** settings inherited the root account’s ownership.  
- **Fix**: Explicitly specify a non-root-owned VPC/subnet in the playbook.

---

## **3. Ansible Vault Setup**
### **Generate Vault Password**
```bash
openssl rand -base64 2048 > vault.pass
chmod 600 vault.pass  # Restrict access
```

### **Store AWS Credentials Securely**
```bash
ansible-vault create secret.yml --vault-password-file vault.pass
```
**Content of `secret.yml`:**
```yaml
vault_ec2_access_key: "AKIAXXXXXXXXXXXXXXXX"
vault_ec2_secret_key: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

## **4. EC2 Provisioning Playbook
### **Playbook: `ec2_create.yml`**
```yaml
---
- name: Create EC2 instances
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - secret.yml

  tasks:
    - name: Launch instances
      amazon.aws.ec2_instance:
        name: "{{ item.name }}"
        key_name: "ubuntu"
        instance_type: t2.micro
        security_group: "default"
        region: ap-south-1
        aws_access_key: "{{ vault_ec2_access_key }}"
        aws_secret_key: "{{ vault_ec2_secret_key }}"
        image_id: "{{ item.image }}"
        network:
          assign_public_ip: true
      loop:
        - { image: "ami-0b09627181c8d5778", name: "manage-node-1" }  # Ubuntu
        - { image: "ami-0f918f7e67a3323f0", name: "manage-node-2" }  # Amazon Linux
      register: ec2_results

    - name: Print instance IDs
      debug:
        var: ec2_results.results[].instance_ids
```

### **Run Playbook**
```bash
ansible-playbook ec2_create.yml --vault-password-file vault.pass
```

---

## **5. Password-less SSH Setup**
### **Copy SSH Key to Instances**
```bash
ssh-keygen -y -f ~/Downloads/ubuntu.pem > ~/.ssh/ubuntu.pub  # Extract public key
ssh-copy-id -i ~/.ssh/ubuntu.pub -o "IdentityFile=~/Downloads/ubuntu.pem" ubuntu@<PUBLIC_IP>
```

### **Test SSH Access**
```bash
ssh -i ~/Downloads/ubuntu.pem ubuntu@<PUBLIC_IP>  # Should log in without password
```

---

## **6. Stop Ubuntu Instances Playbook**
### **Playbook: `stop_ubuntu.yml`**
```yaml
---
- name: Stop Ubuntu instances
  hosts: localhost
  connection: local
  gather_facts: false
  vars_files:
    - secret.yml

  tasks:
    - name: Find Ubuntu instances
      amazon.aws.ec2_instance_info:
        region: ap-south-1
        filters:
          "tag:Name": "manage-node-*"
          "image-id": "ami-0b09627181c8d5778"  # Ubuntu AMI
      register: ubuntu_instances

    - name: Stop instances
      amazon.aws.ec2_instance:
        instance_ids: "{{ ubuntu_instances.instances[].instance_id }}"
        region: ap-south-1
        aws_access_key: "{{ vault_ec2_access_key }}"
        aws_secret_key: "{{ vault_ec2_secret_key }}"
        state: stopped
```

### **Run Playbook**
```bash
ansible-playbook stop_ubuntu.yml --vault-password-file vault.pass
```

---

## **7. Repository Structure**
```
.
├── README.md               # Project report (this file)
├── ansible.cfg             # Ansible configuration
├── inventory.ini           # Dynamic inventory
├── ec2_create.yml          # Create EC2 instances
├── stop_ubuntu.yml         # Stop Ubuntu instances
├── secret.yml              # Encrypted credentials
├── vault.pass              # Vault password file
└── .gitignore              # Exclude vault.pass and secret.yml
```

---

## **Key Learnings**
1. **IAM Best Practices**: Avoid root-owned resources by specifying VPCs.  
2. **Ansible Vault**: Securely manage secrets without hardcoding.  
3. **Targeted Control**: Use `ec2_instance_info` to filter instances by tags/AMI.  

---

## **Next Steps**
1. **Dynamic Inventory**: Use `aws_ec2` plugin for real-time instance tracking.  
2. **Terraform Integration**: Manage infrastructure-as-code alongside Ansible.  

Upload this report to your repo with:
```bash
git add README.md && git commit -m "Add project report"
git push origin main
```
