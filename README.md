# Beats → Kafka → Logstash → Elasticsearch → Kibana

A complete guide to setting up an **ELK 8.x Cluster with Kafka 3.x** on Linux (RHEL/CentOS/Rocky) using a minimal 3-VM setup.

---

## 🏗️ Architecture Overview

| VM | Hostname | Roles |
|---|---|---|
| VM1 | `elk-node1` | Elasticsearch (master+data), Zookeeper, Kafka Broker |
| VM2 | `elk-node2` | Elasticsearch (data), Kafka Broker, Logstash |
| VM3 | `elk-node3` | Elasticsearch (data), Kafka Broker, Kibana |

### Data Flow

```
App/Beats → Kafka Topic (logs) → Logstash Consumer → Elasticsearch → Kibana
```

---

## 📋 Prerequisites (All VMs)

### 1. System Requirements

```bash
# Minimum recommended specs per VM
# CPU: 4 cores | RAM: 8GB | Disk: 50GB+
```

### 2. Set Hostnames & /etc/hosts (on ALL VMs)

```bash
# On VM1
sudo hostnamectl set-hostname elk-node1

# On VM2
sudo hostnamectl set-hostname elk-node2

# On VM3
sudo hostnamectl set-hostname elk-node3
```

```bash
# Add to /etc/hosts on ALL VMs
sudo tee -a /etc/hosts <<EOF
192.168.1.101  elk-node1
192.168.1.102  elk-node2
192.168.1.103  elk-node3
EOF
# ⚠️ Replace IPs with your actual VM IPs
```

### 3. System Tuning (ALL VMs)

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Set vm.max_map_count for Elasticsearch
sudo tee -a /etc/sysctl.conf <<EOF
vm.max_map_count=262144
net.ipv4.tcp_retries2=5
EOF
sudo sysctl -p

# Set file limits
sudo tee -a /etc/security/limits.conf <<EOF
* soft nofile 65536
* hard nofile 65536
* soft nproc 4096
* hard nproc 4096
EOF
```

### 4. Install Java (ALL VMs)

Required for Kafka & Logstash.

```bash
sudo dnf install -y java-17-openjdk java-17-openjdk-devel
java -version
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
echo "export JAVA_HOME=/usr/lib/jvm/java-17-openjdk" >> ~/.bashrc
```

### 5. Firewall Rules (ALL VMs)

```bash
sudo firewall-cmd --permanent --add-port=9200/tcp   # Elasticsearch HTTP
sudo firewall-cmd --permanent --add-port=9300/tcp   # Elasticsearch Transport
sudo firewall-cmd --permanent --add-port=9092/tcp   # Kafka
sudo firewall-cmd --permanent --add-port=2181/tcp   # Zookeeper
sudo firewall-cmd --permanent --add-port=2888/tcp   # Zookeeper peer
sudo firewall-cmd --permanent --add-port=3888/tcp   # Zookeeper leader election
sudo firewall-cmd --permanent --add-port=5601/tcp   # Kibana
sudo firewall-cmd --permanent --add-port=5044/tcp   # Logstash Beats input
sudo firewall-cmd --reload
```

---

## 🐘 STEP 1 — Kafka + Zookeeper Setup (All 3 VMs)

### Install Kafka

```bash
cd /opt
sudo wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
sudo tar -xzf kafka_2.13-3.7.0.tgz
sudo mv kafka_2.13-3.7.0 kafka
sudo useradd -r -s /sbin/nologin kafka
sudo chown -R kafka:kafka /opt/kafka

# Create data dirs
sudo mkdir -p /var/lib/zookeeper /var/log/kafka
sudo chown -R kafka:kafka /var/lib/zookeeper /var/log/kafka
```

### Configure Zookeeper (All VMs)

```bash
sudo tee /opt/kafka/config/zookeeper.properties <<EOF
dataDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=60
tickTime=2000
initLimit=10
syncLimit=5
# Cluster peers
server.1=elk-node1:2888:3888
server.2=elk-node2:2888:3888
server.3=elk-node3:2888:3888
EOF

