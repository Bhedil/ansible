# Ansible Variables - Simple Breakdown

## 🔍 What Are Variables in This Project?

Think of variables as **placeholders** that change based on which server you're working with. Like having different recipes for different kitchens!

## 📋 All Variables Used

### 1. **apache_package** 
- **What it does**: Tells Ansible which Apache software to install
- **Why it's different**: Ubuntu calls it `apache2`, RedHat calls it `httpd`
- **Where it's used**: Installing Apache web server
- **Values**:
  - Ubuntu server (172.31.14.83): `apache2`
  - RedHat server (bhedil@172.31.6.78): `httpd`

### 2. **apache_service**
- **What it does**: Tells Ansible which service name to start/stop
- **Why it's different**: Same reason as package - different OS, different names
- **Where it's used**: Starting Apache, restarting Apache
- **Values**:
  - Ubuntu server: `apache2`
  - RedHat server: `httpd`

### 3. **ssh_template_file**
- **What it does**: Chooses which SSH security template to use
- **Why it's different**: Ubuntu and RedHat have different SSH config formats
- **Where it's used**: Configuring SSH security settings
- **Values**:
  - Ubuntu server: `sshd_config_ubuntu.j2`
  - RedHat server: `sshd_config_redhat.j2`

### 4. **passwd_auth**
- **What it does**: Controls password login (security setting)
- **Why it's the same**: Both servers use key-only authentication
- **Where it's used**: SSH security configuration
- **Values**: `no` (disable password login for security)

### 5. **ansible_distribution** (Built-in Variable)
- **What it does**: Automatically detects what type of Linux is running
- **Why it's important**: Used to make decisions (if Ubuntu do this, if RedHat do that)
- **Where it's used**: Throughout playbooks for conditional tasks
- **Values**: `Ubuntu` or `RedHat`

## 🔄 How Variables Work Together

### Simple Example Flow:

1. **Ansible looks at server**: "What server am I working on?"
   - Answer: `172.31.14.83`

2. **Loads variables for that server**: "What variables does this server use?"
   - Answer: `apache_package: apache2`, `apache_service: apache2`

3. **Runs tasks using those variables**: "Install Apache"
   - Command becomes: Install `apache2` package (not httpd)

### Real Task Example:
```yaml
# Original task (template):
- name: install apache
  package:
    name: "{{ apache_package }}"

# What it becomes for Ubuntu server:
- name: install apache  
  package:
    name: apache2

# What it becomes for RedHat server:
- name: install apache
  package: 
    name: httpd
```

## 🎯 Variable Sources & Priority

### Where Variables Come From (in order of importance):

1. **host_vars files** (highest priority)
   - `host_vars/172.31.14.83.yml` → Ubuntu server variables
   - `host_vars/bhedil@172.31.6.78.yml` → RedHat server variables

2. **Built-in variables** (automatic)
   - `ansible_distribution` → Detected automatically
   - `ansible_hostname` → Server hostname

3. **Default values** (lowest priority)
   - Set in role defaults (not used in this project)

## 📊 Variable Usage Map

| Variable | File Location | Used In | Purpose |
|----------|---------------|---------|---------|
| `apache_package` | host_vars/*.yml | web_servers/tasks | Install correct Apache |
| `apache_service` | host_vars/*.yml | web_servers/tasks & handlers | Manage Apache service |
| `ssh_template_file` | host_vars/*.yml | base/tasks | Configure SSH per OS |
| `passwd_auth` | host_vars/*.yml | SSH templates | Security setting |
| `ansible_distribution` | Auto-detected | Multiple playbooks | OS detection |

## 🔍 Variable Dependencies

### Chain Reactions:
```
Server IP → host_vars file → variable values → task execution → result

Example:
172.31.14.83 → 172.31.14.83.yml → apache_package: apache2 → install apache2 → Ubuntu Apache running
```

## 💡 Simple Rules

1. **Each server has its own variable file** in `host_vars/`
2. **Variables have different values per server** because different OS need different commands
3. **Tasks use `{{ variable_name }}` to insert the actual value**
4. **Ansible picks the right variables automatically** based on which server it's working on

## 🚨 What Happens If Variables Are Wrong?

### Common Problems:
- **Wrong package name**: `apache_package: wrong_name` → Installation fails
- **Wrong service name**: `apache_service: wrong_service` → Can't start/restart Apache  
- **Wrong template**: `ssh_template_file: wrong.j2` → SSH config breaks
- **Missing variables**: No host_vars file → Tasks fail with "variable not found"

## 🔧 How to Add New Variables

### Steps:
1. **Decide what you need**: What's different between servers?
2. **Add to host_vars files**: Put variable in each server's file
3. **Use in tasks**: Reference with `{{ variable_name }}`
4. **Test**: Make sure it works on all servers

### Example - Adding a new web directory variable:
```yaml
# In host_vars/172.31.14.83.yml (Ubuntu)
web_directory: /var/www/html

# In host_vars/bhedil@172.31.6.78.yml (RedHat)  
web_directory: /var/www/html

# In task:
- name: create web directory
  file:
    path: "{{ web_directory }}"
    state: directory
```

This system lets you manage different types of servers with the same playbooks - Ansible figures out what to do based on the variables!
