# Kafka-ATM-Transaction-Processing-with-Message-Keys-and-Offsets
This project demonstrates how Apache Kafka can be used to process and manage ATM transactions efficiently. It shows how to maintain the order of messages in ATM transactions, leveraging Kafka's features like message keys, consumer offsets, and partitions. This setup ensures that messages from the same ATM are processed in sequence,making it highly reliable for financial transactions where message order and accuracy are crucial.

### Why is this Important for ATM Transactions?

In an ATM network, multiple transactions can occur simultaneously across different machines. Kafka helps manage these streams by ensuring that messages from the same ATM (identified by their unique ATM ID) are processed in the correct order. This guarantees that financial transactions are handled accurately, with no mix-ups between ATMs.

---


### Exercise 1: Download and Extract Kafka

1. **Open a terminal**: Navigate to `Terminal -> New Terminal` from the menu bar.
2. **Download Kafka**:  
   ```
   wget https://downloads.apache.org/kafka/3.8.0/kafka_2.13-3.8.0.tgz
   ```
3. **Extract Kafka**:  
   ```
   tar -xzf kafka_2.13-3.8.0.tgz
   ```

### Exercise 2: Configure and Start Kafka Server

1. **Navigate to the Kafka directory**:  
   ```
   cd kafka_2.13-3.8.0
   ```
2. **Generate a Cluster UUID** for the Kafka cluster:  
   ```
   KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
   ```
3. **Configure log directories** and initialize Kafka:  
   ```
   bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties
   ```
4. **Start the Kafka server**:  
   ```
   bin/kafka-server-start.sh config/kraft/server.properties
   ```

### Exercise 3: Create a Kafka Topic for ATM Transactions

1. **Create a `bankbranch` topic** with 2 partitions:  
   ```
   bin/kafka-topics.sh --create --topic bankbranch --partitions 2 --bootstrap-server localhost:9092
   ```
2. **Verify topic creation**:  
   ```
   bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
   bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic bankbranch
   ```

### Exercise 4: Produce and Consume ATM Messages

1. **Create a producer to send ATM transaction messages**:  
   ```
   bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic bankbranch
   ```
   Example ATM messages:
   ```
   {"atmid": 1, "transid": 100}
   {"atmid": 2, "transid": 200}
   ```

2. **Start a consumer to read the messages**:  
   ```
   bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch --from-beginning
   ```


### **Exercise 5: Use Message Keys to Maintain Order**

In Kafka, message keys play a critical role in maintaining the order of messages, especially when a topic has multiple partitions. By using message keys, Kafka ensures that all messages with the same key are directed to the same partition, thus maintaining the order within that partition. This is essential when dealing with transactions from the same source (like an ATM machine) where the order of transactions is crucial.

#### **1. Start a Producer with Message Keys Enabled**

To produce messages with keys, the producer needs to be configured to accept keys along with the message. This is done using the `parse.key=true` property in the producer command. The `key.separator=:` property defines the separator between the key and the message content.

Run the following command to start a producer that uses keys:

```bash
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic bankbranch --property parse.key=true --property key.separator=:
```

Here’s what’s happening:
- **`parse.key=true`**: Enables the producer to accept a key-value pair, where the key determines which partition the message will go to.
- **`key.separator=:`**: Specifies the separator between the key and the value. In this case, the colon `:` is used as a separator.

##### **Example Keyed Messages**

After starting the producer, you can produce messages that look like this:

```bash
1:{"atmid": 1, "transid": 103}
2:{"atmid": 2, "transid": 202}
```

Here’s the structure:
- `1` is the message key (representing the ATM machine ID 1).
- `{"atmid": 1, "transid": 103}` is the message value, which represents a transaction for ATM 1 with transaction ID 103.

Messages with the same key (e.g., ATM ID 1) will always be routed to the same partition, maintaining the order of transactions from that ATM machine.

#### **2. Start a Consumer to View Keyed Messages**

To consume messages along with their keys, you need to enable key printing using the `print.key=true` property in the consumer command. The `key.separator=:` ensures the key and value are separated clearly in the output.

Run the following command to start a consumer:

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch --from-beginning --property print.key=true --property key.separator=:
```

The consumer output will look like this:

```bash
1: {"atmid": 1, "transid": 103}
2: {"atmid": 2, "transid": 202}
```

- `1` is the message key (ATM ID).
- The transaction data `{"atmid": 1, "transid": 103}` is the message value.

Since the same key always goes to the same partition, Kafka ensures that messages for the same key (ATM machine) are consumed in the correct order.

---

### **Exercise 6: Working with Consumer Offsets**

Kafka tracks the progress of each consumer group using **consumer offsets**. The consumer offset is the position of the next message to be consumed. Kafka ensures that each consumer group consumes messages from where they left off, which is crucial for maintaining state across restarts or failures.

#### **1. Create a Consumer Group**

Kafka supports multiple consumers grouped together into a **consumer group**. Each consumer in the group will consume messages from different partitions, allowing Kafka to scale message consumption.

Run the following command to start a consumer in a specific group:

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic bankbranch --group atm-app
```

Here’s what’s happening:
- **`--group atm-app`**: Creates or uses a consumer group called `atm-app`.
- Each consumer in the group will read from different partitions of the `bankbranch` topic.

If you have two partitions, one consumer in the group will read from one partition, and another consumer (if present) will read from the other partition.

#### **2. Check Consumer Group Details**

To see the current progress of a consumer group (i.e., its offsets), you can describe the group:

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group atm-app
```

The output will look something like this:

```bash
GROUP           TOPIC      PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID         HOST            CLIENT-ID
atm-app         bankbranch 0          10              12              2               consumer-1          /127.0.0.1      consumer-1-id
atm-app         bankbranch 1          5               5               0               consumer-2          /127.0.0.1      consumer-2-id
```

Explanation of columns:
- **CURRENT-OFFSET**: The last message offset that the consumer has successfully processed.
- **LOG-END-OFFSET**: The latest offset available in the partition.
- **LAG**: The difference between the current offset and the log end offset, showing how many messages the consumer is behind.
  
In this example, partition 0 is 2 messages behind (`LAG` = 2), while partition 1 is up-to-date (`LAG` = 0).

#### **3. Reset the Consumer Offset**

Sometimes, you may need to reset the consumer offset to reprocess old messages or start fresh from a specific point in the topic. Kafka allows resetting offsets in various ways.

To reset the consumer offset to the earliest available message (reprocess all messages from the beginning):

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --topic bankbranch --group atm-app --reset-offsets --to-earliest --execute
```

Here’s what this command does:
- **`--reset-offsets`**: Instructs Kafka to reset the consumer group's offsets.
- **`--to-earliest`**: Resets the offset to the earliest message in the topic, allowing the consumer to start from the very beginning.
- **`--execute`**: Applies the reset. Without `--execute`, Kafka only shows what would happen.

After running this command, the consumer group will start reading messages from the beginning of the topic.

You can verify the reset by describing the consumer group again:

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group atm-app
```

---

These exercises show how Kafka's message key and offset management features provide control and reliability, especially in applications like ATM transaction processing where message order and fault tolerance are critical.

### Next Steps

- Produce more messages and verify consumption.
- Reset offsets to process messages from specific positions.

---

This project showcases the power of Kafka for managing large-scale, real-time data streams in ATM transactions, ensuring order, consistency, and scalability across the system.
