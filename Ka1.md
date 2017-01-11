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
You’ll notice that we set the KAFKA_ADVERTISED_LISTENERS variable to localhost:29092.
This will make Kafka accessible from outside the container by advertising it’s location on the Docker host.
```

Let’s check the logs to see the broker has booted up successfully:

```
docker logs kafka
```

You should see the following at the end of the log output:

```bash
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
Created topic "foo".
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
  {"f1": "value2
  n
  {"f1": "value3"}
  ```

### Note!
```
If you hit `Enter` with an empty line, it will be interpreted as a null value and cause an error. You can simply start the console producer again to continue sending messages.
```
When you’re done, use `Ctrl+C` to shut down the process. You can also type `exit` to leave the container. Now that we wrote avro data to Kafka, we should check that the data was actually produced as expected by trying to consume it. Although the Schema Registry also ships with a built-in console consumer utility, we’ll instead demonstrate how to read it from outside the container on our local machine via the REST Proxy. The REST Proxy depends on the Schema Registry when producing/consuming avro data, so let’s leave the container running as we head to the next step.


#REST Proxy
Consume data via the REST Proxy.

  First, start up the REST Proxy:

    ```
    docker run -d \
        --net=host \
        --name=kafka-rest \
        -e KAFKA_REST_ZOOKEEPER_CONNECT=localhost:32181 \
        -e KAFKA_REST_LISTENERS=http://localhost:8082 \
        -e KAFKA_REST_SCHEMA_REGISTRY_URL=http://localhost:8081 \
        -e KAFKA_REST_HOST_NAME=localhost \
        confluentinc/cp-kafka-rest:3.1.1
    ```

  For the next two steps, we’re going to use CURL commands to talk to the REST Proxy. For the sake of simplicity, the Schema Registry and REST Proxy containers on same host with the REST Proxy listening at http://localhost:8082.

  ```
  docker run -it --net=host --rm confluentinc/cp-schema-registry:3.1.1 bash
  ```
  Next, we’ll need to create a consumer for Avro data, starting at the beginning of the log for our topic, `bar`.

  ```
  curl -X POST -H "Content-Type: application/vnd.kafka.v1+json" \
  --data '{"name": "my_consumer_instance", "format": "avro", "auto.offset.reset": "smallest"}' \
  http://localhost:8082/consumers/my_avro_consumer
  ```

  You should see the following in your terminal window:

  ```
  {"instance_id":"my_consumer_instance","base_uri":"http://localhost:8082/consumers/my_avro_consumer/instances/my_consumer_instance"}
  ```
  Now we’ll consume some data from a topic. It will be decoded, translated to JSON, and included in the response. The schema used for deserialization is fetched automatically from the Schema Registry, which we told the REST Proxy how to find by setting the `KAFKA_REST_SCHEMA_REGISTRY_URL` variable on startup.

  ```bash
  curl -X GET -H "Accept: application/vnd.kafka.avro.v1+json" \
  http://localhost:8082/consumers/my_avro_consumer/instances/my_consumer_instance/topics/bar
  ```

  You should see the following output:

  ``` bash
  [{"key":null,"value":{"f1":"value1"},"partition":0,"offset":0},{"key":null,"value":{"f1":"value2"},"partition":0,"offset":1},{"key":null,"value":{"f1":"value3"},"partition":0,"offset":2}]
  ```

#Confluent Control Center

### Stream Monitoring
We will walk you through how to run Confluent Control Center with console producers and consumers and monitor consumption and latency.

  First, let’s launch Confluent Control Center. We already have ZooKeeper and Kafka up and running from the steps above. Let’s make a directory on the host for Control Center data. If you are running Docker Machine then you will need to SSH into the VM to run these commands by running `docker-machine ssh <your machine name>` and run the command as root.

  ```
  mkdir -p /tmp/control-center/data
  ```


  Now we start Control Center and bind it’s data directory to the directory we just created and bind it’s HTTP interface to port 9021.

  ```bash
    docker run -d \
      --name=control-center \
      --net=host \
      --ulimit nofile=16384:16384 \
      -p 9021:9021 \
      -v /tmp/control-center/data:/var/lib/confluent-control-center \
      -e CONTROL_CENTER_ZOOKEEPER_CONNECT=localhost:32181 \
      -e CONTROL_CENTER_BOOTSTRAP_SERVERS=localhost:29092 \
      -e CONTROL_CENTER_REPLICATION_FACTOR=1 \
      -e CONTROL_CENTER_MONITORING_INTERCEPTOR_TOPIC_PARTITIONS=1 \
      -e CONTROL_CENTER_INTERNAL_TOPICS_PARTITIONS=1 \
      -e CONTROL_CENTER_STREAMS_NUM_STREAM_THREADS=2 \
      -e CONTROL_CENTER_CONNECT_CLUSTER=http://localhost:28082 \
      confluentinc/cp-enterprise-control-center:3.1.1
  ```

  Control Center will create the topics it needs in Kafka. Check that it started correctly by searching it’s logs with the following command:

  ```
  docker logs control-center | grep Started
  ```

  You should see the following

  ```
  [2016-08-26 18:47:26,809] INFO Started NetworkTrafficServerConnector@26d96e5{HTTP/1.1}{0.0.0.0:9021} (org.eclipse.jetty.server.NetworkTrafficServerConnector)
  [2016-08-26 18:47:26,811] INFO Started @5211ms (org.eclipse.jetty.server.Server)
  ```


  To see the Control Center UI, navigate in a browser using HTTP to port 9021 of the docker host. If you’re using docker-machine, you can get your host IP by running `docker-machine ip <your machine name>`. If your docker daemon is running on a remote machine (such as an AWS EC2 instance), you’ll need to open port 9021 to allow outside TCP access. In AWS, you do this by adding a “Custom TCP Rule” to the security group for port 9021 from any source IP.

  Initially, the Stream Monitoring UI will have no data.

  ![Signup](/img/c3-quickstart-init.png)

  Next, we’ll run the console producer and consumer with monitoring interceptors configured and see the data in Control Center. First we need to create a topic for testing.


  ```
  docker run \
  --net=host \
  --rm confluentinc/cp-kafka:3.1.1 \
  kafka-topics --create --topic c3-test --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
  ```

  Now use the console producer with the monitoring interceptor enabled to send data

  ```bash
  while true;
  do
    docker run \
      --net=host \
      --rm \
      -e CLASSPATH=/usr/share/java/monitoring-interceptors/monitoring-interceptors-3.1.1.jar \
      confluentinc/cp-kafka-connect:3.1.1 \
      bash -c 'seq 10000 | kafka-console-producer --request-required-acks 1 --broker-list localhost:29092 --topic c3-test --producer-property interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor --producer-property acks=1 && echo "Produced 10000 messages."'
      sleep 10;
  done
  ```

  This command will use the built-in Kafka Console Producer to produce 10000 simple messages on a 10 second interval to the `c3-test` topic. Upon running it, you should see the following:

  ```bash
  Produced 10000 messages.
  ```

  Use the console consumer with the monitoring interceptor enabled to read the data.

  ```bash
  OFFSET=0
  while true;
  do
    docker run \
      --net=host \
      --rm \
      -e CLASSPATH=/usr/share/java/monitoring-interceptors/monitoring-interceptors-3.1.1.jar \
      confluentinc/cp-kafka-connect:3.1.1 \
      bash -c 'kafka-console-consumer --consumer-property group.id=qs-consumer --consumer-property interceptor.classes=io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor --new-consumer --bootstrap-server localhost:29092 --topic c3-test --offset '$OFFSET' --partition 0 --max-messages=1000'
    sleep 1;
    let OFFSET=OFFSET+1000
  done
  ```

  If everything is working as expected, each of the original messages we produced should be written back out:

  ```bash
  1
  ....
  1000
  Processed a total of 1000 messages
  ```

  We’ve intentionally setup a slow consumer to consume at a rate of 1000 messages per second. After 15 seconds have passed, you should see monitoring activity reflected in the Control Center UI. You will notice there will be moments where the bars are colored red to reflect the slow consumption of data.

  ![Signup](/img/c3-quickstart-monitoring-data.png)

###Alerts

Confluent Control Center provides alerting functionality when anomalous events occur in your monitoring data. This section assumes the console producer and consumer are still running in the background.

The Overview link the lefthand sidebar takes you to a page which displays a history of all triggered events. To begin receiving alerts on anomalous events in your monitoring data, we’ll need to create a trigger. Click the “Triggers” navigation item and then select “+ New trigger”.

Let’s configure a trigger to fire when the difference between our actual consumption and expected consumption is greater than 1000 messages:

[!Singup](/img/c3-quickstart-new-trigger-form.png)


Set the trigger name to be “Underconsumption”, which is what will be displayed on the history page when our trigger fires. We need to assign `qs-consumer`, which we created previously, to our trigger.

Set the trigger metric to be “Consumption difference” where the condition is “Greater than” 1000 messages. The buffer time (in seconds) is the wall clock time we will wait before firing the trigger to make sure the trigger condition is not too transient.

After saving the trigger, Control Center will now prompt us to associate an action that will fire when our newly created trigger occurs. For now, the only action is send an email. Select our new trigger and choose maximum send rate for your alert email.

[!Singup](/img/c3-quickstart-new-action-form.png)

Let’s return to our trigger history page. In a short while, you should see a new trigger show up in our alert history. This is because we setup our consumer to consume data at a slower rate than our producer.

[!Signup](/img/c3-quickstart-alerts-history)

#Kafka Connect

### Getting Started

Getting Started
We will walk you through an end-to-end data transfer pipeline using Kafka Connect. We’ll start by reading data from a file and writing that data to a new file. We will then extend the pipeline to show how to use connect to read from a database. This example is meant to be simple for the sake of this introductory tutorial. If you’d like a more in-depth example, please refer to (/our tutorial on using a JDBC connector with avro data).

First, let’s start up Kafka Connect. Connect stores config, status, and internal offsets for connectors in Kafka topics. We will create these topics now. We already have Kafka up and running from the steps above.

  ```
  docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.1.1 \
  kafka-topics --create --topic quickstart-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
  ```
  ```
  docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.1.1 \
  kafka-topics --create --topic quickstart-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
  ```
  ```
  docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.1.1 \
  kafka-topics --create --topic quickstart-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
  ```

###Note!
  ```
  It is possible to allow connect to auto-create these topics by enabling the autocreation setting. However, we recommend doing it manually, as these topics are important for connect to function and you’ll likely want to control settings such as replication factor and number of partitions.
  ```
Next, we’ll create a topic for storing data that we’re going to be sending to Kafka for this tutorial.

  ```
  docker run \
  --net=host \
  --rm \
  confluentinc/cp-kafka:3.1.1 \
  kafka-topics --create --topic quickstart-data --partitions 1 --replication-factor 1 --if-not-exists --zookeeper localhost:32181
  ```

  Now you should verify that the topics are created before moving on:

  ```
   vdocker run \
   --net=host \
   --rm \
   confluentinc/cp-kafka:3.1.1 \
   kafka-topics --describe --zookeeper localhost:32181
   ```

For this example, we’ll create a FileSourceConnector, a FileSinkConnector and directories for storing the input and output files. If you are running Docker Machine then you will need to SSH into the VM to run these commands by running `docker-machine ssh <your machine name>`. You may also need to run the command as root.

   First, let’s create the directory where we’ll store the input and output data files:

   ```
   mkdir -p /tmp/quickstart/file
   ```
   Next, start a Connect worker in distributed mode:

   ```bash
   docker run -d \
    --name=kafka-connect \
    --net=host \
    -e CONNECT_PRODUCER_INTERCEPTOR_CLASSES=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor \
    -e CONNECT_CONSUMER_INTERCEPTOR_CLASSES=io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor \
    -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
    -e CONNECT_REST_PORT=28082 \
    -e CONNECT_GROUP_ID="quickstart" \
    -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-config" \
    -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-offsets" \
    -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-status" \
    -e CONNECT_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
    -e CONNECT_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
    -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
    -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
    -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
    -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
    -v /tmp/quickstart/file:/tmp/quickstart \
    confluentinc/cp-kafka-connect:3.1.1
    ```

    As you can see in the command above, we tell Connect to refer to the three topics we create in the first step of this Connect tutorial. Let’s check to make sure that the Connect worker is up by running the following command to search the logs:

    ```
    docker logs kafka-connect | grep started
    ```

    You should see the following

    ```
    [2016-08-25 18:25:19,665] INFO Herder started (org.apache.kafka.connect.runtime.distributed.DistributedHerder)
    [2016-08-25 18:25:19,676] INFO Kafka Connect started (org.apache.kafka.connect.runtime.Connect)
    ```

    We will now create our first connector for reading a file from disk. To do this, let’s start by creating a file with some data. Again, if you are running Docker Machine then you will need to SSH into the VM to run these commands by running `docker-machine ssh <your machine name>`. (You may also need to run the command as root).

    ```bash
    seq 1000 > /tmp/quickstart/file/input.txt
    ```
Now create the connector using the Kafka Connect REST API. (Note: Make sure you have `curl` installed!)

    Set the `CONNECT_HOST` environment variable. If you are running this on Docker Machine, then the hostname will need to be `docker-machine ip <your docker machine name>``. If you are running on a cloud provider like AWS, you will either need to have port `28082` open or you can SSH into the VM and run the following command:

    ```bash
    export CONNECT_HOST=localhost
    ```
    The next step is to create the File Source connector.

    ```bash
    curl -X POST \
      -H "Content-Type: application/json" \
      --data '{"name": "quickstart-file-source", "config": {"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector", "tasks.max":"1", "topic":"quickstart-data", "file": "/tmp/quickstart/input.txt"}}' \
      http://$CONNECT_HOST:28082/connectors
    ```
    Upon running the command, you should see the following output in your terminal window:

    ```bash
    {"name":"quickstart-file-source","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","tasks.max":"1","topic":"quickstart-data","file":"/tmp/quickstart/input.txt","name":"quickstart-file-source"},"tasks":[]}
    ```
    Before moving on, let’s check the status of the connector using curl as shown below:

    ```bash
    curl -X GET http://$CONNECT_HOST:28082/connectors/quickstart-file-source/status
    ```
    You should see the following output including the `state` of the connector as `RUNNING`:

    ```bash
    {"name":"quickstart-file-source","connector":{"state":"RUNNING","worker_id":"localhost:28082"},"tasks":[{"state":"RUNNING","id":0,"worker_id":"localhost:28082"}]}
    ```
  Now that the connector is up and running, let’s try reading a sample of 10 records from the `quickstart-data` topic to check if the connector is uploading data to Kafka, as expected.

    ```
    docker run \
      --net=host \
      --rm \
      confluentinc/cp-kafka:3.1.1 \
      kafka-console-consumer --bootstrap-server localhost:29092 --topic quickstart-data --new-consumer --from-beginning --max-messages 10
    ```
    You should see the following:

    ```bash
    {"schema":{"type":"string","optional":false},"payload":"1"}
    {"schema":{"type":"string","optional":false},"payload":"2"}
    {"schema":{"type":"string","optional":false},"payload":"3"}
    {"schema":{"type":"string","optional":false},"payload":"4"}
    {"schema":{"type":"string","optional":false},"payload":"5"}
    {"schema":{"type":"string","optional":false},"payload":"6"}
    {"schema":{"type":"string","optional":false},"payload":"7"}
    {"schema":{"type":"string","optional":false},"payload":"8"}
    {"schema":{"type":"string","optional":false},"payload":"9"}
    {"schema":{"type":"string","optional":false},"payload":"10"}
    Processed a total of 10 messages
    ```
    Success! We now have a functioning source connector! Now let’s bring balance to the universe by launching a File Sink to read from this topic and write to an output file. You can do so using the following command:

    ```bash
    curl -X POST -H "Content-Type: application/json" \
    --data '{"name": "quickstart-file-sink", "config": {"connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector", "tasks.max":"1", "topics":"quickstart-data", "file": "/tmp/quickstart/output.txt"}}' \
    http://$CONNECT_HOST:28082/connectors
    ```

    You should see the output below in your terminal window, confirming that the `quickstart-file-sink` connector has been created and will write to `/tmp/quickstart/output.txt`:

    ```bash
    {"name":"quickstart-file-sink","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector","tasks.max":"1","topics":"quickstart-data","file":"/tmp/quickstart/output.txt","name":"quickstart-file-sink"},"tasks":[]}
    ```
    As we did before, let’s check the status of the connector:

    ```bash
    curl -s -X GET http://$CONNECT_HOST:28082/connectors/quickstart-file-sink/status
    ```
    Finally, let’s check the file to see if the data is present. Once again, you will need to SSH into the VM if you are running Docker Machine.

    ```
    cat /tmp/quickstart/file/output.txt
    ```

    If everything worked as planned, you should see all of the data we originally wrote to the input file:
    ```
    1
    ...
    1000
    ```
### Monitoring in Control Center

  Next we’ll see how to monitor Kafka Connect in Control Center using the monitoring interceptors and the source and sink previously created.

    Check the Control Center UI and should see both the source and sink running in Kafka Connect.

    (/img/c3-quickstart-connect-view-src)

    /img/c3-quickstart-connect-view-sink

  You should start to see stream monitoring data from Kafka Connect in the Control Center UI from our previous commands.

    [!Signup](/img/c3-quickstart-connect-monitoring)


#Cleanup

  Once you’re done, cleaning up is simple. You can simply run `docker rm -f $(docker ps -a -q)` to delete all the containers we created in the steps above. Because we allowed Kafka and Zookeeper to store data on their respective containers, there are no additional volumes to clean up. If you also want to remove the Docker machine you used, you can do so using `docker-machine rm <your machine name>`.
