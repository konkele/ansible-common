# Common Ansible Role

The **`common` role** provides baseline infrastructure configuration for Ubuntu hosts. It is designed to be **applied first** in a playbook, establishing consistent system state and shared metadata for downstream roles.

This role follows a **deterministic, inventory-driven design** intended for long-term maintainability and safe reuse in larger inventories.

---

## Features

* **User Management**

  * Create system and human users with configurable UID/GID
  * Manage SSH authorized keys (exclusive by default)
  * Optional sudo access per user
  * User tagging for classification and automation
  * Explicit, layered user-definition model (global → group → host)

* **Timezone Management**

  * Enforce a consistent system timezone

* **DNS Configuration**

  * Configure DNS search domains (Netplan-aware)

* **Firewall Management**

  * Optional baseline TCP/UDP rules via `geerlingguy.firewall`
  * Soft dependency, enabled by default and safely disabled per host or group
  * Safe extension by downstream roles when enabled

* **Storage Management**

  * NFS client configuration and mounts
  * Local disk partitioning, formatting, and mounting

* **System Updates**

  * Package upgrades with optional serial reboot handling

* **Metadata Persistence**

  * Persist merged user state to `/etc/ansible/common_users.yml`
  * Structured, human-readable YAML suitable for downstream consumption

---

## Firewall Behavior

Firewall management is a **soft dependency**.

By default, the role will include `geerlingguy.firewall`. This behavior can be disabled globally, per group, or per host.

```yaml
# Disable firewall management
firewall_enabled: false
```

When disabled, the role **does not manage firewall state** and makes no assumptions about existing rules or providers.

---

## User Definition Model

This role intentionally **does not allow `common_users` to be defined directly in inventory**.

Ansible does not merge list variables across scopes. Allowing direct definition would result in silent overwrites and unpredictable behavior in multi-group inventories.

Instead, user intent is expressed through **three explicit input variables**, each corresponding to a natural inventory scope:

| Scope  | Variable              | Intended Location        |
| ------ | --------------------- | ------------------------ |
| Global | `common_users_global` | `group_vars/all.yml`     |
| Group  | `common_users_group`  | `group_vars/<group>.yml` |
| Host   | `common_users_host`   | `host_vars/<host>.yml`   |

The role merges these inputs in a fixed order:

```
common_users_global
        ↓
common_users_group
        ↓
common_users_host   (highest precedence)
```

Users are deduplicated by `name`. When the same user appears multiple times, the **definition with the highest precedence wins**.

This approach avoids Ansible list-precedence pitfalls while remaining explicit and predictable.

---

## Inventory Examples

### Global users

```yaml
# group_vars/all.yml
common_users_global:
  - name: homelab
    uid: 1000
    sudo: true
```

### Group-specific users

```yaml
# group_vars/db.yml
common_users_group:
  - name: postgres
    uid: 101
    system: true
```

### Host-specific overrides

```yaml
# host_vars/db01.yml
common_users_host:
  - name: postgres
    uid: 999
```

---

## Persisted User Metadata

After execution, the role writes the final, reconciled user state to:

```
/etc/ansible/common_users.yml
```

This file represents the **authoritative local user model** and is intended to be consumed by downstream roles.

### Example

```yaml
common_users:
  - name: homelab
    uid: 1000
    gid: 1000
    home: /home/homelab
    shell: /bin/bash
    system: false
    sudo: true
    tags:
      - dev
    authorized_keys_exclusive: true
    authorized_keys:
      - "ssh-ed25519 AAAA..."

  - name: postgres
    uid: 999
    gid: 999
    home: /var/lib/postgresql
    shell: /usr/sbin/nologin
    system: true
    sudo: false
    tags: []
    authorized_keys_exclusive: true
    authorized_keys: []
```

---

## Downstream Role Integration

Downstream roles should treat `/etc/ansible/common_users.yml` as a **required dependency**:

```yaml
- name: Verify common user metadata exists
  ansible.builtin.stat:
    path: /etc/ansible/common_users.yml
  register: common_users_file

- name: Load common user metadata
  ansible.builtin.include_vars:
    file: /etc/ansible/common_users.yml
  when: common_users_file.stat.exists

- name: Abort if missing
  ansible.builtin.fail:
    msg: >
      The common role must be applied before this role.
      Missing /etc/ansible/common_users.yml
  when: not common_users_file.stat.exists
```

---

## Example Playbook

```yaml
- name: Baseline configuration
  hosts: all
  become: true
  roles:
    - common
```

---

## Design Principles

* Inventory expresses **intent**, not final state
* The role merges intent explicitly and deterministically
* `common_users` is **generated output**, never inventory input
* Shared state is persisted for auditability and reuse
* Avoids implicit Ansible behaviors that do not scale

---

## Tags

| Tag        | Description                                 |
| ---------- | ------------------------------------------- |
| `users`    | User and SSH configuration                  |
| `dns`      | DNS search domains (Netplan-aware)          |
| `timezone` | Timezone configuration                      |
| `nfs`      | NFS mounts                                  |
| `disks`    | Local disk formatting and mounting          |
| `update`   | System update and serial reboot if required |
