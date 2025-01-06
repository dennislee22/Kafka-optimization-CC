# Kafka Optimization



```
# kubectl run kafka-admin -it -n $NAMESPACE --image=$IMAGE --rm=true --restart=Never --command -- /opt/kafka/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --create --topic dlee-topic1 --partitions 3 --replication-factor 3
```

```
$ /opt/kafka/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic dlee-topic1
[2025-01-06 08:18:26,460] WARN [AdminClient clientId=adminclient-1] The DescribeTopicPartitions API is not supported, using Metadata API to describe topics. (org.apache.kafka.clients.admin.KafkaAdminClient)
Topic: dlee-topic1	TopicId: CH9jdZ9nSM2rG3on_6ixnQ	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2,message.format.version=3.0-IV1
	Topic: dlee-topic1	Partition: 0	Leader: 1	Replicas: 1,2,0	Isr: 1,2,0	Elr: N/A	LastKnownElr: N/A
	Topic: dlee-topic1	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2	Elr: N/A	LastKnownElr: N/A
	Topic: dlee-topic1	Partition: 2	Leader: 2	Replicas: 2,0,1	Isr: 2,0,1	Elr: N/A	LastKnownElr: N/A
```

```
from kafka import KafkaProducer
import time
import json

# Define the Kafka broker and topic
kafka_broker = 'my-cluster-kafka-bootstrap.dlee-kafkanodepool.svc.cluster.local:9092'
topic_name = 'dlee-topic1'

# Create a KafkaProducer
producer = KafkaProducer(
    bootstrap_servers=kafka_broker,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Simulating uneven message distribution by manually choosing partitions
partition_0 = 0
partition_1 = 2
partition_2 = 2

# Produce messages to partitions manually
count = 0
while True:
    message = {"message": f"Message number {count}"}
    
    # Simulate uneven load by sending more messages to partition 0
    if count % 3 == 0:
        producer.send(topic_name, value=message, partition=partition_0)
    elif count % 3 == 1:
        producer.send(topic_name, value=message, partition=partition_1)
    else:
        producer.send(topic_name, value=message, partition=partition_2)
    
    producer.flush()
    print(f"Produced message {message['message']} to partition {count % 3}")
    count += 1
    time.sleep(0.1)  # Simulate a steady stream of messages
```

```
# kubectl run kafka-check -it -n $NAMESPACE --image=$IMAGE --rm=true --restart=Never --command -- /bin/sh
```

```
$ /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic dlee-topic1 --group dlee-group1 --from-beginning
{"message": "Test message 1"}
{"message": "Test message 2"}
{"message": "Test message 3"}
{"message": "Test message 7"}
{"message": "Test message 9"}
```

```
$ /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
dlee-group1
```

```
$ /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --group dlee-group1

Consumer group 'dlee-group1' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
dlee-group1     dlee-topic1     1          304             1280            976             -               -               -
dlee-group1     dlee-topic1     2          321             3516            3195            -               -               -
dlee-group1     dlee-topic1     0          323             2410            2087            -               -               -
```

```
$ curl -k -u "healthcheck:VAnML37e3OyG" "https://localhost:9090/kafkacruisecontrol/KAFKA_CLUSTER_STATE"
Summary: The cluster has 3 brokers, 360 replicas, 121 leaders, and 8 topics with avg replication factor: 2.98. [Leaders/Replicas per broker] avg: 40.33/120.00 max: 41/120 std: 0.94/0.00

Brokers:
              BROKER           LEADER(S)            REPLICAS         OUT-OF-SYNC             OFFLINE       IS_CONTROLLER          BROKER_SET
                   0                  41                 120                   0                   0               false                null
                   1                  39                 120                   0                   0                true                null
                   2                  41                 120                   0                   0               false                null

LogDirs of brokers with replicas:
              BROKER                                  ONLINE-LOGDIRS  OFFLINE-LOGDIRS
                   0              [/var/lib/kafka/data-0/kafka-log0]               []
                   1              [/var/lib/kafka/data-0/kafka-log1]               []
                   2              [/var/lib/kafka/data-0/kafka-log2]               []

Under Replicated, Offline, and Under MinIsr Partitions:
                    TOPIC PARTITION    LEADER                      REPLICAS                       IN-SYNC              OUT-OF-SYNC                  OFFLINE   MIN-ISR
Offline Partitions:
Partitions with Offline Replicas:
Under Replicated Partitions:
Under MinIsr Partitions:
```

