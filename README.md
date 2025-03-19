# Ansible-AWS Project: Automated Deployment of a Load-Balanced Web Application with MySQL

This project automates the deployment of a web application on AWS using Ansible. It provisions a MySQL database server, two Flask web servers, and an AWS Classic Load Balancer (CLB) to distribute traffic. The setup includes dynamic inventory management, secure credential handling with Ansible Vault, and email notifications upon completion—all orchestrated through Ansible playbooks.

## Project Overview

- **Infrastructure**:
  - 1 MySQL database server (`db-server`)
  - 2 Flask web servers (`web-server`)
  - 1 AWS Classic Load Balancer (CLB)
- **Automation Goals**:
  - Provision EC2 instances
  - Configure security groups
  - Set up MySQL and Flask applications
  - Deploy the load balancer
  - Send deployment notifications
- **Security**:
  - Credentials encrypted with Ansible Vault
  - Minimal permissions via security groups
    
- IMAGE
  
## Prerequisites

- **AWS Free Tier Account**: An active AWS account.
- **WSL (Windows Subsystem for Linux)**: For running Linux commands on Windows.
- **AWS CLI**: Installed and configured with IAM credentials.
- **Ansible**: Installed on your local machine or WSL.
- **VSCode**: Configured with the "Remote - WSL" extension for development.

## Setup and Usage

See [docs/SETUP.md](docs/SETUP.md) for detailed setup instructions.


## Learnings

- Dynamic inventory management with meta: refresh_inventory
- Security group rule handling
- Secure credential storage


## Future Improvements

- HTTPS support
- Enhanced database automation
- CloudWatch monitoring
