# pgpatroniha
PostgreSQL Patroni Setup with HAproxy 
# Patroni + PostgreSQL 17 HA Cluster — dedicated etcd tier (Vagrant)

A self-provisioning, high-availability **PostgreSQL 17** cluster managed by
**[Patroni](https://patroni.readthedocs.io)**, with a **dedicated, standalone
etcd cluster** as the distributed configuration store (DCS) and **HAProxy** as
the connection router.

This is the production-shaped variant of the topology: etcd runs on its **own
nodes**, isolated from the database workload, so DCS quorum and consensus are
not affected by PostgreSQL load, failovers, or resource pressure on the DB
hosts. (A *stacked* variant, where etcd co-locates on the PostgreSQL nodes, is
simpler/cheaper for labs — this repo is the separated version.)

Built in the spirit of the
[oraclebase/vagrant](https://github.com/oraclebase/vagrant) projects: a Vagrant
box (Oracle Linux 9) plus plain shell provisioning scripts, a `config/`
directory for all settings, and `shared_scripts/` mounted into each guest. The
whole cluster is defined in one **multi-machine** `Vagrantfile` driven by
`config/vagrant.yml`.

## Architecture

```
                       +---------------------------+
            client --> |  HAProxy  192.168.56.20   |
                       |   :5000 RW -> primary     |
                       |   :5001 RO -> replicas    |
                       |   :7000 stats UI          |
                       +-------------+-------------+
                                     |
                 +-------------------+-------------------+
                 |                   |                   |
            +----v----+         +----v----+         +----v----+
            |  pg1    |         |  pg2    |         |  pg3    |
            | .56.10  |         | .56.11  |         | .56.12  |
            | Patroni |         | Patroni |         | Patroni |
            | PG 17   |         | PG 17   |         | PG 17   |
            +----+----+         +----+----+         +----+----+
                 |   etcd3 client (2379)  |                |
                 +-----------+------------+----------------+
                             |  (Patroni -> DCS)
            +----------------+----------------+
            |                |                |
       +----v----+      +----v----+      +----v----+
       | etcd1   |<---->| etcd2   |<---->| etcd3   |   dedicated 3-node
       | .56.30  | 2380 | .56.31  | 2380 | .56.32  |   etcd DCS (peer :2380)
       +---------+      +---------+      +---------+
```

| Service          | Port  | Nodes              |
|------------------|-------|--------------------|
| etcd client      | 2379  | etcd1/etcd2/etcd3  |
| etcd peer        | 2380  | etcd1/etcd2/etcd3  |
| PostgreSQL       | 5432  | pg1/pg2/pg3        |
| Patroni REST API | 8008  | pg1/pg2/pg3        |
| HAProxy RW       | 5000  | haproxy            |
| HAProxy RO       | 5001  | haproxy            |
| HAProxy stats    | 7000  | haproxy            |

## Repository layout

```
patroni-pg17-etcd-separate/
├── Vagrantfile                 # multi-machine def, reads config/vagrant.yml
├── config/
│   ├── vagrant.yml             # topology + VM sizing (single source of truth)
│   ├── install.env             # versions, cluster scope, credentials, SSH
│   └── ssh/
│       ├── cluster_id_ed25519      # shared lab key (regenerate for real use)
│       └── cluster_id_ed25519.pub
├── shared_scripts/             # mounted into every guest at /vagrant_scripts
│   ├── setup.sh                # entry point, dispatches by node role
│   ├── configure_hosts.sh      # /etc/hosts from the cluster topology
│   ├── configure_os.sh         # time sync, SELinux, per-role firewall ports
│   ├── configure_ssh.sh        # root SSH login + inter-node key (all nodes)
│   ├── configure_ssh_postgres.sh # postgres SSH login + key (db nodes)
│   ├── install_etcd.sh         # standalone etcd node (etcd tier)
│   ├── install_postgresql.sh   # PostgreSQL 17 from PGDG (pg tier)
│   ├── install_patroni.sh      # Patroni (pip) -> points at the etcd tier
│   └── install_haproxy.sh      # HAProxy router config
├── scripts/
│   └── cluster-status.sh       # host-side helper to inspect the cluster
├── LICENSE
└── README.md
```

## Prerequisites

- [Vagrant](https://www.vagrantup.com/) 2.3+
- [VirtualBox](https://www.virtualbox.org/) 7.x (or libvirt/KVM with the
  `vagrant-libvirt` plugin)
- This topology is **7 VMs**. With the default sizing (etcd 512 MB ×3,
  pg 1536 MB ×3, haproxy 512 MB) it needs **~6.5 GB RAM** and ~12 GB disk.
  Lower the `memory` values in `config/vagrant.yml` if your host is tight, or
  drop to two PostgreSQL nodes (still valid HA).
- Internet access on host **and** guests (PGDG repo, etcd release, pip)

On Windows, ensure git does not convert the shell scripts to CRLF
(`git config --global core.autocrlf input` before cloning).

## Usage

**Order matters**: the etcd tier must reach quorum before Patroni can register.

```bash
git clone <your-repo-url> patroni-pg17-etcd-separate
cd patroni-pg17-etcd-separate

vagrant up etcd1 etcd2 etcd3     # 1) dedicated DCS first
vagrant up pg1 pg2 pg3           # 2) PostgreSQL + Patroni
vagrant up haproxy               # 3) router

# ...or just `vagrant up` - the etcd nodes are listed first in vagrant.yml.
```

> **On the first etcd node, etcd intentionally does not become healthy yet.** A
> fresh 3-member etcd cluster needs a quorum (2 of 3) before it finishes
> bootstrapping, so `etcd1` sits in the `activating` state until `etcd2` joins.
> This is expected — provisioning does **not** fail (the unit uses
> `--no-block` + `TimeoutStartSec=0`), and the cluster converges automatically.
> Likewise, Patroni on the pg nodes retries the DCS until the etcd tier is up.

Check the cluster:

```bash
./scripts/cluster-status.sh
```

Expected `patronictl list` (one Leader, two Replicas):

```
+ Cluster: pg17-cluster ----------+---------+----+-----------+
| Member | Host          | Role    | State   | TL | Lag in MB |
+--------+---------------+---------+---------+----+-----------+
| pg1    | 192.168.56.10 | Leader  | running |  1 |           |
| pg2    | 192.168.56.11 | Replica | running |  1 |         0 |
| pg3    | 192.168.56.12 | Replica | running |  1 |         0 |
+--------+---------------+---------+---------+----+-----------+
```

## Connecting

The `postgres` password is set in `config/install.env` (`PG_SUPERUSER_PASSWORD`).
```bash
# Read/write - always lands on the current primary
psql -h 192.168.56.20 -p 5000 -U postgres

# Read-only - round-robins across healthy replicas
psql -h 192.168.56.20 -p 5001 -U postgres
```

HAProxy stats: <http://192.168.56.20:7000/>

## SSH access (root & postgres)

Direct SSH login is enabled for the **root** and **postgres** OS users on every
node, and the two users can SSH **passwordless between cluster nodes** via a
shared key (`config/ssh/cluster_id_ed25519`). All of this is driven by the
`SSH_*` settings in `config/install.env`.

```bash
# From your host, hop onto a node as the vagrant user, then su - if you like:
vagrant ssh pg1

# Direct login as root or postgres (passwords in config/install.env:
#   SSH_ROOT_PASSWORD / SSH_POSTGRES_PASSWORD):
ssh root@192.168.56.10
ssh postgres@192.168.56.10

# Passwordless node-to-node (shared key), e.g. from pg1:
vagrant ssh pg1 -c "sudo -u postgres ssh postgres@pg2 hostname"
vagrant ssh pg1 -c "ssh root@etcd1 hostname"
```

`StrictHostKeyChecking` is disabled for the cluster hosts (`pg*`, `etcd*`,
`haproxy`, `192.168.56.*`) so the first connection doesn't prompt.

> **Lab convenience, not production hygiene.** This enables root SSH login,
> password authentication, and ships a *shared private key in the repo*. For
> anything real: set `SSH_ENABLE_ROOT_LOGIN`/`SSH_ENABLE_PASSWORD_AUTH` to
> `false`, regenerate `config/ssh/cluster_id_ed25519` (`ssh-keygen -t ed25519`),
> and use per-user keys. Change `SSH_ROOT_PASSWORD` and `SSH_POSTGRES_PASSWORD`.

## Testing failover

```bash
# Graceful switchover (you choose the new leader)
vagrant ssh pg1 -c "sudo patronictl -c /etc/patroni/patroni.yml switchover"

# Simulate a crash - stop Patroni on the leader; a replica is promoted within
# ~ttl seconds (30s default). The etcd tier is untouched by this.
vagrant ssh pg1 -c "sudo systemctl stop patroni"
vagrant ssh pg2 -c "sudo patronictl -c /etc/patroni/patroni.yml list"
```

HAProxy follows the leader automatically: its `OPTIONS /primary` health check
only returns `200` on whichever node holds the leader lock in etcd, so port
5000 always points at the live primary.

You can also test DCS resilience: stopping **one** etcd node (`vagrant ssh
etcd1 -c "sudo systemctl stop etcd"`) leaves quorum intact (2 of 3) and the
cluster keeps running. Stopping two breaks quorum — Patroni then holds the
current state and refuses failovers until quorum returns, as designed.

## Customising

- **Topology / sizing** — edit `config/vagrant.yml`. Add/remove etcd or pg
  nodes, change IPs or RAM; everything downstream (hosts file, etcd peer list,
  Patroni endpoints, HAProxy backends) regenerates automatically. Keep the etcd
  tier an **odd** count (3 or 5) for clean quorum.
- **Versions / credentials / tuning** — edit `config/install.env` (PostgreSQL
  major version, etcd version, cluster scope, passwords).
- **PostgreSQL parameters** — edit the `bootstrap.dcs.postgresql.parameters`
  block in `shared_scripts/install_patroni.sh`, or change them at runtime with
  `patronictl edit-config`.

## Notes & caveats

This is a **lab / learning** setup. Before anything resembling production:

- Change every password in `config/install.env`.
- etcd and the Patroni REST API run over plain HTTP here — add TLS (and client
  cert auth for etcd).
- SELinux is set permissive and only required ports are opened; prefer
  enforcing mode with proper policy in production.
- Add backups (e.g. `pgBackRest` or `barman`) — HA is not a backup.
- For real deployments, spread the three etcd nodes across separate
  failure domains (hosts/racks/AZs).

## Alternative boxes

`config/vagrant.yml` defaults to `oraclelinux/9`. Rocky/Alma Linux 9 also work
with the EL9 PGDG repo used here:

```yaml
box: "bento/rockylinux-9"
# and remove or update box_url
```

## Difference from the stacked variant

| | Stacked etcd | **Dedicated etcd (this repo)** |
|---|---|---|
| VMs | 4 (3 pg+etcd, 1 haproxy) | 7 (3 etcd, 3 pg, 1 haproxy) |
| etcd location | on the DB nodes | own nodes |
| RAM | ~6.5 GB | ~6.5 GB (etcd nodes small) |
| DCS isolation | none | full |
| Best for | labs, quick demos | production-shaped testing |