# Set unique ZK ID — use 1, 2, or 3 per VM
echo "1" | sudo tee /var/lib/zookeeper/myid   # VM1
echo "2" | sudo tee /var/lib/zookeeper/myid   # VM2
echo "3" | sudo tee /var/lib/zookeeper/myid   # VM3
```

### Configure Kafka Broker (per VM)

> ⚠️ Change `broker.id` and `advertised.listeners` on each VM.

```bash
# VM1 example
sudo tee /opt/kafka/config/server.properties <<EOF
broker.id=1                          # Change to 2 on VM2, 3 on VM3
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://elk-node1:9092   # Change hostname per VM
num.network.threads=3
num.io.threads=8
log.dirs=/var/log/kafka
num.partitions=3
default.replication.factor=3
min.insync.replicas=2
log.retention.hours=168
zookeeper.connect=elk-node1:2181,elk-node2:2181,elk-node3:2181
zookeeper.connection.timeout.ms=18000
auto.create.topics.enable=true
EOF
```

### Create Systemd Services (All VMs)

```bash
# Zookeeper service
sudo tee /etc/systemd/system/zookeeper.service <<EOF
[Unit]
Description=Apache Zookeeper
After=network.target

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/zookeeper-server-start.sh /opt/kafka/config/zookeeper.properties
ExecStop=/opt/kafka/bin/zookeeper-server-stop.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# Kafka service
sudo tee /etc/systemd/system/kafka.service <<EOF
[Unit]
Description=Apache Kafka
After=zookeeper.service

[Service]
Type=simple
User=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable zookeeper kafka
sudo systemctl start zookeeper
sleep 10
sudo systemctl start kafka
sudo systemctl status kafka zookeeper
```

### Verify Kafka Cluster

```bash
# Create a test topic
/opt/kafka/bin/kafka-topics.sh --create \
  --topic logs \
  --bootstrap-server elk-node1:9092,elk-node2:9092,elk-node3:9092 \
  --partitions 3 \
  --replication-factor 3

# Verify
/opt/kafka/bin/kafka-topics.sh --describe --topic logs \
  --bootstrap-server elk-node1:9092
```

---

## 🔍 STEP 2 — Elasticsearch Cluster (All 3 VMs)

### Install Elasticsearch

```bash
# Import GPG key and add repo (ALL VMs)
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

sudo tee /etc/yum.repos.d/elasticsearch.repo <<EOF
[elasticsearch]
name=Elasticsearch repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

sudo dnf install -y elasticsearch
```

### Configure Elasticsearch

**VM1** — `/etc/elasticsearch/elasticsearch.yml`:

```yaml
cluster.name: elk-cluster
node.name: elk-node1
node.roles: [master, data]
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["elk-node1", "elk-node2", "elk-node3"]
cluster.initial_master_nodes: ["elk-node1", "elk-node2", "elk-node3"]
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /etc/elasticsearch/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /etc/elasticsearch/certs/elastic-certificates.p12
```

**VM2** — same as above but with:
```yaml
node.name: elk-node2
node.roles: [data]
```

**VM3** — same as above but with:
```yaml
node.name: elk-node3
node.roles: [data]
```

### Generate TLS Certificates (VM1 only, then copy to others)

```bash
# On VM1
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

sudo mkdir -p /etc/elasticsearch/certs
sudo mv elastic-certificates.p12 /etc/elasticsearch/certs/
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs

# Copy certs to VM2 and VM3
sudo scp /etc/elasticsearch/certs/elastic-certificates.p12 \
  user@elk-node2:/etc/elasticsearch/certs/
sudo scp /etc/elasticsearch/certs/elastic-certificates.p12 \
  user@elk-node3:/etc/elasticsearch/certs/