```
$ curl -k -u "healthcheck:VAnML37e3OyG" "https://localhost:9090/kafkacruisecontrol/PARTITION_LOAD?topic=ktopic-1"
                                        PARTITION    LEADER                     FOLLOWERS       CPU (%_CORES)           DISK (MB)        NW_IN (KB/s)       NW_OUT (KB/s)        MSG_IN (#/s)
                                       ktopic-1-1         2                           [1]           0.000000              3.553              0.000              0.000              0.000
                                       ktopic-1-2         1                           [0]           0.000000              1.777              0.000              0.000              0.000
                                       ktopic-1-0         0                           [2]           0.000000              0.000              0.000              0.000              0.000
```

```
# kubectl -n dlee-kafkanodepool apply -f rebalance.yml
kafkarebalance.kafka.strimzi.io/full-rebalance created
```

```
# kubectl -n dlee-kafkanodepool get kafkarebalance
NAME             CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
full-rebalance   my-cluster                     True
```

<img width="1432" alt="image" src="https://github.com/user-attachments/assets/d49009da-2558-4b10-b35b-216569e7ee47" />

<img width="1431" alt="image" src="https://github.com/user-attachments/assets/58da1206-6a33-4d38-824a-fc3b915227e0" />

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: rebalance-1
  namespace: dlee-kafkanodepool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  #goals:
  #  - "DiskCapacityGoal"
  #skipHardGoalCheck: true
  mode: add-brokers
  brokers: [3]
