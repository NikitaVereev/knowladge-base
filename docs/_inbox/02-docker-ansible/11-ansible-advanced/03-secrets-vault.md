---
title: 03 Secrets Management & Vault
---

---

## Ansible Vault

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# View encrypted file
ansible-vault view secrets.yml

# Encrypt existing file
ansible-vault encrypt vars.yml

# Decrypt file
ansible-vault decrypt vars.yml

# Change password
ansible-vault rekey secrets.yml
```

---

## Using Vault in Playbooks

```yaml
---
- name: Database setup
  hosts: databases
  vars_files:
    - secrets.yml  # Encrypted file

  tasks:
    - name: Create database user
      postgresql_user:
        name: dbuser
        password: "{{ db_password }}"
        state: present

    - name: Configure application
      template:
        src: app.conf.j2
        dest: /etc/app/config.conf
      vars:
        api_key: "{{ vault_api_key }}"
```

**Running with vault:**
```bash
# Prompt for password
ansible-playbook site.yml --ask-vault-pass

# From environment variable
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook site.yml

# From script
ansible-playbook site.yml --vault-password-file vault_pass.sh

# In CI/CD (GitHub Actions)
ansible-playbook site.yml --vault-password-file <(echo $VAULT_PASS)
```

---

## External Secret Managers

**HashiCorp Vault:**
```yaml
---
- name: Get secrets from HashiCorp Vault
  hosts: all
  
  tasks:
    - name: Get database password
      set_fact:
        db_password: "{{ lookup('hashi_vault', 'secret=secret/data/database/password') }}"

    - name: Get API key
      set_fact:
        api_key: "{{ lookup('hashi_vault', 'secret=secret/data/api/key') }}"

    - name: Configure application
      template:
        src: config.j2
        dest: /etc/app/config
```

**Kubernetes Secrets:**
```yaml
---
- name: Get Kubernetes secrets
  hosts: localhost
  gather_facts: no
  
  tasks:
    - name: Get database password
      set_fact:
        db_password: "{{ lookup('kubernetes.core.k8s', 'kind=Secret', 'name=db-creds', 'namespace=default').data.password | b64decode }}"
```

---

## Vault Best Practices

**Organization:**
```
project/
├── secrets.yml (vault encrypted)
├── .gitignore (secrets.yml ignored)
├── vault_pass.sh (not in git)
└── playbooks/
    └── site.yml
```

**.gitignore:**
```
secrets.yml
vault_pass.sh
.vault_pass
```

**Security:**
- Never commit vault password
- Use strong passwords (32+ chars)
- Rotate passwords regularly
- Audit access logs
- Use external managers for production

---

**Следующее:** [[04-templates-config|Templates & Configuration]]
