# galera-cluster-dev-794

## Architecture Overview
- **Nodes**: 3-node MariaDB Galera cluster (`galera-node1`, `galera-node2`, `galera-node3`)
- **Engine**: MariaDB 11.4 Community Container Image with Galera wsrep provider (`libgalera_smm.so`)
- **Network**: Docker bridge network (`galera-net`)
- **Port Mapping**:
  - `galera-node1`: 3306
  - `galera-node2`: 3307
  - `galera-node3`: 3308

## Verified Test Scenarios

### 1. Bootstrap & Quorum Formation
- Bootstrapped `galera-node1` with `--wsrep-cluster-address=gcomm://`.
- Joined nodes 2 and 3 pointing to `galera-node1`.
- Verified `wsrep_cluster_size = 3` and `wsrep_cluster_status = Primary`.
- Observed initial **State Snapshot Transfer (SST)** via rsync during new node join.

### 2. Multi-Master Writes & Virtually Synchronous Replication
- Wrote record to `galera-node1` (`glynac_db.inventory`), read immediately from `galera-node2`.
- Wrote secondary record to `galera-node3`, read immediately from `galera-node1`.
- Confirmed bi-directional write-set replication and storage across all nodes.

### 3. Node Outage & Incremental State Transfer (IST)
- Stopped `galera-node3` to simulate single-node failure.
- Verified remaining cluster retained majority quorum (`wsrep_cluster_size = 2`, `Primary`).
- Executed writes on surviving nodes during the outage.
- Restarted `galera-node3` and observed **IST** (write-set catchup via GCache sequence replay without full SST dump).
- Confirmed full dataset integrity across all 3 nodes upon recovery.

### 4. Split-Brain & Quorum Safeguards
- Disconnected `galera-node1` ungracefully from `galera-net`.
- `galera-node1` transitioned to `wsrep_cluster_status = non-Primary` (`weight = 0`).
- Attempted write on isolated node returned: `ERROR 1047 (08S01): WSREP has not yet prepared node for application use`.
- Reconnected node to network; cluster auto-healed back to `size = 3`, `Primary`.