# On VM2 and VM3
sudo chown -R elasticsearch:elasticsearch /etc/elasticsearch/certs
```

### Start Elasticsearch & Set Passwords

```bash
# Start on ALL VMs
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# On VM1 — set built-in passwords
sudo /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive
# ⚠️ Save all passwords — especially 'elastic' and 'kibana_system'

# Verify cluster health
curl -u elastic:YOUR_PASSWORD https://elk-node1:9200/_cluster/health?pretty -k
```

---

## 📊 STEP 3 — Kibana (VM3)

```bash
sudo dnf install -y kibana

sudo tee /etc/kibana/kibana.yml <<EOF
server.port: 5601
server.host: "0.0.0.0"
server.name: "elk-node3"
elasticsearch.hosts: ["https://elk-node1:9200","https://elk-node2:9200","https://elk-node3:9200"]
elasticsearch.username: "kibana_system"
elasticsearch.password: "YOUR_KIBANA_SYSTEM_PASSWORD"
elasticsearch.ssl.verificationMode: none
logging.dest: /var/log/kibana/kibana.log
EOF

sudo systemctl enable kibana
sudo systemctl start kibana
# Access at http://elk-node3:5601
```

---

## 🔄 STEP 4 — Logstash with Kafka Input (VM2)

```bash
sudo dnf install -y logstash

sudo tee /etc/logstash/conf.d/kafka-to-elastic.conf <<EOF
input {
  kafka {
    bootstrap_servers => "elk-node1:9092,elk-node2:9092,elk-node3:9092"
    topics => ["logs"]
    group_id => "logstash-consumer"
    codec => "json"
    consumer_threads => 3
    decorate_events => true
  }
}

filter {
  if [message] {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
      tag_on_failure => ["_grok_failure"]
    }
  }
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
  }
  mutate {
    remove_field => ["@version"]
  }
}

output {
  elasticsearch {
    hosts => ["https://elk-node1:9200", "https://elk-node2:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "YOUR_ELASTIC_PASSWORD"
    ssl => true
    ssl_certificate_verification => false
  }
  stdout { codec => rubydebug }
}
EOF

sudo systemctl enable logstash
sudo systemctl start logstash
sudo systemctl status logstash
```

---

## ✅ STEP 5 — Verify Everything

```bash
# 1. Kafka cluster
/opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server elk-node1:9092,elk-node2:9092,elk-node3:9092

# 2. Elasticsearch cluster health
curl -u elastic:PASSWORD https://elk-node1:9200/_cluster/health?pretty -k

# 3. Check nodes
curl -u elastic:PASSWORD https://elk-node1:9200/_cat/nodes?v -k

# 4. Send test message to Kafka
echo '{"level":"INFO","message":"test log","service":"app1"}' | \
  /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server elk-node1:9092 --topic logs

# 5. Check index in Elasticsearch
curl -u elastic:PASSWORD \
  https://elk-node1:9200/_cat/indices/logs-*?v -k
```

---

## 🔁 Component Port Reference

| Component | Port | Protocol |
|---|---|---|
| Elasticsearch HTTP | 9200 | TCP |
| Elasticsearch Transport | 9300 | TCP |
| Kafka Broker | 9092 | TCP |
| Zookeeper Client | 2181 | TCP |
| Zookeeper Peer | 2888 | TCP |
| Zookeeper Election | 3888 | TCP |
| Kibana | 5601 | TCP |
| Logstash Beats Input | 5044 | TCP |

---

## 📝 Notes

- Replace all `YOUR_PASSWORD` / `YOUR_ELASTIC_PASSWORD` / `YOUR_KIBANA_SYSTEM_PASSWORD` placeholders with actual secure passwords.
- Replace `192.168.1.10x` IPs with your actual VM IPs in `/etc/hosts`.
- The `stdout` output in Logstash should be removed in production environments.
- TLS certificates generated by `elasticsearch-certutil` are self-signed — use a proper CA in production.

---

## 📚 References

- [Elasticsearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Docs](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Logstash Docs](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Apache Kafka Docs](https://kafka.apache.org/documentation/)
