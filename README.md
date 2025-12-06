# Ansible Nginx Web Server Infrastructure Automation

## 📋 Project Overview

This is a production-ready **Infrastructure as Code (IaC)** project demonstrating enterprise-level Ansible automation for deploying and configuring **nginx web servers** across a distributed infrastructure. The project showcases proper separation of concerns, idempotent operations, and Ansible best practices using role-based architecture.

**Key Learning Objectives Achieved:**
- ✅ Ansible playbook design and execution
- ✅ Role-based configuration management
- ✅ Jinja2 templating for dynamic configurations
- ✅ Inventory management and host grouping
- ✅ Firewall automation with UFW
- ✅ Service lifecycle management
- ✅ Handler-based event-driven actions
- ✅ Idempotence and convergence principles

---

## 🏗️ Architecture & Project Structure

```
ansible-first/
├── site.yaml                    # Main playbook entry point
├── inventory.ini                # Static inventory (hosts & groups)
├── README.md                    # This file
├── files/
│   └── index.html               # Static HTML files (for manual deployments)
└── roles/
    └── webserver/               # Nginx webserver role
        ├── tasks/
        │   └── main.yaml        # Task definitions & automation logic
        ├── handlers/
        │   └── main.yaml        # Event-driven handlers
        ├── templates/
        │   ├── nginx-default.conf.j2    # Jinja2 nginx configuration
        │   └── index.html.j2            # Dynamic HTML template
        └── vars/
            └── main.yaml        # Role-specific variables
```

### Architecture Rationale

**Role-Based Structure:** The project uses Ansible's role system, which provides:
- **Modularity**: The `webserver` role is self-contained and reusable
- **Maintainability**: Clear separation between tasks, handlers, templates, and variables
- **Scalability**: Easy to extend with additional roles or duplicate for multi-tier deployments
- **Best Practice Compliance**: Follows Ansible community standards

---

## 🔧 Component Breakdown

### 1. **Main Playbook (`site.yaml`)**

```yaml
---
- name: Configure webservers with nginx
  hosts: webservers
  become: yes
  
  roles:
    - webserver
```

**Deep Dive:**
- **`hosts: webservers`**: Targets all hosts in the `webservers` group (defined in `inventory.ini`)
- **`become: yes`**: Escalates privileges to `root` for all tasks (required for package installation, service management, firewall rules)
- **`roles: [webserver]`**: Imports the webserver role, executing all tasks, handlers, and applying variables defined within

---

### 2. **Inventory Management (`inventory.ini`)**

```ini
[webservers]  # Host group definition
192.168.29.44 ansible_user=ansible-server
```

**Understanding Inventory:**
- **Host Group `[webservers]`**: Logical grouping allows targeting multiple hosts with a single command
- **IP Address `192.168.29.44`**: Target machine IP (must be reachable from control node)
- **`ansible_user=ansible-server`**: SSH authentication user (must have sudo access for `become: yes` operations)
- **SSH Prerequisite**: Ensure SSH key-based authentication is configured:
  ```bash
  ssh-copy-id -i ~/.ssh/id_rsa.pub ansible-server@192.168.29.44
  ```

---

### 3. **Task Orchestration (`roles/webserver/tasks/main.yaml`)**

The tasks are executed sequentially and demonstrate idempotent automation:

#### **A. Nginx Installation**
```yaml
- name: install nginx
  apt:
    name: nginx
    state: present
    update_cache: yes
```
- **Idempotent**: If nginx is already installed, this task skips
- **`update_cache: yes`**: Refreshes APT package cache before installation
- **State Management**: `present` ensures the package exists; does nothing if already installed

#### **B. Deploy User Creation**
```yaml
- name: create deploy user
  user:
    name: deploy
    groups: sudo
    shell: /bin/bash
    state: present
    create_home: yes
```
- **Purpose**: Creates a dedicated deployment user for application deployments
- **`groups: sudo`**: Grants sudo privileges without password (requires sudoers configuration)
- **`create_home: yes`**: Creates `/home/deploy` directory for SSH keys and configs

#### **C. Firewall Installation & Configuration**
```yaml
- name: install ufw firewall
  apt:
    name: ufw
    state: present

- name: allow ssh through the firewall
  ufw:
    rule: allow
    port: "22"
    proto: tcp

- name: allow http through the firewall
  ufw:
    rule: allow
    port: "80"
    proto: tcp

- name: enable the firewall
  ufw:
    state: enabled
    policy: deny
```

