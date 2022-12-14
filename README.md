# Schema References Sandbox

Schema References is a feature introduced in Confluent Platform 5.5 that 
allows to mix different types, in the same topic, and still being able to 
enforce types and data type validation using a `TopicNameStrategy`.

This sample repository contains the example, source code and 
configuration described in 
[this great post](https://www.confluent.io/blog/multiple-event-types-in-the-same-kafka-topic/) 
by [@rayokota](https://github.com/rayokota).


## Starting the environment

```shell
docker-compose up -d
```


## Registering schemas

```shell
mvn schema-registry:register
```

From the output, capture the subject ID for `all-types-value`. You will need that value for the producer application. In the example execution below, the value is 3. Note that you can also get that ID executing `curl -XGET http://localhost:8081/subjects/all-types-value/versions/1` 

```shell
[INFO] Scanning for projects...
[INFO]
[INFO] -------------< org.mcascallares:schema-references-sandbox >-------------
[INFO] Building schema-references-sandbox 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- kafka-schema-registry-maven-plugin:6.1.0:register (default-cli) @ schema-references-sandbox ---
[INFO] Registered subject(customer) with id 1 version 1
[INFO] Registered subject(product) with id 2 version 1
[INFO] Registered subject(all-types-value) with id 3 version 1
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  3.149 s
[INFO] Finished at: 2021-05-24T15:13:33+02:00
[INFO] ------------------------------------------------------------------------
```


## Retrieving registered schemas

```shell
curl -XGET http://localhost:8081/subjects | jq
[
  "product",
  "all-types-value",
  "customer"
]
```

```shell
curl -XGET http://localhost:8081/subjects/product/versions/1 | jq
{
  "subject": "product",
  "version": 1,
  "id": 2,
  "schema": "{\"type\":\"record\",\"name\":\"Product\",\"namespace\":\"org.matias\",\"fields\":[{\"name\":\"product_id\",\"type\":\"int\"},{\"name\":\"product_name\",\"type\":\"string\"},{\"name\":\"product_price\",\"type\":\"double\"}]}"
}
```

```shell
curl -XGET http://localhost:8081/subjects/customer/versions/1 | jq
{
  "subject": "customer",
  "version": 1,
  "id": 1,
  "schema": "{\"type\":\"record\",\"name\":\"Customer\",\"namespace\":\"org.matias\",\"fields\":[{\"name\":\"customer_id\",\"type\":\"int\"},{\"name\":\"customer_name\",\"type\":\"string\"},{\"name\":\"customer_email\",\"type\":\"string\"},{\"name\":\"customer_address\",\"type\":\"string\"}]}"
}
```

```shell
curl -XGET http://localhost:8081/subjects/all-types-value/versions/1 | jq
{
  "subject": "all-types-value",
  "version": 1,
  "id": 3,
  "references": [
    {
      "name": "io.confluent.examples.avro.Customer",
      "subject": "customer",
      "version": 1
    },
    {
      "name": "io.confluent.examples.avro.Product",
      "subject": "product",
      "version": 1
    }
  ],
  "schema": "[\"org.matias.Customer\",\"org.matias.Product\"]"
}
```


## Starting consumer

```shell
docker-compose exec schema-registry kafka-avro-console-consumer \
    --bootstrap-server broker:9092 \
    --topic all-types \
    --from-beginning
```


## Starting producer mixing types and being validated

```shell
# <top-level-id> is 3 in this example
docker-compose exec schema-registry kafka-avro-console-producer \
    --bootstrap-server broker:9092 \
    --topic all-types \
    --property value.schema.id=<top-level-id> \
    --property auto.register=false \
    --property use.latest.version=true

{ "org.matias.Product": { "product_id": 1, "product_name" : "rice", "product_price" : 100.00 } } 
{ "org.matias.Customer": { "customer_id": 100, "customer_name": "acme", "customer_email": "acme@google.com", "customer_address": "1 Main St" } } 
```

You can also try to add a non existent json and the producer will fail: 

```shell
{ "org.matias.NonExistent": { "field" : 10} }

org.apache.kafka.common.errors.SerializationException: Error deserializing json { "org.matias.NonExistent": { "field" : 10} } to Avro of schema [{"type":"record","name":"Customer","namespace":"org.matias","fields":[{"name":"customer_id","type":"int"},{"name":"customer_name","type":"string"},{"name":"customer_email","type":"string"},{"name":"customer_address","type":"string"}]},{"type":"record","name":"Product","namespace":"org.matias","fields":[{"name":"product_id","type":"int"},{"name":"product_name","type":"string"},{"name":"product_price","type":"double"}]}]
	at io.confluent.kafka.formatter.AvroMessageReader.readFrom(AvroMessageReader.java:134)
	at io.confluent.kafka.formatter.SchemaMessageReader.readMessage(SchemaMessageReader.java:325)
	at kafka.tools.ConsoleProducer$.main(ConsoleProducer.scala:50)
	at kafka.tools.ConsoleProducer.main(ConsoleProducer.scala)
Caused by: org.apache.avro.AvroTypeException: Unknown union branch org.matias.NonExistent
	at org.apache.avro.io.JsonDecoder.readIndex(JsonDecoder.java:434)
	at org.apache.avro.io.ResolvingDecoder.readIndex(ResolvingDecoder.java:282)
	at org.apache.avro.generic.GenericDatumReader.readWithoutConversion(GenericDatumReader.java:188)
	at org.apache.avro.generic.GenericDatumReader.read(GenericDatumReader.java:161)
	at org.apache.avro.generic.GenericDatumReader.read(GenericDatumReader.java:154)
	at io.confluent.kafka.schemaregistry.avro.AvroSchemaUtils.toObject(AvroSchemaUtils.java:214)
	at io.confluent.kafka.formatter.AvroMessageReader.readFrom(AvroMessageReader.java:124)
	... 3 more
[2022-11-28 21:16:04,182] INFO [Producer clientId=console-producer] Closing the Kafka producer with timeoutMillis = 9223372036854775807 ms. (org.apache.kafka.clients.producer.KafkaProducer)
[2022-11-28 21:16:04,194] INFO Metrics scheduler closed (org.apache.kafka.common.metrics.Metrics)
[2022-11-28 21:16:04,195] INFO Closing reporter org.apache.kafka.common.metrics.JmxReporter (org.apache.kafka.common.metrics.Metrics)
[2022-11-28 21:16:04,195] INFO Metrics reporters closed (org.apache.kafka.common.metrics.Metrics)
[2022-11-28 21:16:04,195] INFO App info kafka.producer for console-producer unregistered (org.apache.kafka.common.utils.AppInfoParser)
```


## Verifying the broker side schema validation

Based on [this demo](https://docs.confluent.io/platform/current/schema-registry/schema-validation.html#demo-enabling-sv-on-a-topic-at-the-command-line).

1. Create the topic

```shell
  docker-compose exec broker kafka-topics --bootstrap-server broker:9092 --create --partitions 1 --replication-factor 1 --topic test-schemas
```

2. Start a producer (using default serializer)

```shell
  docker-compose exec broker kafka-console-producer --broker-list broker:9092 --topic test-schemas --property parse.key=true --property key.separator=,
```

3. Produce something as example (`1` is the key, `my first record` is the value, no schemas are reinforced)
```
  >1,my first record
```

4. In another shell, create a consumer (you should see the event produced above)
```shell
  docker-compose exec broker kafka-console-consumer --bootstrap-server broker:9092 --from-beginning --topic test-schemas --property print.key=true
```

5. Let's enable the validation (you should see `Completed updating config for topic test-schemas.`)

```shell
  docker-compose exec broker kafka-configs --bootstrap-server broker:9092 --alter --entity-type topics --entity-name test-schemas --add-config confluent.value.schema.validation=true
```

6. Add a new record in the producer started on step 2.
```
  >2,my second record
```

You will see an exception returned:

```shell
>[...] ERROR Error when sending message to topic test-schemas with key: 1 bytes, value: 16 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
org.apache.kafka.common.InvalidRecordException: Log record DefaultRecord(offset=0, timestamp=1671010385783, key=1 bytes, value=16 bytes) is rejected by the record interceptor io.confluent.kafka.schemaregistry.validator.RecordSchemaValidator
```

7. Let's now disable the validation (you should see `Completed updating config for topic test-schemas.`)

```shell
  docker-compose exec broker kafka-configs --bootstrap-server broker:9092 --alter --entity-type topics --entity-name test-schemas --add-config confluent.value.schema.validation=false
```

8. And add a third record in the producer started on step 2.
```
  >3,the third record
```

This one should be accepted as there is no validation on the broker side.

## Clean-up

1. Stop the consumer/producer with a Ctrl + C
2. Stop docker containers

```shell
docker-compose down -v
```