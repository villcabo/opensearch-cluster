# OpenSearch Swarm Cluster

This project deploys a 3-node OpenSearch cluster with OpenSearch Dashboards and Logstash on Docker Swarm.

## Architecture

*   **OpenSearch Nodes**: 3 nodes (`opens1`, `opens2`, `opens3`)
*   **Dashboards**: `opendash` (Port 5601)
*   **Logstash**: `logstash` (Port 5044)
*   **Persistence**: Docker named volumes (`data-01`, `data-02`, `data-03`)
*   **Placement**: Uses node labels `role=b1`, `role=b2`, `role=b3` to pin OpenSearch nodes to specific Swarm nodes.

## Prerequisites

*   Docker Swarm cluster initialized.
*   3 Swarm nodes available for the OpenSearch cluster.

## Deployment Steps

### 1. Label Your Nodes

You must assign the `role` label to your Swarm nodes to ensure each OpenSearch instance lands on the correct server.

Run the following commands on your Swarm manager (replace `<node-id>` with the actual node hostname or ID):

```bash
# Verify your nodes
docker node ls

# Assign labels
docker node update --label-add role=b1 <node-1-id>
docker node update --label-add role=b2 <node-2-id>
docker node update --label-add role=b3 <node-3-id>
```

### 2. Configure System & Storage

On **EACH** worker node (b1, b2, b3), you must configure the kernel limits and create the data directory with correct permissions.

**Run this on all 3 nodes:**

```bash
# 1. Set Max Map Count (Required for OpenSearch)
sudo sysctl -w vm.max_map_count=262144
# (Add 'vm.max_map_count=262144' to /etc/sysctl.conf to persist)

# 2. Create the Data Directory
# You can create specific folders for clarity, or just ensure the target path exists.
# For opens1 (Node 1):
mkdir -p /srv/opensearch/data/b1
# For opens2 (Node 2):
mkdir -p /srv/opensearch/data/b2
# For opens3 (Node 3):
mkdir -p /srv/opensearch/data/b3

# 3. Set Permissions
sudo chown -R 1000:1000 /srv/opensearch/data/
```

# 3. Create the Network
Create the overlay network manually so it allows other services to attach:

```bash
docker network create --driver overlay --attachable opensearch-net
```


### 4. Configure Admin Password (Docker Secret)

For Docker Swarm, the admin password is managed securely using a Docker secret.

**1. Generate a strong password:**

```bash
openssl rand -base64 18
# Ensure the result contains at least one uppercase, one lowercase, one digit, and one special character.
```

**2. Create the Docker secret:**

```bash
echo "<your_strong_password>" | docker secret create opensearch_admin_password -
```

Reemplaza `<your_strong_password>` por la contraseña generada.

> **Nota:** Si necesitas actualizar la contraseña, primero elimina el secret anterior:
> ```bash
> docker secret rm opensearch_admin_password
> ```
> y luego créalo de nuevo con el nuevo valor.

### 5. Deploy the Stacks

You need to deploy the clusters in order.

**1. Deploy Data Nodes:**
```bash
docker stack deploy -c opensearch-stack.yml opensearch
```
*Wait until the cluster is green.*

**2. Deploy Dashboards:**
```bash
docker stack deploy -c dashboard-stack.yml opendash
```

**3. Deploy Logstash:**
```bash
docker stack deploy -c logstash-stack.yml logstash
```

### 6. Verify Deployment

Check the service status:

```bash
docker service ls
```

Check the cluster health:

```bash
curl -XGET http://localhost:9200/_cluster/health?pretty -u admin:MyStrongPass123! --insecure
```

(Note: Ensure you are hitting the port on one of the swarm nodes, or use the service name if inside the network).

### 7. Access OpenSearch Dashboards

Open your browser and navigate to:

`http://<swarm-node-ip>:5601`

## Data Persistence & Storage Strategy

This deployment uses **Host Bind Mounts** combined with **Node Pinning**.

### Why this approach?
We map physical paths on the host to the container.
- Node 1 (`role=b1`): Uses `/srv/opensearch/data/b1`
- Node 2 (`role=b2`): Uses `/srv/opensearch/data/b2`
- Node 3 (`role=b3`): Uses `/srv/opensearch/data/b3`

### Storage Path
*   **Host Paths**: `/srv/opensearch/data/b1`, `/b2`, `/b3`
*   **Permissions**: Must be owned by UID `1000` (OpenSearch).
