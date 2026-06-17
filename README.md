Distributed Redis Cluster Orchestration & Lifecycle Management Engine

A modular, robust, and path-agnostic infrastructure orchestration harness designed to provision, seed, analyze, and perform zero-downtime rolling upgrades on a highly available $6$-node Redis Cluster topology.

Driven by a single CLI utility (redis-tool), this tool integrates standard Ansible orchestration playbooks with python-driven cluster socket orchestration to manage lifecycle configurations across independent container layers natively.

📂 Project Architecture Layout

The directory layout must be structured as follows for relative pathing resolutions to process correctly:

submission/
├── redis-tool                  # Multi-phase Python CLI Orchestration Tool
├── README.md                   # This Comprehensive Documentation File
├── ansible.cfg                 # Global Ansible Configuration File
├── infra/
│   └── redis_cluster_key       # Unshared Private SSH Key for Node Access
└── ansible/
    ├── inventory/
    │   └── hosts.ini           # Node Target Definitions with Local Port Mapping
    └── roles/
        └── redis/
            └── tasks/
                └── main.yml    # Master Ansible Compilation and Setup Plays


1. Bringing Up the Container Infrastructure

The workspace utilizes an isolated network subnet ($10.10.0.0/24$) to bridge internal container endpoints. This transport strategy runs identically on Docker or Podman.

Option A: Using Docker

To spin up the 6 isolated target container nodes with clean network interfaces, run:

docker compose down -v && docker compose up -d


Option B: Using Podman

For daemonless, rootless setups, spin up the nodes using Podman:

podman-compose down -v && podman-compose up -d


Infrastructure Topology

Once initialized, the 6 independent nodes are assigned static IP mappings and bind SSH/Redis interfaces onto host loopbacks as follows:

+--------------------+------------------+---------------+-------------------+------------------+
| Host Container     | Subnet IP        | SSH Host      | Redis Client      | Cluster Bus      |
| Name               | Target           | Port          | Port              | Port             |
+--------------------+------------------+---------------+-------------------+------------------+
| redis-node-1       | 10.10.0.11       | 3311          | 6371              | 16371            |
| redis-node-2       | 10.10.0.12       | 3312          | 6372              | 16372            |
| redis-node-3       | 10.10.0.13       | 3313          | 6373              | 16373            |
| redis-node-4       | 10.10.0.14       | 3314          | 6374              | 16374            |
| redis-node-5       | 10.10.0.15       | 3315          | 6375              | 16375            |
| redis-node-6       | 10.10.0.16       | 3316          | 6376              | 16376            |
+--------------------+------------------+---------------+-------------------+------------------+

2. Running Each redis-tool Command

All lifecycle operations are triggered from the root of the project. Ensure the utility has execution privileges inside your terminal:

chmod +x redis-tool


Command 1: Cluster Provisioning

Builds requested Redis source code from official archives, configures dependencies (gcc, make), deploys customized redis.conf maps, initializes daemons, and automates slot configuration across $16384$ buckets.

./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1


Command 2: Database Data Seeding

Populates the distributed cluster topology using a reproducible hash mapping strategy. It injects a deterministic SHA-256 signature calculated directly from each numerical key signature.

./redis-tool data seed --keys 1000


Command 3: Real-Time Data Integrity Verification

Validates the state of records in memory by pulling seeded keys, recalculating expected cryptographic signatures, and asserting parity.

./redis-tool data verify


Command 4: Topology Status Telemetry

Polls live cluster API parameters directly via TCP sockets, presenting a clean console summary detailing slots, memory consumption, running versions, and replication directions.

./redis-tool status


Command 5: Zero-Downtime Rolling Upgrade

Safely promotes, fails over, compiles, and upgrades each cluster instance node-by-node without taking the cluster offline.

./redis-tool upgrade --target-version 7.2.6 --strategy rolling


Command 6: Full Verification Post-Deployment Audit

Launches a comprehensive 5-phase infrastructure health check (Data Integrity, Version Parity, Topology Health, Cluster Registry State, and Replication Link Status).

./redis-tool verify --full


3. Rolling Upgrade Strategy

The rolling upgrade orchestration pipeline guarantees zero data loss and zero client downtime using a highly optimized Replica-First, Failover-Master execution path.

       [Active Cluster on vOld]
                  │
                  ▼
      ┌───────────────────────┐
      │  Upgrade Replicas     │ ──► Verify Sync & Link Status
      └───────────────────────┘
                  │
                  ▼
      ┌───────────────────────┐
      │ Failover Live Master  │ ──► Promoting Upgraded Replica to Master
      └───────────────────────┘
                  │
                  ▼
      ┌───────────────────────┐
      │  Upgrade Old Master   │ ──► Upgrading Demoted Node (Now Replica)
      └───────────────────────┘
                  │
                  ▼
       [Active Cluster on vNew]


The Step-by-Step Lifecycle

Pre-flight Health Baseline Validation:
Before initiating any changes, the engine runs a dataset verification sweep. If the cluster state reports any issues or fails validation checks, the upgrade is aborted.

Upgrading Replica Nodes:
Because replicas do not directly serve client write slots, they are updated first. The engine upgrades a replica container, re-mounts the environment, and pauses execution until INFO replication explicitly reports master_link_status == up.

Graceful Failover Promotion:
When targeting an active Master, the engine identifies its upgraded replica partner. The CLI establishes connection to that replica and issues an intentional CLUSTER FAILOVER command. This safely moves active read/write slots onto the upgraded node.

Demoted Master Upgrades:
The old Master node (now safely demoted to replica status) is updated, restarted, and resynchronized back into the cluster topology.

Post-Upgrade Status Checks:
Runs the full 5-part verification audit to ensure version alignment across all targets.

Why This Strategy Was Chosen

Preservation of High-Availability: Clients can continue making reads and writes throughout the process because slot buckets are never unassigned.

Avoidance of Split-Brain/Partition Failures: Manual promotion of a synchronized, fully prepared replica node via standard FAILOVER ensures predictable node promotion behaviors, avoiding standard split-brain clustering problems.

4. Assumptions, Trade-offs, and Known Limitations

Key Assumptions

Unshared Port Arrays: The host environment assumes that loopback port ranges $3311-3316$ and $6371-6376$ remain completely unmapped by other local background processes.

Dynamic Root Boundary: The tool assumes that ansible.cfg remains in the root project folder alongside redis-tool so relative path computations are accurately anchored via Python's PROJECT_ROOT pointer.

Architectural Trade-offs

SSH-on-Loopback Connection Mode: Instead of running Ansible via standard direct ansible_connection=docker sockets, the orchestration relies on standard SSH tunnels mapped through localhost port configurations. This introduces the requirement of keeping an SSH daemon active inside the containers, but makes the tool completely portable and runtime-independent (works identically under Docker and Podman daemon layers).