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

## Prerequisites

- **AWS Free Tier Account**: An active AWS account.
- **WSL (Windows Subsystem for Linux)**: For running Linux commands on Windows.
- **AWS CLI**: Installed and configured with IAM credentials.
- **Ansible**: Installed on your local machine or WSL.
- **VSCode**: Configured with the "Remote - WSL" extension for development.

---

## Setup and Deployment Steps

### 0️⃣ Base Setup

- **Create an AWS Free Tier Account**: Sign up at [aws.amazon.com](https://aws.amazon.com).
- **Install WSL on Windows**:
  - Enable WSL and install a Linux distribution (e.g., Ubuntu) via the Microsoft Store.
- **Install AWS CLI on WSL**:
  ```bash
  curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
  unzip awscliv2.zip
  sudo ./aws/install
  aws --version
