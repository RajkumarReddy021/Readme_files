# Apache Airflow Deployment on AWS
## Project Structure
/your-repo │ ├── packer/ │ └── airflow-packer-template.json # Packer template for building the AMI │ ├── ansible/ │ ├── setup-airflow.yml # Ansible playbook for Airflow setup │ └── airflow.cfg # Airflow configuration file │ ├── cloudformation/ │ └── airflow-infrastructure.yaml # CloudFormation template for AWS resources │ └── Jenkinsfile # Jenkins pipeline definition
