
# Twitter-Streaming-using-Apache-kafka-in-Java

Working examples of Apache Kafka fundamentals and a Mini Project (Twitter Sreaming_Producer and Consumer Configurations) implemented in JAVA.

This project is developed based on [Stephane Marek's coursework on Apache Kafka for beginners in Udemy](https://www.udemy.com/share/1013hcAEMZclpURX4H/). I highly recommend taking his course on Udemy to get your hands on Apache Kafka Ecosystem, Architecture, Core Concepts and Operations that include Master Concepts such as Topics, Partitions, Brokers, Producers, Consumers, Extended APIs Overview (Kafka Connect, Kafka Streams) and Kafka CLI.  I would also suggest downloading <b>Conductor </b>, an Apache Kafka Desktop Client, a GUI over the Kafka ecosystem, devoloped by Stepane Marek and his team, to make the development and management of Apache Kafka clusters as easy as possible. Here is a link of it [Conductor || Kafka Desktop Client](https://www.conduktor.io/download/)

We created Conduktor, a GUI over the Kafka ecosystem, to make the development and management of Apache Kafka clusters as easy as possible.
### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#twitter-client)**Twitter Client**

For a twitter client, we have to register for a developer account in the twitter  [developer console](http://www.developer.twitter.com/). Since 2018, twitter has changed the policy for account approvals. So there will be a verification check for the approval of the account. After the account is approved, please create an app and generate an API key and Secret. You guys can follow  [this](https://docs.inboundnow.com/guide/create-twitter-application/)  tutorial for setting up a twitter dev account.

Next, we move to set up a Twitter Java client. We will be referring to  [hbc](https://github.com/twitter/hbc)  java client.  _First_, we need to set up a  _maven project_  and add the following dependency to the  _pom.xml_  file. This will provide access to all the required packages for setting up a java based twitter client.

```
 <dependencies>
    <dependency>
      <groupId>com.twitter</groupId>
      <artifactId>hbc-core</artifactId> <!-- or hbc-twitter4j -->
      <version>2.2.0</version>
    </dependency>
  </dependencies>

```

Next, Fetch all the auth parameters from the twitter dev console and wrap it up in an authentication call from the client.

```
// These secrets should be read from a config file
Authentication auth = new OAuth1("consumerKey", "consumerSecret", "token", "secret");

```

### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#kafka-producer)**KAFKA PRODUCER**

First, we will start the Kafka and Zookeeper server.  _/path/to/properties can vary on the mode of installation and OS._

```
kafka-server-start /usr/local/etc/kafka/server.properties
zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties

```

Next, we will create a topic on which producer will be publishing data.

```
kafka-topics --zookeeper 127.0.0.1:2181 --topic tweets --create --partitions 6 --replication-factor 1

```



```
KafkaProducer<String, String> producer = new KafkaProducer<String, String>(KafkaProducerConfig.getProperties());

```

For sending data and provide a callback function to log successful publish and trace errors/failures.

```
producer.send(new ProducerRecord<String, String>("tweets", null, msg), new Callback() {
                    @Override
                    public void onCompletion(RecordMetadata recordMetadata, Exception ex) {
                        if(ex!=null){
                            logger.error("Error sending message", ex);
                        }
                    }
                });

```

### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#advance-producer-configurations)**Advance Producer Configurations**
#### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#producer-acks)**Producer acks**

The number of Acknowledgment a leader is expecting before marking the request complete. It can have these variants:

-   **_acks=0_**  Leader does not expect any acknowledgment. If the broker goes down, there is every possibility that data is lost and cannot be recovered. No guarantees are assured in this case as the producer never gets to know whether the server has received the record or not, therefore  _retries_  never take place in this configuration. This is usually used when data is not critical and data loss hardly affects.
-   **_acks=1_**  Leader will commit to its log and revert back with acknowledgment without waiting for its followers. It might happen that the leader goes down immediately after sending acknowledgment and before the data gets replicated in any in-sync replicas. In that case, data might get lost.
-   **_acks=all_**  Leader and all in-sync replicas send an acknowledgment. This makes sure that data is not lost due to guaranteed replication and delivery acknowledgment from the broker and its followers. It adds in latency and safety both. This configuration is used mostly when data loss is very critical. Also,  **_acks=all_**  must be used in conjunction with  **_min.insync.replicas_**. Suppose  **_min.insync.replicas_**  =2, that means at least 2 brokers will send an acknowledgment and corollary we can only sustain one broker going down.

#### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#producer-retries)**Producer Retries**

In case of any potential transient error, while sending data in Kafka, the developer must handle the exceptions to avoid data loss. A transient error can be the broker goes down for time being and  _NotEnoughReplicasException_  is thrown.

There is a  _retries_  setting which defaults to 0 in Kafka <=2.0 and 2147483647 in Kafka >2.0. Suppose retries is set to a fairly higher number like 20000000, still, the producer won’t keep trying sending data as there is a  _timeout_  which controls and disconnects producer.

The timeout feature was added in Kafka 2.1 based on KIP-91. Its setting variable is  _delivery.timeout.ms_  and the default value is 120000ms or 2 mins.  
Records will be failed if the producer didn’t get acknowledgment before timeout.

Allowing retries without setting  _max.in.flights.request.per.connection_  will potentially change the ordering of records because if two batches are sent to a single partition, and the first fails and is retried but the second succeeds, then the records in the second batch may appear first. Therefore, if ordering is desired, please set  _max.in.flights.request.per.connection_  to 1.

#### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#idempotent-producer)**Idempotent Producer**

In the case of network errors, the producer may introduce duplicate records in Kafka. Brokers might not be able to identify duplicate requests coming from producers. In the event of failure of receipt of an acknowledgment, a producer might retry sending the same record, and broker might commit it twice.

If Kafka >0.11, an idempotent producer is built into producer config. This ensures a guarantee for a stable and safe pipeline. Kafka broker identifies similar requests by the attached  _producer-id_  with the requests.

Idempotent producer comes with these implicit configurations:

-   retries = Integer.MAX(2147483647)
-   max.in.flight.requests = 1 for kafka <0.11 and = 5 for kafka > 0.11
-   acks = all

#### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#producer-compression)**Producer Compression**

The producer usually sends data in text-based JSON/XML format across the internet. This format is usually bulky and occupies network bandwidth in comparison with compressed formats.

Compressions can be done at the producer level and don’t need any changes in the broker/consumer level.  _**compression.type**_  can be  _none, gzip, snappy, lz4_. Compression is more effective with a bigger batch of messages being sent to Kafka brokers.

Compression enables a much smaller batch request message size. Compression can be achieved until 4X of the original size. The only disadvantage is some CPU cycles are needed for compression and decompression at producer and consumer respectively.

Also, it is advisable to try out all the compression techniques for a given data, there is no standard guidelines for any technique to perform better in any scenario. Consider tweaking linger.ms and batch.size for having more number of messages for more compression and higher throughput.

#### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#producer-batching)**Producer Batching**

By default, Kafka tries to send data immediately. It will have at max 5 requests in at same time to send data. After this, if more data needs to be sent parallely or at same time, kafka is smart and it internally does batching.

This smart batching helps kafka to achieve higher throughput and compression with low latency.

**Linger ms**: No of seconds kafka waits before sending records in batch(default=0). By increasing this wait time, we try to increase chances of kafka sending a batch request.

**Batch.size**: Max no of bytes sent in one batch(default=16KB). Increasing batch size can also increase compression, throughput and efficiency of requests. Batch size will act as a filter, any message having more size will be discarded. A batch is allocated per partition, so choose wisely the batch size to avoid memory leaks.

#### [](https://github.com/bharadwaj995/Twitter-Streaming-uisng-Apche-kafka-in-Java/tree/9a94240d26f9c46b457f6c58763a306ef15dbf6f#maxblockms-and-buffermemory)
**Max.block.ms and buffer.memory**

In case, the producer is producing records faster than the consumer consumes it, the records will be buffered into memory. The buffer will fill up and gradually fill down once the throughput to the broker increases.

If the buffer is full(full 32 MB), the send() method of producer will start blocking, it no longer will send records asynchronously.  _max.block.ms_  specifies the time, the producer will block before throwing an exception.
