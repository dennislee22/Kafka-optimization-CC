# Kafka Optimization


/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --group dlee-group1

Consumer group 'dlee-group1' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
dlee-group1     dlee-topic1     1          304             1280            976             -               -               -
dlee-group1     dlee-topic1     2          321             3516            3195            -               -               -
dlee-group1     dlee-topic1     0          323             2410            2087            -               -               -
