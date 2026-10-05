# Infrastructure as Code (IaC) & Automation
### Terraform & Ansible Production Deployment

---

## 1. Terraform State, Lifecycle & Best Practices

```text
[Configuration .tf] -> [terraform plan] (Execution Graph) -> [terraform apply] -> [Remote State S3 + DynamoDB Lock]
```

* **Remote State Storage:** Store `terraform.tfstate` in an S3 Bucket with Server-Side Encryption (`SSE-KMS`) and Object Versioning.
* **State Locking:** Use DynamoDB table with `LockID` string attribute to prevent concurrent execution conflicts.
* **Module Reusability:** Parameterize VPC, Compute, and Security Group modules across environments (`environments/prod`, `environments/staging`).

---

## 2. Ansible Playbook: Server Hardening & Configuration

```yaml
---
- name: Enterprise Linux Hardening and Telemetry Agent Setup
  hosts: all
  become: true
  tasks:
    - name: Update all apt/dnf packages to latest security baseline
      package:
        name: "*"
        state: latest

    - name: Ensure SSH Root Login and Password Auth are Disabled
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regex }}"
        line: "{{ item.line }}"
        state: present
      loop:
        - { regex: '^#?PermitRootLogin', line: 'PermitRootLogin no' }
        - { regex: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
      notify: Restart SSH

    - name: Install and Enable Auditd for Host Telemetry
      package:
        name:
          - auditd
          - jq
          - curl
        state: present

    - name: Enable and Start Auditd Service
      service:
        name: auditd
        state: started
        enabled: true

  handlers:
    - name: Restart SSH
      service:
        name: sshd
        state: restarted
```
