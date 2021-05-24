# Schema References Sandbox

Schema References is a feature introduced in Confluent Platform 5.5 that 
allows to mix different types, in the same topic, and still being able to 
enforce types and data type validation.

This sample repository contains the example, source code and 
configuration described in 
[this great post](https://www.confluent.io/blog/multiple-event-types-in-the-same-kafka-topic/) 
by [@rayokota](https://github.com/rayokota).


## Starting the environment

```
docker-compose up -d
```


## Registering schemas

```
mvn schema-registry:register
```

From the output, capture the subject ID for `all-types-value`. You will need that value for the producer application. In the example execution below, the value is 3. Note that you can also get that ID executing `curl -XGET http://localhost:8081/subjects/all-types-value/versions/1` 

```
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

```
curl -XGET http://localhost:8081/subjects
[
  "product",
  "all-types-value",
  "customer"
]
```

```
curl -XGET http://localhost:8081/subjects/product/versions/1
{
  "subject": "product",
  "version": 1,
  "id": 2,
  "schema": "{\"type\":\"record\",\"name\":\"Product\",\"namespace\":\"org.matias\",\"fields\":[{\"name\":\"product_id\",\"type\":\"int\"},{\"name\":\"product_name\",\"type\":\"string\"},{\"name\":\"product_price\",\"type\":\"double\"}]}"
}
```

```
curl -XGET http://localhost:8081/subjects/customer/versions/1
{
  "subject": "customer",
  "version": 1,
  "id": 1,
  "schema": "{\"type\":\"record\",\"name\":\"Customer\",\"namespace\":\"org.matias\",\"fields\":[{\"name\":\"customer_id\",\"type\":\"int\"},{\"name\":\"customer_name\",\"type\":\"string\"},{\"name\":\"customer_email\",\"type\":\"string\"},{\"name\":\"customer_address\",\"type\":\"string\"}]}"
}
```

```
curl -XGET http://localhost:8081/subjects/all-types-value/versions/1
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

```
docker-compose exec schema-registry kafka-avro-console-consumer \
    --bootstrap-server broker:9092 \
    --topic all-types \
    --from-beginning
```


## Starting producer

```
docker-compose exec schema-registry kafka-avro-console-producer \
    --bootstrap-server broker:9092 \
    --topic all-types \
    --property value.schema.id=<top-level-id> \
    --property auto.register=false \
    --property use.latest.version=true

{ "org.matias.Product": { "product_id": 1, "product_name" : "rice", "product_price" : 100.00 } } 
{ "org.matias.Customer": { "customer_id": 100, "customer_name": "acme", "customer_email": "acme@google.com", "customer_address": "1 Main St" } } 
```
