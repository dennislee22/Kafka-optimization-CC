# Kafka Optimization w/ Cruise Control

<img width="440" alt="image" src="https://github.com/user-attachments/assets/cfba1b73-fe26-4484-8d88-b4c9eb52d502" />

Kafka optimization using Cruise Control allows for intelligent resource balancing within a Kafka cluster, ensuring optimal disk, CPU, and network utilization. Cruise Control dynamically reallocates partitions across brokers to avoid hotspots, mitigate under-replication, and improve overall cluster performance. By continuously monitoring system metrics, it makes real-time adjustments, enabling more efficient message distribution, reducing consumer lag, and enhancing fault tolerance.
In this article, I will demonstrate how Cruise Control can help readjust the underlying Kafka infrastructure to ensure the platform remains efficient and scalable.

# KafkaNodePool Cluster

1. A `KafkaNodePool` with a Kafka cluster has already been deployed as shown below.
```
# kubectl -n dlee-kafkanodepool get pods
NAME                                          READY   STATUS    RESTARTS      AGE
jupyter-notebook                              1/1     Running   0             20h
my-cluster-cruise-control-6469cd5c74-82hlm    1/1     Running   0             12h
my-cluster-entity-operator-785876bcff-bt6nn   2/2     Running   0             34h
my-cluster-nodepool-1-0                       1/1     Running   0             16h
my-cluster-nodepool-1-1                       1/1     Running   0             16h
my-cluster-nodepool-1-2                       1/1     Running   0             8m21s
my-cluster-nodepool-1-3                       1/1     Running   0             12h
my-cluster-zookeeper-0                        1/1     Running   0             40h
my-cluster-zookeeper-1                        1/1     Running   0             40h
my-cluster-zookeeper-2                        1/1     Running   0             40h
```

2. Create a topic with 3 partitions and 2 replicas by running the embedded `kafka-topics.sh` tool inside one of the Kafka broker pods. 
```
# NAMESPACE=dlee-kafkanodepool
# IMAGE=$(kubectl get pod my-cluster-nodepool-1-0  -n dlee-kafkanodepool --output jsonpath='{.spec.containers[0].image}')
# CLUSTER=my-cluster

# kubectl run kafka-admin -it -n $NAMESPACE --image=$IMAGE --rm=true --restart=Never --command -- /bin/sh

$ /opt/kafka/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --create --topic ktopic-1 --partitions 3 --replication-factor 2
```

3. Verify that the topic has been created successfully with the specific partitions and replicas.
```
$ /opt/kafka/bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic ktopic-1
Topic: ktopic-1	TopicId: J5paBCt6Tz2ugY9T38B14Q	PartitionCount: 3	ReplicationFactor: 2	Configs: min.insync.replicas=2,message.format.version=3.0-IV1
	Topic: ktopic-1	Partition: 0	Leader: 0	Replicas: 0,2	Isr: 0,2	Elr: N/A	LastKnownElr: N/A
	Topic: ktopic-1	Partition: 1	Leader: 2	Replicas: 2,1	Isr: 2,1	Elr: N/A	LastKnownElr: N/A
	Topic: ktopic-1	Partition: 2	Leader: 1	Replicas: 1,0	Isr: 1,0	Elr: N/A	LastKnownElr: N/A
```

4. Create a consumer group.
```
$ /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic ktopic-1 --group cgroup-1 --from-beginning
^CProcessed a total of 0 messages
```

5. Verify that the consumer group has successfully been created.
```
$ /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --list
cgroup-1
```

6. By default, Kafka producers assign messages to partitions based on the message key (which is hashed) or using round-robin assignment. In this scenario, I will simulate uneven message distribution across Kafka partitions. To achieve this, I will deploy a JupyterLab pod within the same Kafka namespace and use the following Python code to produce messages in a way that results in some partitions receiving more messages than others.
 