```

```
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool scale kafkanodepool nodepool-1 --replicas=4
kafkanodepool.kafka.strimzi.io/nodepool-1 scaled
[root@ecs-m-01 ~]# 
[root@ecs-m-01 ~]# 
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get pods
NAME                                          READY   STATUS     RESTARTS      AGE
jupyter-notebook                              1/1     Running    0             7h12m
kafka-cluster-1-zookeeper-0                   1/1     Running    0             3h42m
kafka-cluster-1-zookeeper-1                   1/1     Running    0             3h42m
kafka-cluster-1-zookeeper-2                   1/1     Running    0             3h42m
my-cluster-cruise-control-7f9876696f-hfj2h    1/1     Running    0             131m
my-cluster-entity-operator-785876bcff-bt6nn   2/2     Running    8 (27h ago)   34h
my-cluster-nodepool-1-0                       1/1     Running    0             3h45m
my-cluster-nodepool-1-1                       1/1     Running    0             3h45m
my-cluster-nodepool-1-2                       1/1     Running    0             3h45m
my-cluster-nodepool-1-3                       0/1     Init:0/1   0             8s
my-cluster-zookeeper-0                        1/1     Running    0             28h
my-cluster-zookeeper-1                        1/1     Running    0             28h
my-cluster-zookeeper-2                        1/1     Running    0             28h
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get pvc
NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-0-my-cluster-nodepool-1-0     Bound    pvc-cde00a7b-c4da-48e5-85fa-582a10b64d28   16Mi       RWO            longhorn       3h46m
data-0-my-cluster-nodepool-1-1     Bound    pvc-5806fd3c-3c14-4aec-bc89-eb1f7af3c6f7   16Mi       RWO            longhorn       3h46m
data-0-my-cluster-nodepool-1-2     Bound    pvc-8e468adf-0079-4a8a-a6a4-7e24f9e2e113   16Mi       RWO            longhorn       3h46m
data-0-my-cluster-nodepool-1-3     Bound    pvc-ac3ed582-a30b-4fc5-88d8-5960e3ba2469   16Mi       RWO            longhorn       60s
data-kafka-cluster-1-zookeeper-0   Bound    pvc-840d459c-7cb7-4f24-be18-d40b8966afcf   100Gi      RWO            longhorn       3h45m
data-kafka-cluster-1-zookeeper-1   Bound    pvc-c31fda34-7cf7-4798-a206-63c650151497   100Gi      RWO            longhorn       3h45m
data-kafka-cluster-1-zookeeper-2   Bound    pvc-6ed39c29-55d8-4d1a-9d11-ec4d0fca4442   100Gi      RWO            longhorn       3h45m
data-my-cluster-zookeeper-0        Bound    pvc-a5198bc1-cc90-439b-b413-92aea57cc3dc   100Gi      RWO            longhorn       34h
data-my-cluster-zookeeper-1        Bound    pvc-d6641d3c-d833-4b8f-b473-e470b71c3619   100Gi      RWO            longhorn       34h
data-my-cluster-zookeeper-2        Bound    pvc-555299e5-d5f3-4a84-957a-49f07df0c6ff   100Gi      RWO            longhorn       34h
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get pods
NAME                                          READY   STATUS        RESTARTS      AGE
jupyter-notebook                              1/1     Running       0             7h13m
kafka-cluster-1-zookeeper-0                   1/1     Running       0             3h43m
kafka-cluster-1-zookeeper-1                   1/1     Running       0             3h43m
kafka-cluster-1-zookeeper-2                   1/1     Running       0             3h43m
my-cluster-cruise-control-6469cd5c74-82hlm    1/1     Running       0             24s
my-cluster-cruise-control-7f9876696f-hfj2h    1/1     Terminating   0             132m
my-cluster-entity-operator-785876bcff-bt6nn   2/2     Running       8 (28h ago)   34h
my-cluster-nodepool-1-0                       1/1     Running       0             3h46m
my-cluster-nodepool-1-1                       1/1     Running       0             3h46m
my-cluster-nodepool-1-2                       1/1     Running       0             3h46m
my-cluster-nodepool-1-3                       1/1     Running       0             66s
my-cluster-zookeeper-0                        1/1     Running       0             28h
my-cluster-zookeeper-1                        1/1     Running       0             28h
my-cluster-zookeeper-2                        1/1     Running       0             28h
[root@ecs-m-01 ~]# vi kafkanodepool.yml 
[root@ecs-m-01 ~]# vi rebalance.yml 
[root@ecs-m-01 ~]# vi rebalance.yml 
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool apply -f rebalance.yml
kafkarebalance.kafka.strimzi.io/rebalance-1 created
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
Name:         rebalance-1
Namespace:    dlee-kafkanodepool
Labels:       strimzi.io/cluster=my-cluster
Annotations:  <none>
API Version:  kafka.strimzi.io/v1beta2
Kind:         KafkaRebalance
Metadata:
  Creation Timestamp:  2025-01-06T13:47:01Z
  Generation:          1
  Resource Version:    1618014
  UID:                 633ec846-1690-457a-bf8b-06efa8126a67
Spec:
  Brokers:
    3
  Mode:  add-brokers
Status:
  Conditions:
    Last Transition Time:  2025-01-06T13:47:02.748156001Z
    Status:                True
    Type:                  ProposalReady
  Observed Generation:     1
  Optimization Result:
    After Before Load Config Map:  rebalance-1
    Data To Move MB:               2
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 0
    Num Replica Movements:                88
    On Demand Balancedness Score After:   83.27837138237119
    On Demand Balancedness Score Before:  79.37642559153187
    Provision Recommendation:             
    Provision Status:                     RIGHT_SIZED
    Recent Windows:                       1
  Session Id:                             5d1c9048-b09b-420d-a8ce-f68452e516d7
