# Kafka Optimization


```
# NAMESPACE=dlee-kafkanodepool
# IMAGE=$(kubectl get pod my-cluster-first-pool-0  -n dlee-kafkanodepool --output jsonpath='{.spec.containers[0].image}')
# CLUSTER=my-cluster

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
    #time.sleep(0.1)  # Simulate a steady stream of messages
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



<img width="1432" alt="image" src="https://github.com/user-attachments/assets/d49009da-2558-4b10-b35b-216569e7ee47" />

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

```
# kubectl -n dlee-kafkanodepool scale kafkanodepool nodepool-1 --replicas=4
kafkanodepool.kafka.strimzi.io/nodepool-1 scaled

# kubectl -n dlee-kafkanodepool get pods
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
my-cluster-nodepool-1-3                       1/1     Running    0             1m
my-cluster-zookeeper-0                        1/1     Running    0             28h
my-cluster-zookeeper-1                        1/1     Running    0             28h
my-cluster-zookeeper-2                        1/1     Running    0             28h

# kubectl -n dlee-kafkanodepool apply -f rebalance.yml
kafkarebalance.kafka.strimzi.io/rebalance-1 created

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


# kubectl -n dlee-kafkanodepool annotate kafkarebalances.kafka.strimzi.io rebalance-1 strimzi.io/rebalance=approve
kafkarebalance.kafka.strimzi.io/rebalance-1 annotate

# kubectl -n dlee-kafkanodepool get pods
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

<img width="1431" alt="image" src="https://github.com/user-attachments/assets/58da1206-6a33-4d38-824a-fc3b915227e0" />


