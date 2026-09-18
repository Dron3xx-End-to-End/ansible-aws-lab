# Ansible AWS Lab

## Overview

This repository provisions a small AWS lab with Terraform and configures Ubuntu EC2 hosts with Ansible. Terraform creates the VPC, three public subnets, security group, and EC2 instances in `eu-central-1`. Ansible then targets the managed nodes (`node1` and `node2`) and installs a minimal base package set plus Collectd configured to expose Prometheus-compatible metrics on port `9103`.

## Technologies Used

- Terraform
- AWS
- Ansible
- Ubuntu
- Collectd

## Requirements

- AWS CLI configured with the `ansible-aws-lab` profile
- Terraform 1.2 or newer
- Ansible installed locally
- An existing EC2 key pair named `ansible-aws-lab` in the target AWS account

## Repository Structure

```text
ansible-aws-lab/
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   └── hosts
│   ├── playbooks/
│   │   ├── collectd.yml
│   │   ├── common.yml
│   │   └── interfaces.yml
│   └── roles/
│       ├── collectd/
│       │   ├── defaults/
│       │   ├── handlers/
│       │   ├── tasks/
│       │   └── templates/
│       └── common/
│           ├── defaults/
│           ├── tasks/
│           └── handlers/
├── terraform/
│   ├── main.tf
│   ├── outputs.tf
│   ├── providers.tf
│   └── variables.tf
├── README.md
├── explained.md
├── .gitignore
└── .gitattributes
```

## Provisioning and Deployment

Create the AWS infrastructure from the `terraform` directory:

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

The Terraform stack creates:

- a VPC with three public subnets
- a security group that allows SSH and Collectd Prometheus traffic
- three EC2 instances named `control`, `node1`, and `node2`
- output values for the instance IP addresses

## Ansible Usage

The inventory currently targets the two managed hosts:

```ini
[managed_nodes]
node1
node2
```

From the `ansible` directory:

```bash
cd ansible
ansible managed_nodes -m ping
ansible-playbook playbooks/common.yml
ansible-playbook playbooks/collectd.yml
ansible-playbook playbooks/interfaces.yml
```

The `control` instance is created by Terraform but is not included in the managed-node inventory.

## Current Functionality

- installs a base OS package set on managed hosts when the role defaults enable it
- optionally disables SELinux when configured
- installs Collectd and its build dependencies
- downloads and compiles the Collectd source distribution
- builds the `write_prometheus` plugin
- writes a Prometheus-compatible listener on port `9103`
- exposes the metrics port through the EC2 security group

## Notes

- The repository expects an AWS key pair named `ansible-aws-lab` to exist before `terraform apply`.
- This lab is intentionally infrastructure-first: Terraform provisions the network and hosts, and Ansible configures the managed nodes after launch.
- The code does not create the key pair automatically; it only references it.