**Security Best Practices:**
- **UFW (Uncomplicated Firewall)**: Layer 4 protection for the system
- **Explicit Allow Rules**: SSH (22) and HTTP (80) are explicitly allowed
- **Implicit Deny Policy**: All other inbound traffic is dropped by default
- **Statefulness**: Rules persist across reboots (`enabled: yes`)

#### **D. Nginx Configuration Deployment**
```yaml
- name: deploy nginx default config
  template:
    src: nginx-default.conf.j2
    dest: /etc/nginx/sites-available/default
    owner: root
    group: root
    mode: "0644"
  notify: reload nginx
```

**Template Module Details:**
- **`template` Module**: Processes Jinja2 templates and deploys configuration
- **`notify: reload nginx`**: Triggers the `reload nginx` handler if the file changes
- **Handler Deduplication**: Multiple tasks can notify the same handler; it runs only once at playbook end
- **File Permissions**: `mode: 0644` = `-rw-r--r--` (readable by all, writable by root only)

#### **E. Service Lifecycle Management**
```yaml
- name: make sure nginx is up and running
  service:
    name: nginx
    state: started
    enabled: yes

- name: deploy templated index.html
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
    owner: www-data
    group: www-data
    mode: "0644"
```

**Service Module:**
- **`state: started`**: Ensures the service is currently running
- **`enabled: yes`**: Registers nginx to start on system boot
- **Idempotence**: If already running, nothing happens

---

### 4. **Handlers (`roles/webserver/handlers/main.yaml`)**

```yaml
---
- name: reload nginx
  service:
    name: nginx
    state: reloaded
```

**Handler Pattern Explanation:**
- **Event-Driven Execution**: Handlers only run when notified by a task
- **`state: reloaded`**: Gracefully reloads nginx configuration without dropping connections
- **Convergence**: All handlers execute once at the end of the playbook, in the order defined
- **Benefits**: Avoids unnecessary service restarts; efficient when multiple tasks modify configuration

---

### 5. **Variables (`roles/webserver/vars/main.yaml`)**

```yaml
---
site_title: "Ansible demo site"
heading: "Well hello there!"
body_text: "Jinja 2 template rendered by ansible"
```