```
from kafka import KafkaProducer
import time
import json

# Define the Kafka broker and topic
kafka_broker = 'my-cluster-kafka-bootstrap.dlee-kafkanodepool.svc.cluster.local:9092'
topic_name = 'ktopic-1'

# Create a KafkaProducer
producer = KafkaProducer(
    bootstrap_servers=kafka_broker,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Simulating uneven message distribution by manually choosing partitions
partition_0 = 2
partition_1 = 1
partition_2 = 1

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

7. As a result, the total offsets in each partition are uneven. The following output shows the current offsets, log end offsets, and lag for each partition, indicating that partitions 1 and 2 have messages lagging, while partition 0 has no messages consumed.
```
$ /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --group cgroup-1

Consumer group 'cgroup-1' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
cgroup-1        ktopic-1        2          0               18123           18123           -               -               -
cgroup-1        ktopic-1        1          0               36246           36246           -               -               -
cgroup-1        ktopic-1        0          0               0               0               -               -               -
```

8. The Grafana dashboard shows that PVC storage utilization across broker pods is uneven.
<img width="1432" alt="image" src="https://github.com/user-attachments/assets/d49009da-2558-4b10-b35b-216569e7ee47" />

9. Now, let's rebalance the storage utilization across broker pods. Scale out a broker pod `my-cluster-nodepool-1-3`.
```
# kubectl -n dlee-kafkanodepool scale kafkanodepool nodepool-1 --replicas=4
kafkanodepool.kafka.strimzi.io/nodepool-1 scaled

# kubectl -n dlee-kafkanodepool get pods
NAME                                          READY   STATUS     RESTARTS      AGE
jupyter-notebook                              1/1     Running    0             7h12m
my-cluster-cruise-control-7f9876696f-hfj2h    1/1     Running    0             131m
my-cluster-entity-operator-785876bcff-bt6nn   2/2     Running    0             34h
my-cluster-nodepool-1-0                       1/1     Running    0             3h45m
my-cluster-nodepool-1-1                       1/1     Running    0             3h45m
my-cluster-nodepool-1-2                       1/1     Running    0             3h45m
my-cluster-nodepool-1-3                       1/1     Running    0             1m
my-cluster-zookeeper-0                        1/1     Running    0             28h
my-cluster-zookeeper-1                        1/1     Running    0             28h
my-cluster-zookeeper-2                        1/1     Running    0             28h
```

10. Firstly, prepare the following `KafkaRebalance` in YAML format.
```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: rebalance-1
  namespace: dlee-kafkanodepool
  labels:
    strimzi.io/cluster: my-cluster
spec:
  mode: add-brokers
  brokers: [3]
```

11. Apply the above `KafkaRebalance` file.
```
# kubectl -n dlee-kafkanodepool apply -f rebalance.yml
kafkarebalance.kafka.strimzi.io/rebalance-1 created
```

12. Note that `rebalance-1` is in `ProposalReady` status.
```
# kubectl -n dlee-kafkanodepool describe kafkarebalance rebalance-1 
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
```

13. Approve the proposal to start the rebalancing process.
```
# kubectl -n dlee-kafkanodepool annotate kafkarebalances.kafka.strimzi.io rebalance-1 strimzi.io/rebalance=approve
kafkarebalance.kafka.strimzi.io/rebalance-1 annotate
```

14. The rebalancing process is now progressing and will complete in due course.
```
# kubectl -n dlee-kafkanodepool get kafkarebalance
NAME                 CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1          my-cluster                                     True                             
             
# kubectl -n dlee-kafkanodepool get kafkarebalance
NAME          CLUSTER      PENDINGPROPOSAL   PROPOSALREADY   REBALANCING   READY   NOTREADY   STOPPED
rebalance-1   my-cluster                                                   True   

# kubectl oc -n dlee-kafkanodepool describe kafkarebalance  rebalance-1 
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
```

15. Upon successful rebalancing, note that the PVC storage utilization is now uniform across all Kafka broker stateful pods.
<img width="1431" alt="image" src="https://github.com/user-attachments/assets/58da1206-6a33-4d38-824a-fc3b915227e0" />


