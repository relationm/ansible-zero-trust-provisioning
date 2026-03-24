# 🛡️ Ansible Zero Trust Infrastructure Provisioning (Pet Project)

This project demonstrates the automated configuration of a Linux server using **Ansible** following **Zero Trust** and **Principle of Least Privilege (PoLP)** best practices. To simulate a real-world environment locally, Docker was used to create both the Ansible Control Node and the Target Managed Node.

## 🎯 Project Objectives
1. Automate the provisioning of an Nginx web server using Infrastructure as Code (IaC).
2. Implement strict Identity and Access Management (IAM) and SSH Hardening.
3. Handle Docker-specific limitations (like the absence of `systemd`) gracefully using Ansible Handlers and raw shell modules.
4. Ensure **Idempotency** (the playbook can run multiple times without breaking the system).

---

## 🏗️ Architecture & Commands Used

To run this project, two Docker containers were used to simulate separate machines.

**1. Setting up the Ansible Control Node (Headquarters):**
```bash
docker run -it --name ansible-hq -v "${PWD}:/work" -w /work ubuntu:22.04 bash
apt update && apt install ansible nano ssh -y
ssh-keygen -t rsa -b 4096 -N "" -f /root/.ssh/id_rsa
```
2. Setting up the Target Server (Managed Node):
```Bash
docker run -d --name target-server -p 8080:80 ubuntu:22.04 sh -c "apt update && apt install -y openssh-server python3 && mkdir -p /var/run/sshd && mkdir -p /root/.ssh && echo '<YOUR_PUBLIC_KEY>' > /root/.ssh/authorized_keys && /usr/sbin/sshd -D"
```
3. Executing the Ansible Playbook:
```Bash
# Verify connection
ansible all -i hosts.ini -m ping -u root
# Run the playbook
ansible-playbook -i hosts.ini webserver.yml -u root
```
## 🔒 Security Best Practices Implemented (webserver.yml)
The Ansible playbook is divided into distinct phases, heavily focusing on security before deploying the application.
Phase 1: Identity & Access Management (PoLP)
* Dedicated Admin User: Created a non-root user (devops_admin) to perform daily tasks.
* Passwordless Sudo: Configured /etc/sudoers safely using visudo to allow devops_admin to escalate privileges without passwords, leaving an audit trail.
* Key-Based Auth: Injected the Control Node's public SSH key directly into the new user's authorized_keys.
Phase 2: SSH Hardening (Zero Trust)
* PermitRootLogin no: Completely disabled direct root access via SSH. If an attacker guesses the root password, they still cannot log in.
* PasswordAuthentication no: Disabled password-based logins entirely. Only cryptographic SSH keys are accepted.
* Session Timeouts: Added TMOUT=300 to /etc/profile to automatically drop inactive SSH sessions after 5 minutes.
Phase 3: Idempotency & Handlers
Used Ansible Handlers (notify: Reload SSH) to softly reload the SSH daemon only when configuration changes occur, preventing the accidental termination of the Docker container's main process (PID 1).
## ⚠️ Docker Environment Limitations (Reality Check)
While this project implements robust application-level security, certain enterprise-grade Zero Trust practices were excluded. This is a deliberate choice due to the architectural limitations of running a target node inside a standard (non-privileged) Docker container:
1. Kernel-Level Protection & LUKS: Containers share the host's Linux kernel. Disk encryption (LUKS at rest) and kernel hardening parameters (sysctl) cannot be modified independently inside a standard container.
2. Micro-segmentation (UFW/Firewalld): Network filtering tools like iptables or ufw require elevated network privileges (--cap-add=NET_ADMIN). In a real VM or Bare Metal server, strict firewall rules (Deny All by default) would be applied.
3. Auditd (System Auditing): The Linux Auditing System interacts directly with the kernel to log file access (e.g., monitoring /etc/shadow). This is unsupported in unprivileged containers.  
  
These practices are fully acknowledged and would be integrated via Ansible if deploying to AWS EC2, VMware VMs, or Bare Metal infrastructure.
