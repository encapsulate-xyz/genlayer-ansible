# Ansible Playbook to deploy Genlayer Node

This Ansible playbook automates the deployment and configuration of Genlayer Node. It ensures that the necessary dependencies, configuration files, and services are properly set up and running.

## Table of Contents

- [Requirements](#requirements)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Variables](#variables)
- [Usage](#usage)
- [Testing](#testing)

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

## Testing

Pull requests run the playbooks with [Molecule](https://ansible.readthedocs.io/projects/molecule/) in CI (the shared `molecule` job in `encapsulate-xyz/.github`). The playbook runs unchanged, against fake secrets.

```
molecule/
  node/                 the playbooks in a fresh Ubuntu 24.04 container (runs in CI)
    molecule.yml          container, inventory and test steps
    converge.yml          runs setup_node.yml, setup_alloy.yml and setup_webdriver_healthcheck.yml
    side_effect.yml       runs what changes the host on purpose: sync_node.yml from a tiny snapshot,
                          the post-deploy check against stand-in services, setup_node.yml with GenVM
    verify.yml            checks what they left on the host: services, files, secrets, config, firewall
    host_vars/            the test host, same as the validator in inventory/testnet.yml
  node-server/          the same converge.yml, side_effect.yml and verify.yml on a real throwaway server (run locally)
    molecule.yml
    prepare.yml           allows ssh in ufw, points Grafana Cloud at the server, starts a fake Vault
    cleanup.yml           removes what the test installed, and resets ufw
    host_vars/server.yml  reads the server from MOLECULE_SERVER and MOLECULE_SERVER_KEY
  shared/
    Dockerfile.j2         Ubuntu 24.04 with systemd and SSH, like our servers
    group_vars/all.yml    fake values for group_vars/vault.yml, and test settings
    vars/fake_secrets.yml what the fake HCP Vault serves
    prepare.yml           starts the fake HCP Vault container
    cleanup.yml           removes it
  untested.yml          tasks no scenario runs, each with a reason
```

Each test runs converge.yml twice; the second run must change nothing, so a deploy with no changes never restarts a service. Alloy's config sends to Grafana Cloud, so every scenario points those names at the test host, and converge.yml stops before deploying if a name in the config resolves anywhere else. CI also fails if a task neither runs in a scenario nor is listed in `molecule/untested.yml`.

To run a scenario yourself, install the versions CI uses and stub the encrypted vault as CI does:

```bash
pip install ansible==11.13.0 molecule==26.10.0 "molecule-plugins[docker]==26.10.12" docker==7.2.0
printf -- '---\n{}\n' > group_vars/vault.yml

molecule test -s node

MOLECULE_SERVER=<address> MOLECULE_SERVER_KEY=<private key path> molecule test -s node-server

git checkout group_vars/vault.yml
```

`node-server` deploys a real node to the server, so only point it at a throwaway server you can reach as root over SSH.
