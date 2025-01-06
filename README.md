# Kafka Optimization

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