Events:                                   <none>
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool annotate kafkarebalances.kafka.strimzi.io rebalance-1 strimzi.io/rebalance=approve
kafkarebalance.kafka.strimzi.io/rebalance-1 annotate
[root@ecs-m-01 ~]# 
[root@ecs-m-01 ~]# 
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get pods
NAME                                          READY   STATUS    RESTARTS      AGE
jupyter-notebook                              1/1     Running   0             7h14m
kafka-cluster-1-zookeeper-0                   1/1     Running   0             3h44m
kafka-cluster-1-zookeeper-1                   1/1     Running   0             3h44m
kafka-cluster-1-zookeeper-2                   1/1     Running   0             3h44m
my-cluster-cruise-control-6469cd5c74-82hlm    1/1     Running   0             108s
my-cluster-entity-operator-785876bcff-bt6nn   2/2     Running   8 (28h ago)   34h
my-cluster-nodepool-1-0                       1/1     Running   0             3h48m
my-cluster-nodepool-1-1                       1/1     Running   0             3h48m
my-cluster-nodepool-1-2                       1/1     Running   0             3h48m
my-cluster-nodepool-1-3                       1/1     Running   0             2m30s
my-cluster-zookeeper-0                        1/1     Running   0             28h
my-cluster-zookeeper-1                        1/1     Running   0             28h
my-cluster-zookeeper-2                        1/1     Running   0             28h
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
Name:         rebalance-1
Namespace:    dlee-kafkanodepool
Labels:       strimzi.io/cluster=my-cluster
Annotations:  <none>
API Version:  kafka.strimzi.io/v1beta2
Kind:         KafkaRebalance
Metadata:
  Creation Timestamp:  2025-01-06T13:47:01Z
  Generation:          1
  Resource Version:    1618275
  UID:                 633ec846-1690-457a-bf8b-06efa8126a67
Spec:
  Brokers:
    3
  Mode:  add-brokers
Status:
  Conditions:
    Last Transition Time:  2025-01-06T13:47:25.855854482Z
    Status:                True
    Type:                  Rebalancing
  Observed Generation:     1
  Optimization Result:
    After Before Load Config Map:  rebalance-1
    Data To Move MB:               2
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 0
    Num Replica Movements:                88
    On Demand Balancedness Score After:   83.27837138237119
    On Demand Balancedness Score Before:  79.37642559153187
    Provision Recommendation:             
    Provision Status:                     RIGHT_SIZED
    Recent Windows:                       1
  Session Id:                             8a156947-8353-474d-9aa3-09b24dbb8d62
Events:                                   <none>
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get kafkarebalance
NAME                 CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1          my-cluster                                     True                             
rebalance-ktopic-1   my-cluster                                                   True               
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool delete kafkarebalance  rebalance-ktopic-1 
kafkarebalance.kafka.strimzi.io "rebalance-ktopic-1" deleted
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get kafkarebalance
NAME          CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1   my-cluster                                     True                             
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get kafkarebalance
NAME          CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1   my-cluster                                     True                             
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get kafkarebalance
NAME          CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1   my-cluster                                     True                             
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
Name:         rebalance-1
Namespace:    dlee-kafkanodepool
Labels:       strimzi.io/cluster=my-cluster
Annotations:  <none>
API Version:  kafka.strimzi.io/v1beta2
Kind:         KafkaRebalance
Metadata:
  Creation Timestamp:  2025-01-06T13:47:01Z
  Generation:          1
  Resource Version:    1618275
  UID:                 633ec846-1690-457a-bf8b-06efa8126a67
Spec:
  Brokers:
    3
  Mode:  add-brokers
Status:
  Conditions:
    Last Transition Time:  2025-01-06T13:47:25.855854482Z
    Status:                True
    Type:                  Rebalancing
  Observed Generation:     1
  Optimization Result:
    After Before Load Config Map:  rebalance-1
    Data To Move MB:               2
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 0
    Num Replica Movements:                88
    On Demand Balancedness Score After:   83.27837138237119
    On Demand Balancedness Score Before:  79.37642559153187
    Provision Recommendation:             
    Provision Status:                     RIGHT_SIZED
    Recent Windows:                       1
  Session Id:                             8a156947-8353-474d-9aa3-09b24dbb8d62
