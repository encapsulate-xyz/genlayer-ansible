# Ansible Playbook to deploy Genlayer Node

This Ansible playbook automates the deployment and configuration of Genlayer Node. It ensures that the necessary dependencies, configuration files, and services are properly set up and running.

## Table of Contents

- [Requirements](#requirements)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Variables](#variables)
- [Usage](#usage)

## Requirements

Before using this playbook, ensure the following requirements are met:

1. **Ansible version**: Make sure you have Ansible 2.15+ installed.
2. **SSH Access**: Passwordless SSH access to all target servers.
3. **Python**: Python 3.x installed on the control node and all target hosts.
4. **Privileges**: The user running the playbook must have sudo privileges on the target machines.

## Prerequisites

**Install HashiCorp Vault**

This playbook relies on HashiCorp Vault to securely retrieve sensitive files, such as validator and node keys. Follow the [HashiCorp Vault Installation Guide](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-install) to set up Vault on your infrastructure.

**Note on Secrets Management**

The playbook dynamically retrieves secret environment file from HashiCorp Vault. The keys are expected to follow a structured path format:
`<environment>/<project>/<organization>/<type>/<file_name>`

For example:
- [`genlayer/encapsulate/validator/genlayer.secrets.env`](roles/node/templates/secrets.env.example)
- `genlayer/encapsulate/validator/wallet.yaml`

This structure ensures easy organization and secure retrieval of secrets.

## Setup

### 1. Install Ansible

If Ansible is not installed, visit the official documentation for detailed instructions on how to install Ansible on various Linux distributions:

[Ansible Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html)

### 2. Clone the repository

Clone this repository to your Ansible control node:

```bash
git clone https://github.com/encapsulate-xyz/genlayer-ansible.git
cd genlayer-ansible
```

### 3. Inventory

Define your target servers' IP address or DNS in the inventory folder, and select either `mainnet.yml` or `testnet.yml` to update.

Example for testnet.yml

```yaml
---
# maintains testnet inventory grouped by project names
all:
  vars:
    env: testnet
  children:
    genlayer:
      hosts:
        validator.genlayer.testnet.encapsulate.xyz:
          type: validator
        llm_host:
          ansible_host: validator.genlayer.testnet.encapsulate.xyz
          type: llm
        webdriver_host:
          ansible_host: validator.genlayer.testnet.encapsulate.xyz
          type: webdriver
        web_host:
          ansible_host: validator.genlayer.testnet.encapsulate.xyz
          type: web
```

## Variables

This playbook allows customization through several variables. You can define these variables in the following locations:

- **`group_vars/all.yml`**: Contains all the source urls and configurations.
- **`group_vars/mainnet.yml`** or **`group_vars/testnet.yml`**: Contains version-specific variables.
- **`group_vars/vault.yml`**: Store secret variables, such as `jwtsecret`, in this file.

### Usage

1. First, install the dependencies:

  ```bash
   ansible-galaxy install -r requirements.yml
  ```

2. Create a `ansible_vault_password` file containing ansible-vault password

3. Then run the playbook:

  ```bash
  ansible-playbook setup_node.yml -l validator.genlayer.testnet.encapsulate.xyz -i inventory/testnet.yml
  ```

4. Similarly to deploy other services

  ```bash
  ansible-playbook setup_genvm_llm.yml -l llm_host -i inventory/testnet.yml
  ansible-playbook setup_webdriver.yml -l webdriver_host -i inventory/testnet.yml
  ansible-playbook setup_genvm_web.yml -l web_host -i inventory/testnet.yml
  ```

After you run the playbook, it will ask for confirmation, displaying all the variables and the IP address or DNS of the server you are going to deploy.

Example output:

```bash
TASK [Display environment being deployed to] ***************************************************************************************************
ok: [validator.genlayer.testnet.encapsulate.xyz] => {
    "msg": [
        "Deploying to Host: validator.genlayer.testnet.encapsulate.xyz",
        "Groups: ['genlayer']",
        "Project: genlayer",
        "Environment: testnet",
        "Type: validator",
        "Version: v0.3.10",
        "Username: genlayer",
        "Service Name: genlayer",
        "Operating System: linux",
        "System Architecture: amd64"
    ]
}

TASK [Confirm deployment details] ********************************************************************************************************************
Pausing for 40 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [validator.genlayer.testnet.encapsulate.xyz]

TASK [Please confirm again] ********************************************************************************************************************
ok: [validator.genlayer.testnet.encapsulate.xyz] => {
    "msg": [
        "Deploying to Host: validator.genlayer.testnet.encapsulate.xyz",
        "Project: genlayer",
        "Environment: testnet",
        "Type: validator"
    ]
}

TASK [Confirm deployment details] **************************************************************************************************************
Pausing for 20 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [validator.genlayer.testnet.encapsulate.xyz]
```
