# graylog-single-node

Ansible role that deploys a full **Graylog 7.0 LTS** single-node stack on **Ubuntu 24.04 (noble)**.

## Stack

| Component | Version | Notes |
|---|---|---|
| **Graylog Server** | 7.0 (LTS) | Supported until Nov 2027 |
| **graylog-datanode** | 7.0 | Manages embedded OpenSearch internally — no standalone OS needed |
| **MongoDB** | 8.0 | Replica set (`rs0`) required by graylog-datanode |
| **OpenSearch** | 2.19 (bundled) | Runs inside graylog-datanode, no separate install |

> **No Java, no standalone OpenSearch install.** Both are bundled inside the `graylog-datanode` package.

---

## Requirements

- Ubuntu 24.04 (noble)
- Root SSH access
- ~5 GB free disk space

---

## Quick Start

### 1. Generate secrets

```bash
# password_secret
openssl rand -hex 32

# root_password_sha2
echo -n 'yourpassword' | sha256sum | cut -d' ' -f1
```

### 2. Fill in group_vars/all.yml

```yaml
graylog_password_secret:    "<output of openssl rand -hex 32>"
graylog_root_password_sha2: "<output of sha256sum>"
graylog_http_publish_uri:   "http://<SERVER_IP>:9000/"
graylog_http_external_uri:  "http://<SERVER_IP>:9000/"
```

### 3. Update inventory/hosts.ini

```ini
[graylog]
graylog01 ansible_host=<SERVER_IP>

[graylog:vars]
ansible_user=root
```

### 4. Run the playbook

```bash
ansible-playbook site.yml
```

The playbook prints the one-time preflight password at the end:

```
============================================
 Graylog is UP — complete the setup wizard
 URL      : http://<SERVER_IP>:9000
 Username : admin
 Password : <one-time preflight password>
 (one-time preflight password — your normal admin password takes over after wizard)
============================================
```

Complete the preflight wizard in the browser, then use your normal `admin` password.

---

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `graylog_version` | `7.0` | Graylog major.minor version |
| `graylog_password_secret` | `""` | **Required.** `openssl rand -hex 32` |
| `graylog_root_password_sha2` | `""` | **Required.** SHA-256 of admin password |
| `graylog_root_username` | `admin` | Admin username |
| `graylog_is_leader` | `true` | Leader node flag |
| `graylog_http_bind_address` | `0.0.0.0:9000` | API/UI bind address |
| `graylog_http_publish_uri` | `http://127.0.0.1:9000/` | Public URI |
| `graylog_mongodb_uri` | `mongodb://127.0.0.1:27017/graylog?replicaSet=rs0` | MongoDB URI |
| `datanode_heap` | `1g` | Heap for embedded OpenSearch (~50% of RAM, max 31g) |
| `mongodb_version` | `8.0` | MongoDB version |
| `mongodb_replicaset_name` | `rs0` | Replica set name |

See [defaults/main.yml](defaults/main.yml) for the full list.

---

## Directory Structure

```
graylog-single-node/
├── defaults/main.yml               # All overridable variables
├── vars/main.yml                   # Internal constants (URLs, paths)
├── meta/main.yml                   # Galaxy metadata
├── handlers/main.yml               # Service restart handlers
├── tasks/
│   ├── main.yml                    # Entry-point
│   ├── install_dependencies.yml    # apt utils
│   ├── install_mongodb.yml         # MongoDB 8.0 + replica set init
│   ├── install_graylog.yml         # graylog-datanode + graylog-server
│   └── configure_graylog.yml       # server.conf + start + print password
└── templates/
    ├── server.conf.j2              # Graylog server config
    ├── datanode.conf.j2            # graylog-datanode config
    └── mongod.conf.j2              # MongoDB config
```

---

## How It Works

1. **MongoDB 8.0** is installed and a single-node replica set (`rs0`) is initiated — required by `graylog-datanode` for its preflight checks.
2. **graylog-datanode** is installed. It bundles OpenSearch 2.x internally and self-registers with graylog-server. Runtime directories (`/datanode/data`, `/datanode/logs`, `/datanode/config`) are created with correct ownership.
3. **graylog-server** is installed and configured. On first boot it enters preflight mode — a one-time password is printed in the logs and by the playbook.
4. After completing the browser wizard, Graylog is fully operational.

---

## References

- [Graylog 7.0 docs](https://go2docs.graylog.org/current/)
- [graylog-datanode configuration reference](https://go2docs.graylog.org/current/setting_up_graylog/data_node_configuration_file.htm)
- [Graylog compatibility matrix](https://go2docs.graylog.org/current/downloading_and_installing_graylog/compatibility_matrix.htm)
