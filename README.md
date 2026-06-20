# Redis Cluster Lifecycle Management Tool

This project provides a CLI tool (`redis-tool`) that wraps Ansible to provision, operate, verify, and perform a zero-downtime rolling upgrade of a 6-node Redis Cluster running inside containers (Docker or Podman).

## Project Structure

```
redis-cluster-lifecycle/
├── redis-tool                  # Python CLI entrypoint
├── ansible/
│   ├── ansible.cfg             # Ansible defaults
│   ├── inventory/
│   │   └── hosts.ini           # Inventory listing the 6 containers
│   ├── playbooks/
│   │   ├── provision.yml       # Playbook to install Redis & form cluster
│   │   ├── upgrade.yml         # Playbook to upgrade a single node
│   │   └── status.yml          # Playbook to fetch node status
│   └── roles/
│       └── redis/
│           └── tasks/
│               └── main.yml    # Build-from-source & service config tasks
├── infra/
│   ├── Containerfile           # Ubuntu base image with systemd & SSH
│   └── compose.yml             # 6-node network infrastructure
└── README.md                   # This instruction file
```

---

## Prerequisites & Infrastructure Setup

### Automated Setup (Recommended)
You can let `redis-tool` automatically install missing dependencies (like Podman or Ansible) on the host machine. Simply add the `--auto-install` flag to your command. The tool will check what is missing, ask for your confirmation, and run the installation commands:
```bash
./redis-tool --auto-install provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

### Manual Setup
If you prefer to install dependencies yourself:

#### Option A: macOS
1. **Container Runtime**: Install [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-os/) (Recommended) or install Podman via Homebrew:
   ```bash
   brew install podman podman-compose
   podman machine init && podman machine start
   ```
2. **Ansible**: Install via Homebrew:
   ```bash
   brew install ansible
   ```

#### Option B: Linux (Ubuntu/Debian)
1. **Container Runtime**: Install Docker Engine via the [official Docker repo](https://docs.docker.com/engine/install/ubuntu/) or install Podman via apt:
   ```bash
   sudo apt-get update
   sudo apt-get install -y podman podman-compose
   ```
2. **Ansible**: Install via the official PPA or Python pip:
   ```bash
   sudo apt-get update && sudo apt-get install -y software-properties-common
   sudo add-apt-repository --yes --update ppa:ansible/ansible
   sudo apt-get install -y ansible
   ```

After installing dependencies, make the CLI entrypoint executable:
```bash
chmod +x redis-tool
```

---

### Optional: Manual Keygen & Container Setup
The `provision` command automatically generates the SSH key pair and starts the Docker Compose stack for you. However, if you prefer to set up your environment manually before running the tool, you can do so by running:

1. **Manually Generate SSH Keys**:
   ```bash
   ssh-keygen -t rsa -b 2048 -f infra/redis_cluster_key -N ''
   chmod 600 infra/redis_cluster_key
   ```
2. **Manually Spin Up Containers**:
   ```bash
   # If using Docker:
   docker compose -f infra/compose.yml up -d
   
   # If using Podman:
   podman-compose -f infra/compose.yml up -d
   ```

*Note: If you pre-create the keys and start the compose stack manually, `./redis-tool provision` will automatically detect them and proceed directly to running the Ansible playbook without regenerating keys or recreating containers.*

---

## How to Run Each Command

### 1. Provision the Cluster (Phase 1)
To download, compile, and install Redis `7.0.15` from source across all 6 nodes, and form a cluster (3 masters + 3 replicas):
```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

### 2. Seed Data (Phase 2)
To seed 1000 key-value pairs (generating deterministic checksums like `key:0001` -> `sha256(key:0001)`) across all masters:
```bash
./redis-tool data seed --keys 1000
```

### 3. Verify Data (Phase 2)
To read back all 1000 keys from the cluster, recompute the expected checksums, and check for mismatches or missing keys:
```bash
./redis-tool data verify
```

### 4. Cluster Status (Phase 3)
To print a clear, human-readable summary of the cluster showing node IPs, ports, master/replica roles, slots, keys, and memory usage:
```bash
./redis-tool status
```

### 5. Rolling Upgrade (Phase 4 - Zero Downtime)
To upgrade the cluster to a target version (e.g. `7.2.6`) without losing cluster availability or data integrity:
```bash
./redis-tool upgrade --target-version 7.2.6
```

### 6. Comprehensive Health Checks (Phase 5)
To execute a thorough post-upgrade sanity check across five axes (data integrity, version consistency, topology health, cluster state, and replication lag):
```bash
./redis-tool verify --full
```

---

## Rolling Upgrade Strategy

To achieve **zero-downtime** rolling upgrades, the tool enforces the following strategy:

1. **Pre-flight Checks**: Verify that the cluster is healthy (`cluster_state:ok`), all nodes are reachable, and the data is fully verified before starting.
2. **Upgrade Replicas First (One at a time)**:
   * Replicas are safe to take down because they do not serve write requests.
   * Stop Redis on the replica, compile/install the new version, restart it, and **poll** until it re-syncs with its master (`master_link_status:up`) before moving to the next replica.
3. **Upgrade Masters (One at a time via graceful failover)**:
   * You cannot stop a master directly without causing slot errors and downtime.
   * Find the replica of the target master, connect to that replica, and trigger `CLUSTER FAILOVER`.
   * The replica gracefully takes over the master's slots. The old master gets demoted to a replica.
   * Wait for the failover swap to complete in the topology (`CLUSTER NODES`).
   * The old master (now a replica) is safe to stop, upgrade, and restart.
   * Wait for the upgraded node to sync back with its new master.
4. **Post-upgrade verification**: Verify all 1000 keys and make sure all nodes report the new version.

---

## Idempotency (Stretch Goal S4)
* **Provision**: If the target version of Redis is already installed on a container, the Ansible task skips compilation and downloads, ensuring faster execution and preventing service interruptions.
* **Upgrade**: If the pre-flight check detects that all nodes are already running the target version, the tool exits cleanly.

## Structured Logging (Stretch Goal S5)
Every command writes detailed JSON-structured logs to the `logs/redis-tool.log` file. Each log contains:
* Timestamps (`timestamp`)
* The command (`command`)
* Specific actions taken (`action`)
* Outcome status (`SUCCESS`, `FAILED`, `INFO`)
* Associated node name (`node`)
* Error or progress details (`message`)

---

## Assumptions & Trade-offs

1. **Direct SSH Daemon**: We run `/usr/sbin/sshd -D` directly in our containers instead of running a systemd init daemon. This bypasses WSL2 cgroup and systemd compatibility issues, ensuring that the SSH ports become reachable immediately.
2. **Compilation from Source**: To guarantee precise versions (`7.0.15` and `7.2.6`) without relying on potentially missing or modified third-party APT repositories, we compile Redis from source. While compilation takes a minute or two on initial provision, it is highly reliable.
3. **Direct Container Exec**: Seeding, status checks, and failovers are driven by piping commands to `redis-cli` via container execution (`docker exec` or `podman exec`). This avoids the need to expose ports `6379-6384` to the host machine or install Redis clients on the host.
