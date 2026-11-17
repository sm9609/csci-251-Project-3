# CSCI-251 Chaos Engineering Lab - Docker Compose Starters

These Docker Compose configurations provide ready-to-use distributed system clusters for the Chaos Engineering Lab project.

## Available Systems

### Redis Cluster (`redis-cluster/`)
A 6-node Redis cluster with 3 masters and 3 replicas.

**Start:**
```bash
cd redis-cluster
docker-compose up -d

# Wait 10 seconds for nodes to start, then initialize cluster:
docker exec redis-1 redis-cli --cluster create \
  redis-1:6379 redis-2:6379 redis-3:6379 \
  redis-4:6379 redis-5:6379 redis-6:6379 \
  --cluster-replicas 1 --cluster-yes
```

**Verify:**
```bash
docker exec redis-1 redis-cli cluster info
docker exec redis-1 redis-cli cluster nodes
```

**Test read/write:**
```bash
docker exec redis-1 redis-cli -c set mykey "hello"
docker exec redis-1 redis-cli -c get mykey
```

---

### etcd Cluster (`etcd-cluster/`)
A 3-node etcd cluster using Raft consensus.

**Start:**
```bash
cd etcd-cluster
docker-compose up -d
```

**Verify:**
```bash
docker exec etcd-1 etcdctl endpoint health --cluster
docker exec etcd-1 etcdctl endpoint status --cluster -w table
```

**Test read/write:**
```bash
docker exec etcd-1 etcdctl put greeting "Hello, World"
docker exec etcd-1 etcdctl get greeting
```

**Watch for changes (in separate terminal):**
```bash
docker exec etcd-1 etcdctl watch greeting
```

---

### RabbitMQ Cluster (`rabbitmq-cluster/`)
A 3-node RabbitMQ cluster with management UI.

**Start:**
```bash
cd rabbitmq-cluster
docker-compose up -d

# Wait 30 seconds for cluster to form, then join nodes:
docker exec rabbitmq-2 rabbitmqctl stop_app
docker exec rabbitmq-2 rabbitmqctl join_cluster rabbit@rabbitmq-1
docker exec rabbitmq-2 rabbitmqctl start_app

docker exec rabbitmq-3 rabbitmqctl stop_app
docker exec rabbitmq-3 rabbitmqctl join_cluster rabbit@rabbitmq-1
docker exec rabbitmq-3 rabbitmqctl start_app

# Enable queue mirroring:
docker exec rabbitmq-1 rabbitmqctl set_policy ha-all ".*" \\
  '{"ha-mode":"all","ha-sync-mode":"automatic"}'
```

**Verify:**
```bash
docker exec rabbitmq-1 rabbitmqctl cluster_status
```

**Management UI:** http://localhost:15672 (guest/guest)

---

## Chaos Injection Commands

Use these Docker commands from your **host terminal** (not inside the containers).

### Node Failures (all systems)
```bash
# Kill a node (simulates crash)
docker stop <container>

# Bring it back
docker start <container>

# Freeze a node (simulates unresponsive process)
docker pause <container>
docker unpause <container>
```

### Network Partitions (all systems)
```bash
# Disconnect a node from the cluster network
docker network disconnect <network-name> <container>

# Reconnect it
docker network connect <network-name> <container>

# Find your network name
docker network ls
```

Network names follow the pattern `<folder>_<network>`, e.g., `redis-cluster_redis-net`, `mongo-cluster_mongo-net`.

### Advanced: Packet Loss and Latency (`tc`) and Fine-Grained Partitions (`iptables`)

`tc` and `iptables` change the network interface *inside* a container's network namespace, so they have to run there. There are two ways to do that:

**Option 1 - install the tools inside the container.** Works when the image has a shell and a package manager:

```bash
# Redis and RabbitMQ (Alpine images)
docker exec <container> apk add --no-cache iproute2 iptables

# MongoDB (Ubuntu image)
docker exec <container> sh -c "apt-get update && apt-get install -y iproute2 iptables"
```

