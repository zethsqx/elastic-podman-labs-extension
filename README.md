# 🧩 Elasticsearch 3-Node Cluster on Podman

This lab spins up as identical of a **production-style Elasticsearch cluster** using only `podman` commands — no Docker or Compose needed.  
It’s designed as extension of Elastic Instruqt Lab (ECK 101 Workshop) use, and works inside that minimal environments.

---

## 🪣 Prerequisites

- Linux host with **Podman** installed (pre-installed in the Instruqt)
- Internet access to `docker.elastic.co`

---

## ⚙️ Steps

### 1️⃣ Switch to root shell and clean up

You need `sudo` privileges to adjust networking:

```bash
sudo -i
```

### 2️⃣ Cleanup any existing containers or networks
# Clean up any existing containers

```bash
podman stop es01 es02 es03 2>/dev/null || true
podman rm -f es01 es02 es03 2>/dev/null || true
podman network rm elastic 2>/dev/null || true
podman system prune -a -f
```

### 3️⃣ Create the Podman network

```bash
podman network create elastic

sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/' /etc/cni/net.d/elastic.conflist 2>/dev/null || true
sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/' ~/.config/cni/net.d/elastic.conflist 2>/dev/null || true
```

### 4️⃣ Start Elasticsearch nodes

```bash
podman run -d --name es01 \
  --net elastic \
  -p 9200:9200 \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e node.name=es01 \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es02,es03 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e xpack.security.enabled=false \
  -e xpack.security.http.ssl.enabled=false \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3

podman run -d --name es02 \
  --net elastic \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e node.name=es02 \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es01,es03 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e xpack.security.enabled=false \
  -e xpack.security.http.ssl.enabled=false \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3

podman run -d --name es03 \
  --net elastic \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e node.name=es03 \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es01,es02 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  -e xpack.security.enabled=false \
  -e xpack.security.http.ssl.enabled=false \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3
```

### 5️⃣ Verify the cluster (Wait for a few minute for cluster to come up)

```bash
podman ps

curl -s http://localhost:9200/
curl -s http://localhost:9200/_cat/nodes?v
```