Events:                                   <none>
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
Name:         rebalance-1
Namespace:    dlee-kafkanodepool
Labels:       strimzi.io/cluster=my-cluster
Annotations:  <none>
API Version:  kafka.strimzi.io/v1beta2
Kind:         KafkaRebalance
Metadata:
  Creation Timestamp:  2025-01-06T13:47:01Z
  Generation:          1
  Resource Version:    1618275
  UID:                 633ec846-1690-457a-bf8b-06efa8126a67
Spec:
  Brokers:
    3
  Mode:  add-brokers
Status:
  Conditions:
    Last Transition Time:  2025-01-06T13:47:25.855854482Z
    Status:                True
    Type:                  Rebalancing
  Observed Generation:     1
  Optimization Result:
    After Before Load Config Map:  rebalance-1
    Data To Move MB:               2
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 0
    Num Replica Movements:                88
    On Demand Balancedness Score After:   83.27837138237119
    On Demand Balancedness Score Before:  79.37642559153187
    Provision Recommendation:             
    Provision Status:                     RIGHT_SIZED
    Recent Windows:                       1
  Session Id:                             8a156947-8353-474d-9aa3-09b24dbb8d62
Events:                                   <none>
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
Name:         rebalance-1
Namespace:    dlee-kafkanodepool
Labels:       strimzi.io/cluster=my-cluster
Annotations:  <none>
API Version:  kafka.strimzi.io/v1beta2
Kind:         KafkaRebalance
Metadata:
  Creation Timestamp:  2025-01-06T13:47:01Z
  Generation:          1
  Resource Version:    1618275
  UID:                 633ec846-1690-457a-bf8b-06efa8126a67
Spec:
  Brokers:
    3
  Mode:  add-brokers
Status:
  Conditions:
    Last Transition Time:  2025-01-06T13:47:25.855854482Z
    Status:                True
    Type:                  Rebalancing
  Observed Generation:     1
  Optimization Result:
    After Before Load Config Map:  rebalance-1
    Data To Move MB:               2
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 0
    Num Replica Movements:                88
    On Demand Balancedness Score After:   83.27837138237119
    On Demand Balancedness Score Before:  79.37642559153187
    Provision Recommendation:             
    Provision Status:                     RIGHT_SIZED
    Recent Windows:                       1
  Session Id:                             8a156947-8353-474d-9aa3-09b24dbb8d62
Events:                                   <none>
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
Name:         rebalance-1
Namespace:    dlee-kafkanodepool
Labels:       strimzi.io/cluster=my-cluster
Annotations:  <none>
API Version:  kafka.strimzi.io/v1beta2
Kind:         KafkaRebalance
Metadata:
  Creation Timestamp:  2025-01-06T13:47:01Z
  Generation:          1
  Resource Version:    1620876
  UID:                 633ec846-1690-457a-bf8b-06efa8126a67
Spec:
  Brokers:
    3
  Mode:  add-brokers
Status:
  Conditions:
    Last Transition Time:  2025-01-06T13:50:56.207991439Z
    Status:                True
    Type:                  Ready
  Observed Generation:     1
  Optimization Result:
    After Before Load Config Map:  rebalance-1
    Data To Move MB:               2
    Excluded Brokers For Leadership:
    Excluded Brokers For Replica Move:
    Excluded Topics:
    Intra Broker Data To Move MB:         0
    Monitored Partitions Percentage:      100
    Num Intra Broker Replica Movements:   0
    Num Leader Movements:                 0
    Num Replica Movements:                88
    On Demand Balancedness Score After:   83.27837138237119
    On Demand Balancedness Score Before:  79.37642559153187
    Provision Recommendation:             
    Provision Status:                     RIGHT_SIZED
    Recent Windows:                       1
Events:                                   <none>
[root@ecs-m-01 ~]# oc -n dlee-kafkanodepool get kafkarebalance
NAME          CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1   my-cluster                                                   True               

```



