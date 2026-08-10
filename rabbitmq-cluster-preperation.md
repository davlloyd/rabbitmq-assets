# Runbook 1: Core Cluster Provisioning & Telemetry

This runbook covers the initial setup of a 5-node RabbitMQ cluster, user access, and Prometheus/Grafana telemetry integration[cite: 5].

## 1. Environment Details

The following table outlines the target nodes for the primary cluster, which are deployed as Virtual Machines via OVA with host firewalls disabled[cite: 5]:

| Node Name | IP Address |
| :--- | :--- |
| `<PRIMARY_NODE_1_FQDN>` | `<PRIMARY_NODE_1_IP>`[cite: 5] |
| `<PRIMARY_NODE_2_FQDN>` | `<PRIMARY_NODE_2_IP>`[cite: 5] |
| `<PRIMARY_NODE_3_FQDN>` | `<PRIMARY_NODE_3_IP>`[cite: 5] |
| `<PRIMARY_NODE_4_FQDN>` | `<PRIMARY_NODE_4_IP>`[cite: 5] |
| `<PRIMARY_NODE_5_FQDN>` | `<PRIMARY_NODE_5_IP>`[cite: 5] |

## 2. Base Configuration & Clustering

* Enable the required management and stream plugins using `sudo rabbitmq-plugins enable rabbitmq_management rabbitmq_stream rabbitmq_stream_browser`[cite: 5].
* Copy the Erlang cookie located at `/var/lib/rabbitmq/.erlang.cookie` from Node 1 to all other nodes to authorize cluster communication[cite: 5].
* Set the network partition handling strategy to `pause_minority` inside `/etc/rabbitmq/conf.d/10-defaults.conf` on every node[cite: 5].

Execute the following sequence on nodes 2 through 5 to join them to Node 1[cite: 5]:

```bash
# 1. Stop the broker application
rabbitmqctl stop_app

# 2. Join the cluster using the short node name
rabbitmqctl join_cluster rabbit@<PRIMARY_NODE_1_HOSTNAME>

# 3. Start the broker application back up
rabbitmqctl start_app
```

Verify the final cluster health by running `rabbitmqctl cluster_status`[cite: 5].

## 3. Administrator Access

Create a dedicated administrator user to access the Management UI at `http://<PRIMARY_NODE_1_FQDN>:15672`[cite: 5]:

```bash
sudo rabbitmqctl add_user <ADMIN_USER> <ADMIN_PASSWORD>
sudo rabbitmqctl set_user_tags <ADMIN_USER> administrator
sudo rabbitmqctl set_permissions -p / <ADMIN_USER> ".*" ".*" ".*"
```

## 4. Telemetry (Prometheus & Grafana)

* Define a global cluster name for Grafana visibility by running `sudo rabbitmqctl set_cluster_name <PRIMARY_CLUSTER_NAME>`[cite: 5].
* Enable the required metric plugins: `rabbitmq-plugins enable rabbitmq_prometheus rabbitmq_standby_replication_prometheus rabbitmq_standby_replication_delayed_queue_prometheus`[cite: 5].
* Verify the metrics endpoint is active by curling `http://localhost:15692/metrics`[cite: 5].
* Import Grafana Dashboard ID `10991` (RabbitMQ-Overview) and Dashboard ID `11340` (RabbitMQ-Quorum-Queues)[cite: 5].

Add the following scrape configuration to your Prometheus server[cite: 5]:

```yaml
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: 
        - '<PRIMARY_NODE_1_FQDN>:15692'
        - '<PRIMARY_NODE_2_FQDN>:15692'
        - '<PRIMARY_NODE_3_FQDN>:15692'
        - '<PRIMARY_NODE_4_FQDN>:15692'
        - '<PRIMARY_NODE_5_FQDN>:15692'
```

---

