# Ansible Codebase Visual Flow Diagram

## 📊 Variable Flow & Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                        ANSIBLE CONFIGURATION                     │
├─────────────────────────────────────────────────────────────────┤
│ ansible.cfg                                                     │
│ ├── inventory = inventory                                       │
│ ├── private_key_file = ~/.ssh/ansiblepoc                      │
│ └── remote_user = bhedil                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          INVENTORY                              │
├─────────────────────────────────────────────────────────────────┤
│ [apache] Group              │  [db_servers] Group               │
│ ├── 172.31.14.83           │  └── bhedil@172.31.6.78          │
│ └── bhedil@172.31.6.78     │                                   │
└─────────────────────────────────────────────────────────────────┘
              │                               │
              ▼                               ▼
┌─────────────────────────┐        ┌─────────────────────────┐
│    HOST_VARS            │        │    HOST_VARS            │
│  172.31.14.83.yml       │        │ bhedil@172.31.6.78.yml  │
├─────────────────────────┤        ├─────────────────────────┤
│ apache_package: apache2 │        │ apache_package: httpd   │
│ apache_service: apache2 │        │ apache_service: httpd   │
│ passwd_auth: no         │        │ passwd_auth: no         │
│ ssh_template_file:      │        │ ssh_template_file:      │
│   sshd_config_ubuntu.j2 │        │   sshd_config_redhat.j2 │
└─────────────────────────┘        └─────────────────────────┘
              │                               │
              └───────────────┬───────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      SITE.YML EXECUTION                         │
├─────────────────────────────────────────────────────────────────┤
│ 1. Pre-tasks: Update packages (all hosts)                      │
│    ├── RedHat: dnf update                                      │
│    └── Ubuntu: apt upgrade                                     │
│                                                                 │
│ 2. Base Role (all hosts)                                       │
│    ├── Create user: bhedil                                     │
│    ├── Deploy sudo permissions                                 │
│    ├── Add SSH key                                             │
│    └── Configure SSH (uses ssh_template_file variable)         │
│                                                                 │
│ 3. DB Servers Role (db_servers group only)                     │
│    └── Install MariaDB (RedHat only)                          │
│                                                                 │
│ 4. Apache Role (apache group only)                            │
│    └── Uses external role: geerlingguy.apache                 │
└─────────────────────────────────────────────────────────────────┘
```

## 🔄 Role Dependency Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                          BASE ROLE                              │
├─────────────────────────────────────────────────────────────────┤
│ Runs on: ALL HOSTS                                              │
│ Variables Used: ssh_template_file                               │
│                                                                 │
│ Tasks:                           Files:                         │
│ ├── Create user bhedil    ──────► sudoer_bhedil               │
│ ├── Deploy sudo file      ──────► SSH public key              │  
│ ├── Add SSH key          ──────► sshd_config_ubuntu.j2        │
│ └── Configure SSH        ──────► sshd_config_redhat.j2        │
│                                                                 │
│ Handlers Triggered: restart_sshd                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       WEB_SERVERS ROLE                         │
├─────────────────────────────────────────────────────────────────┤
│ Runs on: APACHE GROUP HOSTS                                     │
│ Variables Used: apache_package, apache_service                  │
│                                                                 │
│ Tasks:                           Files:                         │
│ ├── Install Apache pkg    ──────► (uses apache_package var)    │
│ ├── Start Apache service  ──────► (uses apache_service var)    │
│ ├── Config ServerAdmin    ──────► (RedHat only)               │
│ └── Deploy HTML page      ──────► default_site.html           │
│                                                                 │
│ Handlers Triggered: restart_apache                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DB_SERVERS ROLE                          │
├─────────────────────────────────────────────────────────────────┤
│ Runs on: DB_SERVERS GROUP                                       │
│ Condition: ansible_distribution == "RedHat"                     │
│                                                                 │
│ Tasks:                                                          │
│ └── Install MariaDB (dnf package manager)                      │
└─────────────────────────────────────────────────────────────────┘
```

## 🎯 Server Assignment Matrix

```
┌────────────────┬─────────────┬─────────────┬──────────────────────┐
│ Server         │ Groups      │ OS Type     │ Variables Applied    │
├────────────────┼─────────────┼─────────────┼──────────────────────┤
│ 172.31.14.83   │ [apache]    │ Ubuntu      │ apache2, apache2,    │
│                │             │             │ sshd_config_ubuntu   │
├────────────────┼─────────────┼─────────────┼──────────────────────┤
│bhedil@172.31.6.78│[apache]    │ RedHat      │ httpd, httpd,        │
│                │[db_servers] │             │ sshd_config_redhat   │
└────────────────┴─────────────┴─────────────┴──────────────────────┘
```

## 🔗 Variable Interaction Chain

```
INVENTORY GROUPS ──► ROLE ASSIGNMENT ──► VARIABLE LOADING ──► TASK EXECUTION
       │                    │                   │                 │
       │                    │                   │                 │
   [apache]     ──►    web_servers    ──►   apache_package   ──►  pkg install
   [db_servers] ──►    db_servers     ──►   (none specific)  ──►  mariadb install
   [all]        ──►    base           ──►   ssh_template_file ──► SSH config
```

## 🏷️ Tag Execution Flow

```
ansible-playbook site.yml --tags "apache"
                   │
                   ▼
            ┌─────────────┐
            │   ALWAYS    │ ──► User creation, updates (always run)
            └─────────────┘
                   │
                   ▼
            ┌─────────────┐
            │   APACHE    │ ──► Apache installation and config
            └─────────────┘
                   │
                   ▼
            ┌─────────────┐
            │  REDHAT/    │ ──► OS-specific configurations
            │  UBUNTU     │
            └─────────────┘
```

## 🔄 Handler Notification Chain

```
Task Changes ──► notify ──► Handler Execution

SSH Template Change    ──notify──► restart_sshd
Apache Config Change   ──notify──► restart_apache
```

## 📊 File Deployment Map

```
SOURCE FILES                    DESTINATION
├── files/
│   ├── sudoer_bhedil      ──►  /etc/sudoers.d/bhedil
│   └── default_site.html  ──►  /var/www/html/index.html
│
└── templates/
    ├── sshd_config_ubuntu.j2  ──►  /etc/ssh/sshd_config (Ubuntu)
    └── sshd_config_redhat.j2  ──►  /etc/ssh/sshd_config (RedHat)
```

This visual representation shows how your Ansible codebase orchestrates multi-OS server management with clear variable dependencies and role interactions.
