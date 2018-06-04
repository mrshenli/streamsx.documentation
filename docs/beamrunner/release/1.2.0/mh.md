---
layout: docs
title:  Using the IBM Cloud Message Hub service with IBM Streams Runner for Apache Beam
navtitle: Using IBM Cloud Message Hub
description:  Beam applications can produce/consume messages to/from IBM Cloud Message Hub using Beam's native KafkaIO.
weight:  10
published: true
tag: beam-120
prev:
  file: objstor
  title: Using IBM Cloud Object Storage
next:
  file: issues
  title: Limitations and known issues
---

Beam applications can produce/consume messages to/from IBM Cloud Message Hub
using Beam's native [KafkaIO](https://beam.apache.org/documentation/sdks/javadoc/2.4.0/org/apache/beam/sdk/io/kafka/KafkaIO.html).
IBM Cloud Message Hub is a scalable, distributed, high throughput messaging
service that enables applications and services to communicate easily and
reliably. For more information, see [Getting started with Message Hub](https://console.bluemix.net/docs/services/MessageHub/index.html).

## Creating a Message Hub service on IBM Cloud

If you have not already done so, you must create a Message Hub service on IBM Cloud.

1. Navigate to IBM Cloud [Catalog page](https://console.bluemix.net/catalog/), and search for *Message Hub*.
2. Click the *Message Hub* service.
3. For *Pricing Plan*, choose standard.
4. Click Create. IBM Cloud returns to the manage page of the Message Hub service.
5. On the manage page, navigate to *Topic* tab, click the plus button (*create*), provide a topic name, and then click *Create Topic*. You will provide this topic name to the producer and consumer in subsequent steps.

## Setting up credentials for the service

To communicate with Message Hub from Beam applications, you must specify the
IBM CLoud service credentials.

1. From the Message Hub manage page, click *Service credentials* on the left navigation bar.
2. If necessary, create a credential by clicking New credential. Use the default information and click Add.
3. Click View credentials.
4. Copy the credentials JSON content to a file (_e.g._, mh.cred) for future uses.

## Running an example application

In the `pom.xml` file, add Beam's `KafkaIO` and `kafka-clients`
(version *0.10.2* or Later). This example also includes `json-simple`
as a dependency to parse the Message Hub credential file.

```xml
<dependency>
    <groupId>org.apache.beam</groupId>
    <artifactId>beam-sdks-java-io-kafka</artifactId>
    <version>2.4.0</version>
</dependency>

<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>0.11.0.1</version>
</dependency>

<dependency>
     <groupId>com.googlecode.json-simple</groupId>
     <artifactId>json-simple</artifactId>
     <version>1.1.1</version>
 </dependency>
```

The code snippet below shows a producer example that demonstrates how to setup
`bootstrapServers`, `topic`, and `producerProperties` using the information
provided in the credentials JSON.

```java
public class Producer {
    private static final String SERVERS = "kafka_brokers_sasl";
    private static final String BOOTSTRAP_SERVERS = "bootstrap.servers";
    private static final String JAAS_TEMPLATE =
            "org.apache.kafka.common.security.plain.PlainLoginModule required"
                  + " username=\"USERNAME\" password=\"PASSWORD\";";

    interface MessageHubOptions extends PipelineOptions, ApplicationNameOptions {
        void setTopic(String topic);
        String getTopic();

        void setCred(String credFilePath);
        String getCred();
    }

    public static void main(String args[]) throws IOException, ParseException {
        // create options
        MessageHubOptions options =
                PipelineOptionsFactory.fromArgs(args)
                        .withValidation()
                        .as(MessageHubOptions.class);

        // parse credentials
        JSONParser parser = new JSONParser();
        JSONObject cred = (JSONObject) parser.parse(new FileReader(options.getCred()));

        // cat servers into a string
        String bootstrapServers = String.join(",", (JSONArray) cred.get(SERVERS));

        // add additional properties
        Map<String, Object> configs = new HashMap<>();
        configs.put(BOOTSTRAP_SERVERS, bootstrapServers);
        configs.put("security.protocol","SASL_SSL");
        configs.put("ssl.protocol","TLSv1.2");
        configs.put("ssl.enabled.protocols","TLSv1.2");
        configs.put("sasl.mechanism","PLAIN");

        String username = (String) cred.get("user");
        String password = (String) cred.get("password");
        String jaas = JAAS_TEMPLATE.replace("USERNAME", username)
                                   .replace("PASSWORD", password);
        configs.put("sasl.jaas.config", jaas);

        // run the producer
        Pipeline wp = Pipeline.create(options);
        wp.apply(GenerateSequence.from(0)
                .withRate(10, Duration.standardSeconds(1)))
          .apply(MapElements
                .into(TypeDescriptors.kvs(TypeDescriptors.strings(), TypeDescriptors.strings()))
                .via((Long x) -> KV.of("beam", String.valueOf(x))))
          .apply(KafkaIO.<String, String>write()
                  .withBootstrapServers(bootstrapServers)
                  .withTopic(options.getTopic())
                  .updateProducerProperties(configs)
                  .withKeySerializer(StringSerializer.class)
                  .withValueSerializer(StringSerializer.class));

        wp.run().waitUntilFinish();
    }
}
```

To launch the producer, run the following command after replacing `your-topic`
with a proper name.

```bash
$ java -cp $STREAMS_BEAM_TOOLKIT/lib/com.ibm.streams.beam.translation.jar:$STREAMS_INSTALL/lib/com.ibm.streams.operator.samples.jar:/path/to/mh.jar \
  com.ibm.streams.beam.example.Producer\
  --topic="your-topic" \
  --runner=StreamsRunner \
  --jarsToStage=/path/to/mh.jar \
  --contextType=STANDALONE \
  --cred=mh.cred
```

Creating a consumer takes similar steps, where the main difference is to use
`KafkaIO.read()` instead of `KafkaIO.write()`.

```java
Pipeline rp = Pipeline.create(options);
rp.apply(KafkaIO.<String, String>read()
        .withBootstrapServers(bootstrapServers)
        .withTopic(options.getTopic())
        .updateConsumerProperties(configs)
        .withKeyDeserializer(StringDeserializer.class)
        .withValueDeserializer(StringDeserializer.class)
        .withoutMetadata())
  .apply(Values.create());

rp.run().waitUntilFinish();
```
