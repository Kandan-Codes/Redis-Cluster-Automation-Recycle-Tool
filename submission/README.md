# Redis Cluster Lifecycle Tool

A CLI tool (`redis-tool-CLI`) that wraps hand-written Ansible playbooks to provision, operate, and perform a zero-downtime rolling upgrade of a 6-node Redis Cluster (3 masters + 3 replicas) running in Podman/Docker containers.

## Project Structure

```
submission/               
├── redis-tool-CLI              ← CLI entrypoint (Python, same commands)
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   └── hosts.ini
│   ├── playbooks/
│   │   ├── provision.yml
│   │   ├── data_seed.yml
│   │   ├── data_verify.yml
│   │   ├── status.yml
│   │   ├── upgrade.yml
│   │   ├── verify_full.yml
│   └── roles/
│       └── redis/
│           ├── defaults/
│           │   └── main.yml
│           ├── handlers/
│           ├── tasks/
│           │   └── main.yml
│           └── templates/
│               └── redis.conf.j2
├── infra/
│   ├── Containerfile
│   ├── compose.yml             ← base 6-node cluster
│   └── authorized_keys
├── logs/
│   └── operation_log.txt
├── output/
│   ├── provision_output.txt
│   ├── data_seed_output.txt
│   ├── status_output.txt
│   ├── upgrade_output.txt
│   └── verify_output.txt
└── README.md
```

## Bringing Up the Infrastructure

Works with either Podman (preferred) or Docker.

```bash
cd infra/
podman-compose up -d --build      # or: docker compose up -d --build
cd ..
```

This builds the Ubuntu 22.04 + SSH image from `Containerfile` and starts 6 containers (`redis-node-1`..`redis-node-6`) on a static `10.10.0.0/24` network at `10.10.0.11`–`10.10.0.16`, with SSH key auth already baked in via `infra/authorized_keys`.

## Running Each Command

```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
./redis-tool data seed --keys 1000
./redis-tool data verify
./redis-tool status
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
./redis-tool verify --full
```

Every command runs a prerequisite check first (Podman/Docker + Ansible 2.14+) and exits with a clear message if either is missing. Pass `--auto-install` to have it offer to install missing dependencies (asks for confirmation first).

Note: `--masters` and `--replicas-per-master` are accepted and logged, but the actual topology is currently fixed at 3+3 by the 6-container compose file — changing those flags doesn't yet resize the base cluster. Use `scale --add-nodes 2` to grow it instead (see below).

## Rolling Upgrade Strategy

`upgrade --target-version <ver> --strategy rolling` runs in three stages:

**Stage 0 — Pre-flight.** Confirms all 6 nodes are reachable, the cluster is currently `cluster_state:ok`, the requested version actually differs from what's running (if every node is already on the target version, the playbook prints a message and exits cleanly — this is the idempotency guarantee), and runs a `data verify` baseline before touching anything.

**Stage 1 — Replicas first, one at a time (`serial: 1`).** Each replica is stopped, recompiled to the target version, restarted, and checked against the live cluster (`cluster_state:ok`) before moving to the next. Replicas go first because they hold no hash slots, so taking one offline briefly has no client-visible effect.

**Stage 2 — Masters, one at a time, via failover.** For each master, its (already-upgraded) replica is promoted with `CLUSTER FAILOVER`, which hands off the hash slots without dropping client requests. The old master — now a replica — is then stopped, upgraded, and restarted, rejoining as a replica of the newly promoted master. Cluster health is re-checked after each node.

**Stage 3 — Post-upgrade verification.** Re-runs `data verify`, confirms every node reports the target version, and prints `UPGRADE COMPLETE` only if both checks pass.

If any task fails, Ansible halts immediately — it does not proceed to the next node and does not attempt an automatic rollback.

## Idempotency (S4)

- `provision`: the `redis` role now checks the installed binary version and whether Redis is already responding before doing anything. If a node is already at the requested version and running, the role skips download/compile/install/restart entirely and only refreshes the config file if it has actually drifted.
- `upgrade`: Stage 0 compares the currently-running version on every node against `--target-version`. If they already match, it prints a message and exits cleanly without touching any node.

## Assumptions and Trade-offs

- Redis is built from source inside each container so the exact requested version is always what's installed, rather than whatever a distro package happens to ship.
- Containers have no systemd, so Redis is started directly via `redis-server ... --daemonize yes` and stopped with `pkill -f redis-server`, rather than through a service unit.
- The control node reaches containers through host-mapped SSH ports (`localhost:2221`–`2228`) rather than container IPs directly, since the container network is isolated from the host's network namespace.
- `--cluster rebalance` is used for scale-out slot migration rather than manual `--cluster reshard`, trading fine-grained control for simplicity.

## Known Limitations

- `--masters` / `--replicas-per-master` on `provision` don't yet resize the base topology (see note above).
- This project ships two CLI entrypoints: `redis-tool-CLI` (Python). Both now support every command, including `provision`, `data seed/verify`, `status`, `upgrade` and `verify --full`. They call the same underlying Ansible playbooks, so either can be used interchangeably; `redis-tool-CLI` additionally parses raw Ansible output into a cleaner formatted report for `status`, `data verify`, and `verify --full`.
