# Ansible AWS Lab - Technical Explanation

This repository combines Terraform and Ansible to build a small AWS lab. Terraform provisions the network and EC2 instances, and Ansible configures the managed hosts so they install a base package set and expose Collectd metrics in a Prometheus-compatible format.

## Architecture

The project has two deployment layers:

1. `terraform/` creates the AWS network and compute resources.
2. `ansible/` configures the Ubuntu hosts after they are running.

### Terraform layer

What this does:
- creates the VPC and three public subnets in `eu-central-1`
- creates the security group used by the lab hosts
- launches three EC2 instances named `control`, `node1`, and `node2`
- exposes each instance on a public IP and returns their IPs as Terraform outputs

How it works:
- the VPC module is configured in `terraform/main.tf` using `terraform-aws-modules/vpc/aws`
- the instance map is defined as:

```hcl
locals {
  instances = {
    control = module.vpc.public_subnets[0]
    node1   = module.vpc.public_subnets[1]
    node2   = module.vpc.public_subnets[2]
  }
}
```

- the security group allows inbound SSH on port `22` and Collectd metrics on port `9103`
- the AWS provider in `terraform/providers.tf` uses the `ansible-aws-lab` CLI profile and the region set in `terraform/variables.tf`
- the AMI comes from the Ubuntu 22.04 Canonical image family, selected by the `aws_ami` data source

Why this matters:
- the infrastructure exists before Ansible begins task execution
- the managed nodes have a known subnet layout and access model
- the lab remains reproducible by re-running Terraform in the same AWS profile and region

### Ansible layer

The playbook structure under `ansible/` is small and role-based. The current inventory is defined in `ansible/inventory/hosts`:

```ini
[managed_nodes]
node1
node2
```

The `control` instance is created by Terraform but is not included in the managed-node inventory. The Ansible global config sets the role path in `ansible/ansible.cfg`:

```ini
[defaults]
roles_path=roles
```

## Main components

### Common role

What this does:
- gives the managed nodes a minimal operating-system baseline
- includes optional package installation and SELinux handling when enabled through role defaults

How it works:
- `ansible/roles/common/tasks/main.yml` includes `install_packages.yml` only when `common_packages_install` is true
- it includes `selinux.yml` only when `common_selinux_disable` is true
- the role defaults define the package list and the toggle values:

```yaml
common_packages_install: false
common_selinux_disable: false
```

Why this matters:
- the role is available as a baseline configuration step but is intentionally not forced into the current deployment flow
- the repository keeps the role reusable while leaving the default lab behavior conservative

### Collectd role

What this does:
- installs Collectd on the managed nodes
- downloads and compiles the Prometheus write plugin
- publishes system metrics on port `9103`

How it works:
- the role executes `install.yml` when `collectd_install` is enabled and `remove.yml` when `collectd_remove` is set
- the task flow performs the following work:

1. installs the `collectd` package
2. installs build dependencies
3. downloads the Collectd 5.12.0 source archive
4. configures with `./configure --prefix=/usr --enable-write_prometheus`
5. runs `make`
6. copies the compiled plugin to `/usr/lib/collectd/write_prometheus.so`
7. creates `/etc/collectd.d`
8. includes the directory from `/etc/collectd/collectd.conf`
9. deploys the template from `ansible/roles/collectd/templates/prometheus.conf.j2`
10. starts the `collectd` service

The template is:

```text
LoadPlugin write_prometheus

<Plugin write_prometheus>
    Port {{ collectd_prometheus_port }}
</Plugin>
```

The default port is `9103`, stored in `ansible/roles/collectd/defaults/main.yml`.

Why this matters:
- the monitoring endpoint is reachable from the EC2 security group on a known port
- a downstream monitoring system can scrape Prometheus-style metrics without additional host-specific customization

## Data flow

The current implementation flows as follows:

1. Terraform creates the VPC, public subnets, security group, and EC2 instances.
2. Terraform outputs the instance IP information.
3. Ansible targets the managed hosts defined in `ansible/inventory/hosts`.
4. The `common` role applies the OS-level baseline if enabled.
5. The `collectd` role downloads, builds, and installs the Prometheus plugin.
6. The EC2 security group permits inbound access on port `9103`.
7. Metrics become available from the managed nodes on the configured Collectd endpoint.

## Configuration defaults

Important defaults in the repository:

- AWS region: `eu-central-1` in `terraform/variables.tf`
- Collectd Prometheus port: `9103` in `ansible/roles/collectd/defaults/main.yml`
- `common_packages_install: false` and `common_selinux_disable: false` in `ansible/roles/common/defaults/main.yml`
- `collectd_install: true` and `collectd_remove: false` in `ansible/roles/collectd/defaults/main.yml`

## Practical notes

- The Terraform configuration expects an AWS key pair named `ansible-aws-lab` to exist before deployment.
- The key pair is used by the EC2 module and is not generated by the repository.
- The environment uses public subnets and public IP addresses by design.
- The Collectd role includes a removal path, but the active default state is installation-enabled.

## Summary

The repository is a minimal AWS lab that demonstrates:

- Terraform provisioning of VPC and EC2 resources
- Ansible configuration of Ubuntu-managed nodes
- a Collectd build configured with the Prometheus write plugin
- metrics exposed on port `9103`
- a reusable infrastructure-and-configuration workflow for a small AWS environment
