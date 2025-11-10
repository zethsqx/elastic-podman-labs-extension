# 🧩 Extra Lab 1: Elasticsearch 3-Node Cluster (3 All-In-One/Uniform)

This lab spins up a **Elasticsearch cluster** using only `podman` commands — no Docker or Compose needed.  
It’s designed as extension of Elastic Instruqt Lab (ECK 101 Workshop) use, and works inside that minimal environments.

---

## 🪣 Prerequisites

- Linux host with **Podman** installed (pre-installed in the Instruqt)
- Internet access to `docker.elastic.co`

---

## ⚙️ Steps

### 1. Switch to root shell and clean up

You need `sudo` privileges to adjust networking:

```bash
sudo -i
```

### 2. Cleanup any existing containers or networks

```bash
podman stop es01 es02 es03 2>/dev/null || true
podman rm -f es01 es02 es03 2>/dev/null || true
podman network rm elastic 2>/dev/null || true
podman system prune -a -f
```

### 3. Create the Podman network

```bash
podman network create elastic

sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/' /etc/cni/net.d/elastic.conflist 2>/dev/null || true
sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/' ~/.config/cni/net.d/elastic.conflist 2>/dev/null || true
```

### 4. Start Elasticsearch nodes

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

### 5. Verify the cluster (Wait for a few minute for cluster to come up)

```bash
podman ps

curl -s http://localhost:9200/
curl -s http://localhost:9200/_cat/nodes?v
```

# 🧩 Extra Lab 2: Elasticsearch 4-Node Cluster (2 Dedicated Master, 1 Master+Hot, 1 Hot)

### 1. Clean up and recreate the Podman network

```bash
podman stop es01 es02 es03 hot01 2>/dev/null || true
podman rm -f es01 es02 es03 hot01 2>/dev/null || true

podman network create elastic

sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/' /etc/cni/net.d/elastic.conflist 2>/dev/null || true
sed -i 's/"cniVersion": "1.0.0"/"cniVersion": "0.4.0"/' ~/.config/cni/net.d/elastic.conflist 2>/dev/null || true
```
### 2. Create Self Signed Certficate

```bash
mkdir -p /home/student/es-lab/certs
cd /home/student/es-lab/certs

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=es-lab-CA"

openssl genrsa -out es-wildcard.key 4096

cat > es-wildcard.cnf <<EOF
[ req ]
default_bits       = 4096
distinguished_name = dn
req_extensions     = ext
prompt             = no

[ dn ]
CN = *.es-lab.local

[ ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = es01.es-lab.local
DNS.2 = es02.es-lab.local
DNS.3 = es03.es-lab.local
DNS.4 = hot01.es-lab.local
DNS.5 = es-lab.local
DNS.6 = *.es-lab.local
EOF

openssl req -new -key es-wildcard.key -out es-wildcard.csr -config es-wildcard.cnf
openssl x509 -req -in es-wildcard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out es-wildcard.crt -days 825 -sha256 -extensions ext -extfile es-wildcard.cnf

cd /home/student/es-lab
chmod -R 777 *

cd /home/student/es-lab/certs
```

### 3. Bring up Cluster

```bash
podman run -d --name es01 \
  --net elastic \
  -p 9200:9200 \
  -v /home/student/es-lab/certs:/usr/share/elasticsearch/config/certs:ro \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e ELASTIC_PASSWORD=Password123! \
  -e node.name=es01 \
  -e node.roles=master \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es02,es03,hot01 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms256m -Xmx256m" \
  -e xpack.security.enabled=true \
  -e xpack.security.http.ssl.enabled=true \
  -e xpack.security.http.ssl.verification_mode=certificate \
  -e xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  -e xpack.security.transport.ssl.enabled=true \
  -e xpack.security.transport.ssl.verification_mode=certificate \
  -e xpack.security.transport.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.transport.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.transport.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3

podman run -d --name es02 \
  --net elastic \
  -v /home/student/es-lab/certs:/usr/share/elasticsearch/config/certs:ro \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e ELASTIC_PASSWORD=Password123! \
  -e node.name=es02 \
  -e node.roles=master \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es01,es03,hot01 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms256m -Xmx256m" \
  -e xpack.security.enabled=true \
  -e xpack.security.http.ssl.enabled=true \
  -e xpack.security.http.ssl.verification_mode=certificate \
  -e xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  -e xpack.security.transport.ssl.enabled=true \
  -e xpack.security.transport.ssl.verification_mode=certificate \
  -e xpack.security.transport.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.transport.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.transport.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3

podman run -d --name es03 \
  --net elastic \
  -v /home/student/es-lab/certs:/usr/share/elasticsearch/config/certs:ro \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e ELASTIC_PASSWORD=Password123! \
  -e node.name=es03 \
  -e node.roles=master,data_hot,ingest \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es01,es02,hot01 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms256m -Xmx256m" \
  -e xpack.security.enabled=true \
  -e xpack.security.http.ssl.enabled=true \
  -e xpack.security.http.ssl.verification_mode=certificate \
  -e xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  -e xpack.security.transport.ssl.enabled=true \
  -e xpack.security.transport.ssl.verification_mode=certificate \
  -e xpack.security.transport.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.transport.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.transport.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3

podman run -d --name hot01 \
  --net elastic \
  -v /home/student/es-lab/certs:/usr/share/elasticsearch/config/certs:ro \
  --ulimit memlock=-1:-1 \
  --ulimit nofile=65536:65536 \
  -e ELASTIC_PASSWORD=Password123! \
  -e node.name=hot01 \
  -e node.roles=data_hot,ingest \
  -e cluster.name=es-lab-cluster \
  -e discovery.seed_hosts=es01,es02,es03 \
  -e cluster.initial_master_nodes=es01,es02,es03 \
  -e bootstrap.memory_lock=true \
  -e "ES_JAVA_OPTS=-Xms256m -Xmx256m" \
  -e xpack.security.enabled=true \
  -e xpack.security.http.ssl.enabled=true \
  -e xpack.security.http.ssl.verification_mode=certificate \
  -e xpack.security.http.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.http.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.http.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  -e xpack.security.transport.ssl.enabled=true \
  -e xpack.security.transport.ssl.verification_mode=certificate \
  -e xpack.security.transport.ssl.certificate=/usr/share/elasticsearch/config/certs/es-wildcard.crt \
  -e xpack.security.transport.ssl.key=/usr/share/elasticsearch/config/certs/es-wildcard.key \
  -e xpack.security.transport.ssl.certificate_authorities=/usr/share/elasticsearch/config/certs/ca.crt \
  docker.elastic.co/elasticsearch/elasticsearch:8.15.3
```

### 4. Test the connectivity and password

```bash
podman ps

curl -kv -u elastic:Password123! https://localhost:9200/
curl -kv -u elastic:Password123! https://localhost:9200/_cluster/health?pretty
curl -kv -u elastic:Password123! https://localhost:9200/_cat/nodes?v
```

### 5. Create a Data Stream

```bash
curl -kv -u elastic:Password123! -X PUT https://localhost:9200/_data_stream/logs-foo-bar
curl -kv -u elastic:Password123! https://localhost:9200/_cat/indices?expand_wildcards=all
```

# Clean up before proceeding

```bash
podman stop es01 es02 es03 hot01 2>/dev/null || true
podman rm -f es01 es02 es03 hot01 2>/dev/null || true
podman network rm elastic 2>/dev/null || true
podman system prune -a -f
```

