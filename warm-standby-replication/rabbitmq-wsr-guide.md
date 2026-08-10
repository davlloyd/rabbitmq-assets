# Runbook 2: Tanzu RabbitMQ Warm Standby Operations

**Golden Architectural Rule:** The Tanzu Schema Sync engine forces the Standby cluster to act as a 1:1 mirror of the Primary cluster[cite: 5, 6]. Dynamic configurations, such as message endpoints and vhost tags, must be applied to the Primary cluster first so they can securely synchronize downward[cite: 5, 6].

## Phase 1: Replication Foundation

* Define the operating modes in `rabbitmq.conf` and restart the nodes to apply them[cite: 5, 6].
* The active cluster must have `schema_definition_sync.operating_mode = upstream` and `standby.replication.operating_mode = upstream`[cite: 5, 6].
* The passive cluster must have `schema_definition_sync.operating_mode = downstream` and `standby.replication.operating_mode = downstream`[cite: 5, 6].
* Create the replication service account (`<REPL_USER>`) with identical credentials on both clusters[cite: 5, 6].

Apply the following permissions and virtual host tags on the Primary cluster[cite: 5, 6]:

```bash
rabbitmqctl add_user <REPL_USER> <REPL_PASSWORD>
rabbitmqctl set_user_tags <REPL_USER> administrator
rabbitmqctl set_permissions -p / <REPL_USER> ".*" ".*" ".*"

sudo rabbitmqctl set_permissions -p rabbitmq_schema_definition_sync <REPL_USER> ".*" ".*" ".*"
sudo rabbitmqctl set_permissions -p rabbitmq_standby_replication <REPL_USER> ".*" ".*" ".*"
sudo rabbitmqctl set_vhost_tags / standby_replication
```

## Phase 2: Traffic Generation Demo

To demonstrate replication telemetry, generate synthetic traffic against a quorum queue named `<TARGET_QUEUE_NAME>`[cite: 5, 6]:

```bash
# Start a publisher generating 100 persistent messages per second
docker run -it --rm pivotalrabbitmq/perf-test:latest \
    --uri "amqp://<ADMIN_USER>:<ADMIN_PASSWORD>@<PRIMARY_NODE_1_FQDN>:5672" \
    --queue "<TARGET_QUEUE_NAME>" \
    --queue-args x-queue-type=quorum \
    --auto-delete false \
    --flag persistent \
    --producers 5 \
    --consumers 0 \
    --rate 100 \
    --confirm 100

# Drain the queue with 5 concurrent consumers
docker run -it --rm pivotalrabbitmq/perf-test:latest \
    --uri "amqp://<ADMIN_USER>:<ADMIN_PASSWORD>@<PRIMARY_NODE_1_FQDN>:5672" \
    --queue "<TARGET_QUEUE_NAME>" \
    --queue-args x-queue-type=quorum \
    --producers 0 \
    --consumers 5 \
    --qos 1
```

## Phase 3: Activating Downstream Replication

Execute this sequence on the Downstream cluster (`<STANDBY_CLUSTER_NAME>`) to establish connections to the Upstream cluster (`<PRIMARY_CLUSTER_NAME>`)[cite: 5, 6]:

```bash
# 1. Enable replication plugins
rabbitmq-plugins enable rabbitmq_warm_standby rabbitmq_schema_definition_sync_prometheus rabbitmq_standby_replication_prometheus rabbitmq_standby_replication_delayed_queue_prometheus

# 2. Set global operating modes to downstream
rabbitmqctl set_schema_replication_mode downstream
rabbitmqctl set_standby_replication_mode downstream

# 3. Configure upstream endpoints and authentication
rabbitmqctl set_schema_replication_upstream_endpoints '{"endpoints":["<PRIMARY_NODE_1_HOSTNAME>:5672","<PRIMARY_NODE_2_HOSTNAME>:5672","<PRIMARY_NODE_3_HOSTNAME>:5672"],"username":"<REPL_USER>","password":"<REPL_PASSWORD>"}'
rabbitmqctl set_standby_replication_upstream_endpoints '{"endpoints":["<PRIMARY_NODE_1_HOSTNAME>:5552","<PRIMARY_NODE_2_HOSTNAME>:5552","<PRIMARY_NODE_3_HOSTNAME>:5552"],"username":"<REPL_USER>","password":"<REPL_PASSWORD>"}'

# 4. Enable and restart node-local background processes across all standby nodes
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> enable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> restart_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> enable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> restart_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> enable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> restart_warm_standby

# 5. Verify the replication status
rabbitmqctl schema_replication_status
rabbitmqctl standby_replication_status
```

