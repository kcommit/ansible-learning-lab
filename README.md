# Ansible Learning Lab

This repository documents my hands-on Ansible learning journey. It contains personal study notes, lab configurations, command explanations, assignments, and practical automation projects.

The goal is to build a clear record of what I learn while progressing from Ansible fundamentals to reusable playbooks, roles, and real-world automation.

## Learning Objectives

- Understand Ansible control nodes and managed nodes
- Configure secure SSH access between Linux systems
- Build and validate static inventories
- Understand `ansible.cfg` and configuration precedence
- Run ad hoc commands and use Ansible modules
- Write idempotent YAML playbooks
- Organize variables, templates, handlers, and roles
- Complete practical assignments and projects

## Current Lab Environment

| Role | Host or group | IP address | Purpose |
| --- | --- | --- | --- |
| Control node | `PracticeLab` | `192.168.1.233` | Runs Ansible commands and playbooks |
| Managed node | `web-server` | `192.168.1.154` | Web-server practice |
| Managed node | `app-server` | `192.168.1.190` | Application-server practice |
| Managed node | `db-server` | `192.168.1.215` | Database-server practice |

### Control-node software

- Rocky Linux 9.8
- Python 3.9.25 at `/usr/bin/python3`
- Ansible community package 7.7.0
- Ansible Core 2.14.18
- Administrative account: `ansibleadmin`

## Topics Documented So Far

- Agent-based and agentless automation
- Why Ansible is used
- Push and pull operating models
- Idempotency
- Rocky Linux control-node preparation
- Python and EPEL verification
- Ansible installation and local testing
- Project-level `ansible.cfg`
- Ansible configuration precedence
- Static inventory groups, child groups, aliases, and variables
- Inventory IP and hostname range patterns



## Repository Structure

The current files can remain in place while the repository is small. As more practical work is added, the repository will gradually be reorganized into the following structure:

```text
ansible-learning-lab/
├── README.md
├── .gitignore
├── study-notes/
│   ├── english/
│   │   ├── 01-ansible-introduction.md
│   │   ├── 02-control-node-setup.md
│   │   ├── 03-ansible-configuration.md
│   │   └── 04-static-inventory.md
│   │
│   └── roman-urdu/
│       ├── 01-ansible-introduction.md
│       ├── 02-control-node-setup.md
│       ├── 03-ansible-configuration.md
│       └── 04-static-inventory.md
│
├── automation/
│   ├── ansible.cfg
│   ├── inventory/
│   │   ├── hosts.ini
│   │   ├── group_vars/
│   │   │   └── all.yml
│   │   └── host_vars/
│   │       ├── web-server.yml
│   │       ├── app-server.yml
│   │       └── db-server.yml
│   │
│   ├── playbooks/
│   │   ├── ping.yml
│   │   ├── packages.yml
│   │   ├── users.yml
│   │   └── services.yml
│   │
│   ├── roles/
│   ├── templates/
│   ├── files/
│   └── vault/
│
├── labs/
│   ├── 01-connectivity-test/
│   ├── 02-ad-hoc-commands/
│   ├── 03-package-management/
│   ├── 04-user-management/
│   ├── 05-file-management/
│   └── 06-service-management/
│
├── assignments/
│   ├── assignment-01-user-management/
│   ├── assignment-02-package-management/
│   ├── assignment-03-web-service/
│   └── assignment-04-system-update/
│
├── projects/
│   ├── 01-web-server-deployment/
│   ├── 02-three-tier-application/
│   ├── 03-linux-security-baseline/
│   └── 04-patch-management/
│
└── resources/
    ├── diagrams/
    ├── screenshots/
    ├── pdfs/
    └── references/
```

### Folder purposes

| Folder | Purpose |
| --- | --- |
| `study-notes/` | English and Roman Urdu explanations |
| `automation/` | Working Ansible configuration, inventory, playbooks, and roles |
| `labs/` | Small guided exercises for individual concepts |
| `assignments/` | Tasks completed independently to test understanding |
| `projects/` | Larger end-to-end automation scenarios |
| `resources/` | Diagrams, PDFs, screenshots, and reference material |

`assignments` is preferred over `homework` because it sounds more professional and works well for a public portfolio. A lab is a small focused exercise, while a project combines several skills into one complete solution.

### Keeping empty directories in Git

Git does not track empty directories. Until a directory contains real work, a `.gitkeep` placeholder can be added:

```bash
touch labs/.gitkeep
touch assignments/.gitkeep
touch projects/.gitkeep
```

Remove the corresponding `.gitkeep` file after adding real files to that directory.

## Repository Navigation

- Start with `study-notes/` when reviewing a concept.
- Use `automation/` for the active Ansible configuration, inventory, playbooks, and roles.
- Open `labs/` for small guided exercises.
- Open `assignments/` for tasks completed independently.
- Open `projects/` for complete end-to-end automation work.
- Use `resources/` for supporting PDFs, screenshots, diagrams, and references.

## Basic Usage

From the repository root, enter the working Ansible directory:

```bash
cd automation
```

Check which configuration file Ansible is using:

```bash
ansible --version
```

Validate the inventory:

```bash
ansible-inventory --graph
ansible-inventory --list
```

Test connectivity to all managed nodes:

```bash
ansible all -m ping
```

Run a simple ad hoc command:

```bash
ansible all -m command -a "hostname"
```

When privilege escalation is required:

```bash
ansible all -b -m command -a "whoami"
```

## Planned Labs and Assignments

1. Configure SSH key-based authentication
2. Verify connectivity with the `ping` module
3. Collect managed-node facts
4. Manage packages with `dnf`
5. Create users and groups
6. Manage files, ownership, and permissions
7. Control services with `systemd`
8. Deploy an Nginx or Apache web page
9. Use variables, loops, conditions, and handlers
10. Create templates with Jinja2
11. Encrypt secrets with Ansible Vault
12. Convert a playbook into an Ansible role

## Planned Projects

- Multi-node web-server deployment
- Three-tier application configuration
- Linux user and security-baseline automation
- Patch-management workflow
- Service monitoring and validation playbook
- Reusable role-based server provisioning

## Important Security Notes

- Do not commit passwords, private SSH keys, vault passwords, tokens, or other secrets.
- Use SSH key authentication instead of permanent password-based automation.
- Keep private keys outside the repository.
- Use Ansible Vault for sensitive variables when required.
- Review changes with `--check` and `--diff` when the selected module supports them.

## Useful Validation Commands

```bash
# Display the active Ansible configuration
ansible --version

# Display changed configuration values
ansible-config dump --only-changed

# Validate inventory syntax and structure
ansible-inventory --graph

# Check playbook syntax without running it
ansible-playbook playbook.yml --syntax-check

# Preview supported changes
ansible-playbook playbook.yml --check --diff
```

## Progress

- [x] Prepare the Rocky Linux control node
- [x] Install and verify Ansible
- [x] Document `ansible.cfg`
- [x] Create and document a static inventory
- [ ] Configure SSH key authentication to all managed nodes
- [ ] Verify all managed nodes with the Ansible `ping` module
- [ ] Complete the first ad hoc command lab
- [ ] Write the first playbook
- [ ] Complete the first assignment
- [ ] Complete the first end-to-end project

This repository is a work in progress. The README, notes, examples, and progress checklist will be updated as new labs, assignments, and projects are completed.
