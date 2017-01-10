# Kafka

### Start Kafka

```
docker run -d \
    --net=host \
    --name=kafka \
    -e KAFKA_ZOOKEEPER_CONNECT=localhost:32181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:29092 \
    confluentinc/cp-kafka:3.1.1
```

### Note!
```
You’ll notice that we set the KAFKA_ADVERTISED_LISTENERS variable to localhost:29092. This will make Kafka accessible from outside the container by advertising it’s location on the Docker host.
```

Let’s check the logs to see the broker has booted up successfully:

```
docker logs kafka
```

You should see the following at the end of the log output:

```
....
[2016-07-15 23:31:00,295] INFO [Kafka Server 1], started (kafka.server.KafkaServer)
[2016-07-15 23:31:00,295] INFO [Kafka Server 1], started (kafka.server.KafkaServer)
...
...
[2016-07-15 23:31:00,349] INFO [Controller 1]: New broker startup callback for 1 (kafka.controller.KafkaController)
[2016-07-15 23:31:00,349] INFO [Controller 1]: New broker startup callback for 1 (kafka.controller.KafkaController)
[2016-07-15 23:31:00,350] INFO [Controller-1-to-broker-1-send-thread], Starting  (kafka.controller.RequestSendThread)
...

```

Take it for a test drive. Test that the broker is functioning as expected by creating a topic and producing data to it:

First, we’ll create a topic. We’ll name it `foo` and keep things simple by just giving it one partition and only one replica. You’ll likely want to increase both if you’re running in a more high-stakes environment in which you are concerned about data loss.

```
docker run \
--net=host \
--rm confluentinc/cp-kafka:3.1.1 \
kafka-topics --create --topic foo --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
```

You should see the following output in your terminal window:
```
Created topic `"foo"`.
```

Before moving on, verify that the topic was created successfully:

```
docker run \
--net=host \
--rm confluentinc/cp-kafka:3.1.1 \
kafka-topics --describe --topic foo --zookeeper localhost:32181
```

You should see the following output in your terminal window:

```
Topic:foo   PartitionCount:1    ReplicationFactor:1 Configs:
Topic: foo  Partition: 0    Leader: 1001    Replicas: 1001  Isr: 1001
```

Next, we’ll try generating some data to our new topic:

  ```
  docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.1.1 \
  bash -c "seq 42 | kafka-console-producer --request-required-acks 1 --broker-list localhost:29092 --topic foo && echo 'Produced 42 messages.'"
  ```

This command will use the built-in Kafka Console Producer to produce 42 simple messages to the topic. Upon running it, you should see the following:

```
Produced 42 messages.
```
To complete the story, let’s read back the message using the built-in Console consumer:

```
docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.1.1 \
  kafka-console-consumer --bootstrap-server localhost:29092 --topic foo --new-consumer --from-beginning --max-messages 42
  ```

  If everything is working as expected, each of the original messages we produced should be written back out:

```
  1
  ....
  42
  Processed a total of 42 messages
```

#Schema Registry

Now we have all Kafka and Zookeeper up and running, we can start trying out some of the other components included in Confluent Platform. We’ll start by using the Schema Registry to create a new schema and send some Avro data to a Kafka topic. Although you would normally do this from one of your applications, we’ll use a utility provided with Schema Registry to send the data without having to write any code.

  First, let’s fire up the Schema Registry container:

  ```
  docker run -d \
  --net=host \
  --name=schema-registry \
  -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=localhost:32181 \
  -e SCHEMA_REGISTRY_HOST_NAME=localhost \
  -e SCHEMA_REGISTRY_LISTENERS=http://localhost:8081 \
  confluentinc/cp-schema-registry:3.1.1
  ```

  As we did before, we can check that it started correctly by viewing the logs.

  ```
  docker logs schema-registry
  ```

  For the next two steps, we’re going to use CURL commands to talk to the Schema Registry. For the sake of simplicity, we’ll run a new Schema Registry container on the same host, where we’ll be using the `kafka-avro-console-producer` utility.

  ```
  docker run -it --net=host --rm confluentinc/cp-schema-registry:3.1.1 bash
  ```


  Direct the utility at the local Kafka cluster, tell it to write to the topic `bar`, read each line of input as an Avro message, validate the schema against the Schema Registry at the specified URL, and finally indicate the format of the data.

  ```
  /usr/bin/kafka-avro-console-producer \
  --broker-list localhost:29092 --topic bar \
  --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"}]}'
  ```

  Once started, the process will wait for you to enter messages, one per line, and will send them immediately when you hit the Enter key. Try entering a few messages:

  ```
  {"f1": "value1"}
  {"f1": "value2"}
  {"f1": "value3"}
  ```

### Note!
```
If you hit `Enter` with an empty line, it will be interpreted as a null value and cause an error. You can simply start the console producer again to continue sending messages.
```
When you’re done, use `Ctrl+C` to shut down the process. You can also type `exit` to leave the container. Now that we wrote avro data to Kafka, we should check that the data was actually produced as expected by trying to consume it. Although the Schema Registry also ships with a built-in console consumer utility, we’ll instead demonstrate how to read it from outside the container on our local machine via the REST Proxy. The REST Proxy depends on the Schema Registry when producing/consuming avro data, so let’s leave the container running as we head to the next step.
