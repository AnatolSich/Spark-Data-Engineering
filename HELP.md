##Task
Avro part (3 point)

1. In this task, you need to create a Kafka producer and consumer using Avro to serialize messages.
2. Create a simple Kafka producer that sends a simple message to a topic serializing it using Avro.
3. Create a simple Kafka consumer which listens to a topic for a message, deserializes the message, 
and prints it.
4. Start Kafka, create a topic, and run consumer and producer.
5. Try to use different Avro schemas for serialization and deserialization. Include Avro schema version/id 
into the message so a client can understand which schema to use.





# Guides
1. Generate Avro with mvn clean compile
2. Start Kafka with docker compose
3. Run Consumer
4. Run Producer