**Variable Management:**
- **Role-Scoped**: These variables are specific to the webserver role
- **Jinja2 Interpolation**: Available in templates as `{{ site_title }}`, `{{ heading }}`, `{{ body_text }}`
- **Data Structure**: Can use dictionaries, lists, and nested structures for complex configurations
- **Variable Precedence**: Role vars > defaults > inventory vars (following Ansible's precedence rules)

---

### 6. **Jinja2 Templates**

#### **Nginx Configuration (`roles/webserver/templates/nginx-default.conf.j2`)**
```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    
    server_name _;
    root /var/www/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

**Configuration Analysis:**
- **IPv4 & IPv6 Support**: Listens on both `80` (IPv4) and `[::]:80` (IPv6)
- **Catch-All Server**: `server_name _;` matches any hostname (default behavior)
- **Document Root**: `/var/www/html` serves static files
- **Efficient Routing**: `try_files` attempts to serve static files, returns 404 if not found
- **No Dynamic Processing**: Pure static content delivery (HTTP 200 or 404 only)

#### **HTML Template (`roles/webserver/templates/index.html.j2`)**
Uses Jinja2 variables for dynamic content:
```jinja2
{{ site_title }}
{{ heading }}
{{ body_text }}
```

**Templating Power:**
- **Dynamic Content**: Variables rendered at deployment time
- **Reusability**: Same template deployed to multiple servers with different values
- **Version Control**: Template source is tracked; generated content is reproducible

---

## 🚀 Execution Guide

### Prerequisites
```bash
# 1. Ansible installation (on control node)
sudo apt install ansible

# 2. Python 3 on target hosts
ssh ansible-server@192.168.29.44 "sudo apt install python3-minimal"

# 3. SSH key configuration
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_rsa
ssh-copy-id -i ~/.ssh/id_rsa.pub ansible-server@192.168.29.44

# 4. Sudo access without password (on target)
ssh ansible-server@192.168.29.44 'echo "ansible-server ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansible'
```

### Running the Playbook

**Syntax validation:**
```bash
ansible-playbook site.yaml --syntax-check
```

**Dry-run (check mode):**
```bash
ansible-playbook site.yaml --check
```
*Shows what would change without making modifications*

**Execute with verbose output:**
```bash
ansible-playbook site.yaml -v
```

**Execute all tasks:**
```bash
ansible-playbook site.yaml
```

**Run with extra verbosity for debugging:**
```bash
ansible-playbook site.yaml -vvv
```

**Run specific tasks only:**
```bash
ansible-playbook site.yaml --tags "nginx"
# Note: Add `tags: ["nginx"]` to tasks to enable this filtering
```

---

## 🔐 Security Considerations

| Aspect | Implementation |
|--------|-----------------|
| **Privilege Escalation** | Uses `become: yes` with passwordless sudo via `/etc/sudoers.d/` |
| **Firewall** | UFW with explicit allow rules; implicit deny default policy |
| **SSH Access** | Key-based authentication only; SSH port (22) explicitly allowed |
| **File Ownership** | Web files owned by `www-data:www-data` (nginx's default user) |
| **File Permissions** | 0644 for configs (readable globally, writable by root only) |
| **Dedicated Deploy User** | Non-root `deploy` user with sudo access for deployments |

**Future Hardening:**
```yaml
- name: disable password authentication
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    state: present
  notify: restart sshd
```

---

## 📊 Idempotence & Convergence

This playbook is **idempotent**—running it multiple times produces the same result:

| Scenario | Behavior |
|----------|----------|
| First run | All tasks execute; services installed, configured, started |
| Second run | Tasks verify state; nginx already installed, config unchanged, service already running |
| After manual change to nginx config | Task re-detects manual change; handler triggers reload |
| Config reverted to state in playbook | Automatic correction on next run |

**Convergence:** The infrastructure automatically converges to the desired state defined in the playbook, regardless of current state.

---

## 🔄 Extending the Project

### Adding HTTPS Support
```yaml
- name: install certbot
  apt:
    name: certbot python3-certbot-nginx
    state: present

- name: obtain ssl certificate
  shell: certbot certonly --standalone -d example.com -n --agree-tos -m admin@example.com
  args:
    creates: /etc/letsencrypt/live/example.com/fullchain.pem
```

### Adding Additional Hosts
```ini
[webservers]
192.168.29.44 ansible_user=ansible-server hostname=web1
192.168.29.45 ansible_user=ansible-server hostname=web2
```

### Environment-Specific Variables
```yaml
# group_vars/webservers/main.yaml
---
nginx_worker_processes: "{{ ansible_processor_vcpus }}"
nginx_worker_connections: 1024
```

---

## 📚 Ansible Concepts Demonstrated

- **Playbooks**: Orchestration of multiple tasks in declarative YAML
- **Roles**: Modular, reusable automation units with clear structure
- **Handlers**: Event-driven actions that run once at playbook completion
- **Jinja2 Templating**: Dynamic configuration file generation
- **Idempotence**: Safe, repeatable automation
- **Become/Privilege Escalation**: Sudo-based root access management
- **Modules**: `apt`, `user`, `ufw`, `template`, `service` (specialized task types)
- **Inventory**: Host and group management with variable assignment

---

## 🧪 Validation & Troubleshooting

### Check connectivity
```bash
ansible all -i inventory.ini -m ping
```

### Inspect gathered facts
```bash
ansible all -i inventory.ini -m setup | head -50
```

### Debug variable resolution
```bash
ansible-playbook site.yaml -e "ansible_verbosity=4"
```

### SSH connection issues
```bash
ssh -vvv ansible-server@192.168.29.44
```

---

## 📖 References & Learning Resources

- [Ansible Official Documentation](https://docs.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_and_tricks.html)
- [Jinja2 Template Engine](https://jinja.palletsprojects.com/)
- [UFW Firewall Guide](https://wiki.ubuntu.com/UncomplicatedFirewall)
- [Nginx Configuration Pitfalls](https://wiki.nginx.org/Pitfalls)

---

## 📝 License

This project is for educational purposes demonstrating infrastructure automation best practices.

**Author:** Learning Infrastructure Automation with Ansible
**Date:** December 2025
**Branch:** restruct-roles (demonstrating role-based restructuring)