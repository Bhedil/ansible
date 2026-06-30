# Ansible Codebase Mapping & Analysis

## 📋 Overview
This Ansible project manages web servers and database servers with different configurations for Ubuntu and RedHat systems.

## 🏗️ Project Structure

```
ansible-main/
├── ansible.cfg              # Main configuration
├── inventory               # Host inventory
├── site.yml               # Main playbook
├── host_vars/             # Host-specific variables
├── roles/                 # Organized functionality
├── files/                 # Static files
└── *.yml playbooks       # Individual playbooks
```

## 🔧 Configuration Files

### ansible.cfg
**Purpose**: Main Ansible configuration
**Key Settings**:
- `inventory = inventory` → Points to inventory file
- `private_key_file = ~/.ssh/ansiblepoc` → SSH key location
- `remote_user = bhedil` → Default user for connections

### inventory
**Purpose**: Defines target servers and groups
**Server Groups**:
- `[apache]` → Web servers (172.31.14.83, bhedil@172.31.6.78)
- `[db_servers]` → Database servers (bhedil@172.31.6.78)

**Note**: `bhedil@172.31.6.78` serves BOTH roles (web + db server)

## 📊 Variable System

### Global Variables (host_vars/)

#### 172.31.14.83.yml (Ubuntu Server)
```yaml
apache_package: apache2          # Ubuntu Apache package name
apache_service: apache2          # Ubuntu Apache service name
passwd_auth: no                  # Disable password authentication
ssh_template_file: sshd_config_ubuntu.j2  # SSH template for Ubuntu
```

#### bhedil@172.31.6.78.yml (RedHat Server)
```yaml
apache_package: httpd            # RedHat Apache package name
apache_service: httpd            # RedHat Apache service name
passwd_auth: no                  # Disable password authentication
ssh_template_file: sshd_config_redhat.j2  # SSH template for RedHat
```

### Variable Usage Patterns

| Variable | Used In | Purpose |
|----------|---------|---------|
| `apache_package` | web_servers/tasks/main.yml | Install correct Apache package per OS |
| `apache_service` | web_servers/tasks/main.yml, handlers/main.yml | Manage Apache service per OS |
| `ssh_template_file` | base/tasks/main.yml | Configure SSH with OS-specific template |
| `ansible_distribution` | Throughout playbooks | Conditional OS-based execution |

## 🎯 Playbook Flow

### site.yml (Main Orchestrator)
```yaml
1. Play 1: ALL HOSTS
   └── Pre-tasks: Update packages (OS-specific)
       ├── RedHat: Uses dnf
       └── Ubuntu: Uses apt

2. Play 2: ALL HOSTS
   └── Role: base (user management, SSH config)

3. Play 3: DB_SERVERS GROUP
   └── Role: db_servers (MariaDB installation)

4. Play 4: APACHE GROUP
   └── Role: geerlingguy.apache (external role)
```

## 🔄 Role Dependencies & Interactions

### base Role
**Purpose**: Foundation setup for all servers
**Tasks**:
1. Create `bhedil` user with root group access
2. Deploy sudo permissions file
3. Add SSH public key for authentication
4. Configure SSH daemon using OS-specific template

**Files Used**:
- `files/sudoer_bhedil` → Sudo permissions
- `templates/sshd_config_ubuntu.j2` → Ubuntu SSH config
- `templates/sshd_config_redhat.j2` → RedHat SSH config

**Variables Required**:
- `ssh_template_file` (from host_vars)

**Handlers Triggered**:
- `restart_sshd` when SSH config changes

### web_servers Role
**Purpose**: Apache web server setup
**Tasks**:
1. Install Apache package (OS-specific via `apache_package` variable)
2. Start Apache service (OS-specific via `apache_service` variable)
3. Configure ServerAdmin email (RedHat only)
4. Deploy custom HTML page (Ubuntu only)

**Variables Required**:
- `apache_package` (from host_vars)
- `apache_service` (from host_vars)

**Files Used**:
- `files/default_site.html` → Custom web page

**Handlers Triggered**:
- `restart_apache` when configuration changes

### db_servers Role
**Purpose**: Database server setup
**Tasks**:
1. Install MariaDB (RedHat systems only)

**Conditional Logic**:
- Only runs on `ansible_distribution == "RedHat"`

## 🔗 Variable Interaction Map

```
inventory → groups → host_vars → roles
    │         │         │         │
    │         │         │         └── Tasks use variables
    │         │         └── Variables defined per host
    │         └── Groups trigger role assignment  
    └── Hosts belong to groups
```

### Variable Flow Example:
1. **Host**: `172.31.14.83` belongs to `[apache]` group
2. **Variables**: Loaded from `host_vars/172.31.14.83.yml`
3. **Role Assignment**: `apache` group triggers `web_servers` role  
4. **Task Execution**: Uses `apache_package: apache2` for Ubuntu

## 🏷️ Tag System
- `always` → Runs on every execution (user creation, updates)
- `apache` → Apache-related tasks only
- `redhat` → RedHat-specific configurations
- `ubuntu` → Ubuntu-specific configurations

## 🔄 Handler Dependencies

### Handler Trigger Chain:
```
SSH Config Change → notify: restart_sshd → Handler: restart_sshd
Apache Config Change → notify: restart_apache → Handler: restart_apache
```

## 📁 File Dependencies

### Static Files:
- `files/sudoer_bhedil` → Deployed to `/etc/sudoers.d/bhedil`
- `files/default_site.html` → Deployed to `/var/www/html/index.html`

### Template Files:
- `templates/sshd_config_ubuntu.j2` → Deployed to `/etc/ssh/sshd_config`
- `templates/sshd_config_redhat.j2` → Deployed to `/etc/ssh/sshd_config`

## 🔍 Key Insights

### Multi-OS Support Strategy:
1. **Variables**: OS-specific package/service names in host_vars
2. **Conditionals**: `when: ansible_distribution == "RedHat/Ubuntu"`
3. **Templates**: Separate SSH configs per OS

### Server Role Overlap:
- `bhedil@172.31.6.78` serves both web and database roles
- Different servers use different OS (Ubuntu vs RedHat)

### Security Configuration:
- SSH key-based authentication only (`passwd_auth: no`)
- Dedicated sudo user with NOPASSWD access
- SSH daemon hardening via templates

## 🚀 Execution Order

1. **Pre-tasks**: Package updates (OS-specific)
2. **Base Role**: User setup, SSH configuration
3. **Database Role**: MariaDB installation (RedHat db_servers only)
4. **Web Role**: Apache setup (apache group only)

This structure provides a scalable, multi-OS Ansible setup with clear separation of concerns and flexible variable management.
