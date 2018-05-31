---
layout: docs
title:  Using the IBM Cloud Message Hub service with IBM Streams Runner for Apache Beam
navtitle: Using IBM Cloud Message Hub
description:  You can use Beam native KafkaIO to produce/consume IBM Cloud Message Hub messages.
weight:  10
published: true
tag: beam
prev:
  file: beamrunner-6b-objstor
  title: Using IBM Cloud Object Storage
next:
  file: beamrunner-7-issues
  title: Limitations and known issues
---


```java
Pipeline wp = Pipeline.create(options);

wp.apply(GenerateSequence.from(0)
        .withRate(10, Duration.standardSeconds(1)))
  .apply(ParDo.of(new ToKV()))
  .apply(KafkaIO.<String, String>write()
          .withBootstrapServers(bootstrapServers)
          .withTopic(options.getTopic())
          .updateProducerProperties(configs)
          .withKeySerializer(StringSerializer.class)
          .withValueSerializer(StringSerializer.class));
```

```java
Pipeline rp = Pipeline.create(options);

rp.apply(KafkaIO.<String, String>read()
        .withBootstrapServers(bootstrapServers)
        .withTopic(options.getTopic())
        .updateConsumerProperties(configs)
        .withKeyDeserializer(StringDeserializer.class)
        .withValueDeserializer(StringDeserializer.class)
        .withoutMetadata())
        .apply(ParDo.of(new FromKV()));
```

### Connecting to Bluemix MessageHub Using Kafka Client 0.10.2 or Later

1. In the application `pom.xml`, exclude the default Beam Kafka Client, and add Kafka Client of a
preferred version.

    ```xml
    <dependency>
        <groupId>org.apache.beam</groupId>
        <artifactId>beam-sdks-java-io-kafka</artifactId>
        <version>2.0.0</version>
        <exclusions>
            <exclusion>
                <groupId>org.apache.kafka</groupId>
                <artifactId>kafka-clients</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>0.11.0.1</version>
    </dependency>
    ```

2. Copy MessageHub VCAP JSON, and paste it to a file named `mh.vcap`.

3. Navigate to MessageHub service `Manage` tab, and create a topic with name `your-topic`.

4. Use the following command to submit the producer:

    ```bash
    $ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:$STREAMS_INSTALL/lib/com.ibm.streams.operator.samples.jar:`pwd`/kafka/target/kafka-1.0-SNAPSHOT.jar \
      com.ibm.streams.beam.io.MessageHubProduceConsume \
      --topic="your-topic" \
      --runner=StreamsRunner \
      --jarsToStage=`pwd`/kafka/target/kafka-1.0-SNAPSHOT.jar \
      --contextType=STREAMING_ANALYTICS_SERVICE \
      --serviceName=beam-service \
      --vcapServices=/home/streamsadmin/Project/vcap/bluemix.vcap \
      --vcap=mh.vcap \
      --isProducer
    ```

4. Use the following command to submit the consumer:

    ```bash
    $ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:$STREAMS_INSTALL/lib/com.ibm.streams.operator.samples.jar:`pwd`/kafka/target/kafka-1.0-SNAPSHOT.jar \
      com.ibm.streams.beam.io.MessageHubProduceConsume \
      --topic="your-topic" \
      --runner=StreamsRunner \
      --jarsToStage=`pwd`/kafka/target/kafka-1.0-SNAPSHOT.jar \
      --contextType=STREAMING_ANALYTICS_SERVICE \
      --serviceName=beam-service \
      --vcapServices=/home/streamsadmin/Project/vcap/bluemix.vcap \
      --vcap=mh.vcap \
      --isProducer=false
    ```


### Connecting to Bluemix MessageHub Using Kafka Client 0.10.1 or Earlier

[Beam 2.0 KafkaIO](https://github.com/apache/beam/blob/release-2.0.0/sdks/java/io/kafka/pom.xml#L33)
uses Kafka Client `0.9.0.1`, which does not yet provide the `PLAIN` SASL mechanism as required by the latest
[Messagehub tutorial](https://console.bluemix.net/docs/services/MessageHub/messagehub111.html#kafka_streams).
Besides, this version of Kafka Client requires the `java.security.auth.login.config` system
property point to a JAAS configuration file. It works fine in EMBEDDED mode. But it will be
difficult to configure when the pipeline is submitted to a cluster/cloud-service. If you would
like to use a more recent Kafka Client, or need to submit to cloud, please refer to
the next section. The following steps show how to run Beam pipeline with default Kafka Client in
Beam-2.0 in EMBEDDED mode.


1. Download MessageHub library which contains the required login modle:

    ```bash
    $ wget https://github.com/ibm-messaging/message-hub-samples/raw/master/kafka-0.9/message-hub-login-library/messagehub.login-1.0.0.jar
    ```

2. Prepare JAAS configuration file. You can get the `username` and `password` from the MessageHub
VCAP file. The `serviceName` should match your MessageHub service name in Bluemix.

    ```
    KafkaClient {
    com.ibm.messagehub.login.MessageHubLoginModule required
    serviceName="your service name"
    username="your username in the vcap"
    password="your password in the vcap";
    };
    ```

3. Navigate to MessageHub service `Manage` tab, and create a topic with name `your-topic`.

4. Start the producer using the following command. Make sure the path to `messagehub.login-1.0.0.jar`
in `-cp` is correct:

    ```bash
    $ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:$STREAMS_INSTALL/lib/com.ibm.streams.operator.samples.jar:`pwd`/kafka/target/kafka-1.0-SNAPSHOT.jar:messagehub.login-1.0.0.jar \
      -Djava.security.auth.login.config=`pwd`/jaas.conf \
      com.ibm.streams.beam.io.MessageHubProduceConsume \
      --topic="your-topic" \
      --runner=StreamsRunner \
      --contextType=EMBEDDED \
      --vcap=mh.vcap \
      --isProducer
    ```

5. Start the consumer using the following command.

    ```bash
    $ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:$STREAMS_INSTALL/lib/com.ibm.streams.operator.samples.jar:`pwd`/kafka/target/kafka-1.0-SNAPSHOT.jar:messagehub.login-1.0.0.jar \
      -Djava.security.auth.login.config=`pwd`/jaas.conf \
      com.ibm.streams.beam.io.MessageHubProduceConsume \
      --topic="your-topic" \
      --runner=StreamsRunner \
      --contextType=EMBEDDED \
      --vcap=mh.vcap \
      --isProducer=false
    ```