**Option 2 - a toolbox container that shares the target's network.** Works for every system, and it is the *only* option for etcd, whose image has no shell at all (`docker exec etcd-1 sh` fails; only `etcd` and `etcdctl` exist in it):

```bash
# Open a shell that lives inside <container>'s network namespace
docker run --rm -it --cap-add NET_ADMIN --network container:<container> alpine sh
apk add iproute2 iptables
# ...run the tc / iptables commands below, then `exit`
```

Anything you do to `eth0` in the toolbox shell happens to `<container>`, because the two share one network namespace. The rules stay in place after you exit until you delete them (or restart the container).

The commands are the same either way:

```bash
# Packet loss (30%)
tc qdisc add dev eth0 root netem loss 30%

# Latency (500ms)
tc qdisc add dev eth0 root netem delay 500ms

# Remove network effects
tc qdisc del dev eth0 root

# Fine-grained partition (block one specific peer instead of the whole network)
iptables -A INPUT -s <target-ip> -j DROP
iptables -A OUTPUT -d <target-ip> -j DROP

# Remove partition
iptables -F
```

With Option 1, prefix each command with `docker exec <container>`. Node IPs are listed below.

**Note:** All starter configs already grant the containers `NET_ADMIN`; the toolbox container needs its own `--cap-add NET_ADMIN`, as shown. If any of this fights you, `docker stop`, `docker pause`, and `docker network disconnect` achieve the same chaos engineering goals and are perfectly acceptable experiments.

---

## IP Addresses

Each cluster uses a dedicated subnet with static IPs for predictable chaos injection:

| System | Subnet | Node IPs |
|--------|--------|----------|
| Redis | 172.28.0.0/16 | .11 through .16 |
| etcd | 172.29.0.0/16 | .11 through .13 |
| RabbitMQ | 172.30.0.0/16 | .11 through .13 |
| MongoDB | 172.27.0.0/16 | .11 through .13 |

---

### MongoDB Replica Set (`mongo-cluster/`)
A 3-node MongoDB replica set (one primary, two secondaries).

**Start:**
```bash
cd mongo-cluster
docker-compose up -d

# Wait ~10 seconds, then initialize the replica set (once):
docker exec mongo-1 mongosh --quiet --eval 'rs.initiate({_id:"rs0",members:[{_id:0,host:"mongo-1:27017"},{_id:1,host:"mongo-2:27017"},{_id:2,host:"mongo-3:27017"}]})'
```

**Verify (wait ~15 seconds after initiate for the election):**
```bash
docker exec mongo-1 mongosh --quiet --eval 'rs.status().members.forEach(m => print(m.name, m.stateStr))'
# Expect one PRIMARY and two SECONDARY
```

**Test read/write:** writes must go to the PRIMARY, and any node can be elected, so connect with the replica-set URI (the driver finds the primary for you):
```bash
docker exec mongo-1 mongosh --quiet "mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0" --eval 'db.test.insertOne({greeting:"Hello, World"}, {writeConcern:{w:"majority"}})'
docker exec mongo-1 mongosh --quiet "mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0" --eval 'db.test.find()'
```

**Interactive shell:** `docker exec -it mongo-1 mongosh "mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017/?replicaSet=rs0"` (follows the primary) or `docker exec -it mongo-1 mongosh` (talks only to mongo-1, useful during experiments)

**Key insight:** Writes go only to the primary and replicate to the secondaries. With 3 nodes, a majority (2) is needed to elect a primary, so the set survives one failure but refuses writes when two nodes are down. `w:"majority"` makes a write wait until a majority has it; `w:1` is acknowledged by the primary alone and can be rolled back if that primary is later partitioned away.

---

## Cleanup

```bash
# Stop and remove containers
docker-compose down

# Also remove volumes (deletes all data)
docker-compose down -v
```
