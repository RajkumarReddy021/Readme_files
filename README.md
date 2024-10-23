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
```


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
###ansible/setup-airflow.yml
```yaml file

---
- hosts: all
  become: true
  tasks:
    - name: Install system packages
      yum:
        name:
          - python3
          - python3-pip
          - gcc
          - python3-devel
          - libffi-devel
          - openssl-devel
        state: present

    - name: Configure pip to use Nexus
      copy:
        dest: /home/ec2-user/.pip/pip.conf
        content: |
          [global]
          index-url = http://<nexus-url>/repository/pypi-hosted/simple
        owner: ec2-user
        mode: '0644'

    - name: Install Apache Airflow from Nexus
      pip:
        name: apache-airflow
        state: latest

    - name: Copy Airflow configuration
      copy:
        src: airflow.cfg
        dest: /etc/airflow/airflow.cfg

    - name: Start Airflow scheduler and webserver
      systemd:
        name: airflow-scheduler
        state: started
        enabled: yes
[core]
dags_folder = /usr/local/airflow/dags
plugins_folder = /usr/local/airflow/plugins
executor = LocalExecutor

[webserver]
web_server_port = 8080
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS infrastructure for Apache Airflow

Parameters:
  GoldenAMI:
    Type: String
    Description: The AMI ID of the golden Airflow AMI
  KeyPairName:
    Type: String
    Description: EC2 Key Pair for SSH access

Resources:
  AirflowInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref GoldenAMI
      KeyName: !Ref KeyPairName
      SecurityGroupIds:
        - !Ref AirflowSecurityGroup
      Tags:
        - Key: Name
          Value: "AirflowInstance"

  AirflowSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP access
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-west-2'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git 'https://your-repo-url.git'
            }
        }

        stage('Build AMI with Packer') {
            steps {
                script {
                    sh 'packer build packer/airflow-packer-template.json'
                }
            }
        }

        stage('Deploy with CloudFormation') {
            steps {
                script {
                    def amiId = sh(script: 'aws ec2 describe-images --filters "Name=name,Values=airflow-golden-ami-*" --query "Images[*].ImageId" --output text', returnStdout: true).trim()
                    sh """
                        aws cloudformation create-stack --stack-name airflow-stack \
                        --template-body file://cloudformation/airflow-infrastructure.yaml \
                        --parameters ParameterKey=GoldenAMI,ParameterValue=\${amiId} \
                        ParameterKey=KeyPairName,ParameterValue=<your-key-pair>
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
        }
    }
}