## Phase 4: Cluster Promotion & Reversion

To promote the standby cluster to a primary role during a disaster recovery event, execute `rabbitmqctl promote_warm_standby`[cite: 5, 6].

To revert the promoted cluster back to a downstream state, execute the following full recovery sequence[cite: 5, 6]:

```bash
# 1. Set global operating modes to downstream in Mnesia
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> set_schema_replication_mode downstream
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> set_standby_replication_mode downstream
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> set_schema_replication_mode downstream
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> set_standby_replication_mode downstream
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> set_schema_replication_mode downstream
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> set_standby_replication_mode downstream

# 2. Configure Upstream Endpoints
rabbitmqctl set_schema_replication_upstream_endpoints '{"endpoints":["<PRIMARY_NODE_1_HOSTNAME>:5672","<PRIMARY_NODE_2_HOSTNAME>:5672","<PRIMARY_NODE_3_HOSTNAME>:5672"],"username":"<REPL_USER>","password":"<REPL_PASSWORD>"}'
rabbitmqctl set_standby_replication_upstream_endpoints '{"endpoints":["<PRIMARY_NODE_1_HOSTNAME>:5552","<PRIMARY_NODE_2_HOSTNAME>:5552","<PRIMARY_NODE_3_HOSTNAME>:5552"],"username":"<REPL_USER>","password":"<REPL_PASSWORD>"}'

# 3. Clear promoted test data
rabbitmqctl delete_all_data_on_standby_replication_cluster

# 4. Enable processes across all nodes
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> enable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> enable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> enable_warm_standby

# 5. Flush Erlang VM memory by cycling the RabbitMQ application layer
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> stop_app && rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> start_app
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> stop_app && rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> start_app
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> stop_app && rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> start_app

# 6. Trigger connection to Primary Cluster
rabbitmqctl restart_warm_standby
rabbitmqctl connect_standby_replication_downstream
```

## Phase 5: Decommission & Cleanup

To completely disable replication and revert a downstream cluster back to a standalone state, run the following[cite: 5, 6]:

```bash
# 1. Empty the internal endpoint lists in Mnesia
rabbitmqctl clear_global_parameter schema_definition_sync_upstream
rabbitmqctl clear_global_parameter standby_replication_upstream

# 2. Stop WSR background processes on all nodes
rabbitmqctl -n rabbit@<STANDBY_NODE_1_HOSTNAME> disable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_2_HOSTNAME> disable_warm_standby
rabbitmqctl -n rabbit@<STANDBY_NODE_3_HOSTNAME> disable_warm_standby

# 3. Disable WSR plugins permanently across all nodes
rabbitmq-plugins -n rabbit@<STANDBY_NODE_1_HOSTNAME> disable rabbitmq_warm_standby rabbitmq_schema_definition_sync rabbitmq_standby_replication
rabbitmq-plugins -n rabbit@<STANDBY_NODE_2_HOSTNAME> disable rabbitmq_warm_standby rabbitmq_schema_definition_sync rabbitmq_standby_replication
rabbitmq-plugins -n rabbit@<STANDBY_NODE_3_HOSTNAME> disable rabbitmq_warm_standby rabbitmq_schema_definition_sync rabbitmq_standby_replication

# 4. Clean up WSR internal virtual host
rabbitmqctl delete_vhost rabbitmq_schema_definition_sync
```