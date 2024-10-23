# Apache Airflow Deployment on AWS
## Project Structure
```bash
mkdir -p your-repo/packer
mkdir -p your-repo/ansible
mkdir -p your-repo/cloudformation

touch your-repo/packer/airflow-packer-template.json
touch your-repo/ansible/setup-airflow.yml
touch your-repo/ansible/airflow.cfg
touch your-repo/cloudformation/airflow-infrastructure.yaml
touch your-repo/Jenkinsfile


## Code Details

### 1. Packer Configuration

**File: `packer/airflow-packer-template.json`**

```json
{
  "builders": [
    {
      "type": "amazon-ebs",
      "region": "us-west-2",
      "source_ami": "ami-xxxxxxxx",  // Your base AMI ID
      "instance_type": "t2.micro",
      "ssh_username": "ec2-user",
      "ami_name": "airflow-golden-ami-{{timestamp}}",
      "ami_description": "Golden AMI for Apache Airflow"
    }
  ],
  "provisioners": [
    {
      "type": "ansible",
      "playbook_file": "../ansible/setup-airflow.yml"
    }
  ]
}

