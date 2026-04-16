# Graylog Ansible Role

Automates the full Graylog 6.1 stack on Ubuntu 22.04 / 24.04.

Installs and configures:
- **MongoDB 7.0** (with optional Replica Set for HA)
- **OpenSearch 2.x** (Graylog 6.x requires OpenSearch, not Elasticsearch)
- **Java 17** (OpenJDK headless)
- **Graylog Server 6.1**

---

## Multi-node Cluster Recommendation

| Scale | Nodes | Layout |
|---|---|---|
| **Dev / Test** | 1 node | All services on one VM |
| **Minimum HA** | **3 nodes** | Graylog + MongoDB (replica set) + OpenSearch co-located |
| **Production** | **6 nodes** | 3 dedicated Graylog+MongoDB nodes + 3 dedicated OpenSearch nodes |

> **Why 3?** MongoDB requires an **odd number** of replica set members for quorum-based elections. Three nodes give you 1-node fault tolerance. Two nodes offer no fault tolerance.

### 3-Node Architecture (recommended starting point)

```
┌─────────────────────────────────────────────────────────┐
│                    Load Balancer (nginx/HAProxy)          │
│                    :9000  ──►  Graylog nodes              │
└───────────────┬─────────────┬─────────────┬─────────────┘
                │             │             │
        ┌───────┴──────┐ ┌───┴──────┐ ┌───┴──────┐
        │  graylog01   │ │graylog02 │ │graylog03 │
        │  (LEADER)    │ │(worker)  │ │(worker)  │
        │  MongoDB P   │ │MongoDB S │ │MongoDB S │
        │  OpenSearch M│ │OpenSearch│ │OpenSearch│
        └──────────────┘ └──────────┘ └──────────┘
```
- **MongoDB** forms a 3-member replica set (`rs0`) — 1 primary + 2 secondaries
- **OpenSearch** forms a 3-node cluster — 1 elected master + 2 data nodes
- **Graylog** — 1 leader + 2 workers (leader flag auto-set via `inventory_hostname`)

---

## Quick Start

### 1. Generate secrets
```bash
# password_secret
pwgen -s 96 1

# root_password_sha2
echo -n 'yourpassword' | sha256sum | cut -d' ' -f1
```

### 2. Fill in group_vars/all.yml
```yaml
graylog_password_secret:    "< output of pwgen >"
graylog_root_password_sha2: "< output of sha256sum >"
```

### 3. Update inventory/hosts.ini
Replace the placeholder IPs with your actual server IPs.

### 4. Run the playbook
```bash
# Full cluster deploy
ansible-playbook site.yml

# Only reconfigure Graylog (skip install)
ansible-playbook site.yml --tags configure

# Only a specific node
ansible-playbook site.yml --limit graylog01
```

---

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `graylog_version` | `6.1` | Graylog major.minor version |
| `graylog_password_secret` | `""` | **Required.** Random 96-char secret |
| `graylog_root_password_sha2` | `""` | **Required.** SHA-256 of admin password |
| `graylog_root_username` | `admin` | Admin username |
| `graylog_is_leader` | `false` | Set `true` on exactly ONE node |
| `graylog_http_bind_address` | `127.0.0.1:9000` | API/UI bind address |
| `graylog_mongodb_uri` | `mongodb://127.0.0.1:27017/graylog` | MongoDB connection URI |
| `graylog_opensearch_hosts` | `http://127.0.0.1:9200` | OpenSearch URI(s) |
| `mongodb_version` | `7.0` | MongoDB version |
| `mongodb_replicaset_name` | `rs0` | Replica set name |
| `opensearch_heap_size` | `2g` | JVM heap (~50% of RAM, max 32g) |
| `opensearch_disable_security` | `true` | Disable OpenSearch security plugin |
| `java_package` | `openjdk-17-jre-headless` | Java package name |

See [defaults/main.yml](defaults/main.yml) for the full list.

---

## Directory Structure
```
roles/graylog/
├── defaults/main.yml               # All overridable variables
├── vars/main.yml                   # Internal constants (URLs, paths)
├── meta/main.yml                   # Galaxy metadata
├── handlers/main.yml               # Service restart handlers
├── tasks/
│   ├── main.yml                    # Entry-point, includes all sub-tasks
│   ├── install_dependencies.yml    # apt utils (curl, gnupg2, pwgen…)
│   ├── install_mongodb.yml         # MongoDB 7.0 + service
│   ├── install_java.yml            # OpenJDK
│   ├── install_opensearch.yml      # OpenSearch 2.x + service
│   ├── install_graylog.yml         # Graylog package
│   ├── configure_graylog.yml       # server.conf + start service
│   └── configure_mongodb_replicaset.yml  # rs0 bootstrap (primary only)
└── templates/
    ├── server.conf.j2              # Graylog server config
    ├── opensearch.yml.j2           # OpenSearch cluster config
    └── mongod.conf.j2              # MongoDB config with replication
```

---

## Reference
- [Graylog official Ansible role](https://github.com/Graylog2/graylog-ansible-role)
- [Graylog multi-node docs](https://docs.graylog.org/v1/docs/multinode-setup)
- [Graylog compatibility matrix](https://go2docs.graylog.org/current/downloading_and_installing_graylog/compatibility_matrix.htm)
