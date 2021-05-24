# Schema References Sandbox

This sample repository contains the example, source code and configuration described in [this post](https://www.confluent.io/blog/multiple-event-types-in-the-same-kafka-topic/).


## Starting the environment

```
docker-compose up -d
```


## Registering schemas

```
mvn schema-registry:register
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
curl -XGET curl -XGET http://localhost:8081/subjects/product/versions
[
  1
]
```

```
curl -XGET curl -XGET http://localhost:8081/subjects/customer/versions
[
  1
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
docker-compose exec schema-registry kafka-avro-console-consumer --topic all-types --bootstrap-server broker:9092 --from-beginning
```


## Starting producer (assuming version=1)

```
docker-compose exec schema-registry kafka-avro-console-producer --bootstrap-server broker:9092 --topic all-types --property value.schema.id=<top-level-id> --property auto.register=false --property use.latest.version=true
{ "org.matias.Product": { "product_id": 1, "product_name" : "rice", "product_price" : 100.00 } } 
{ "org.matias.Customer": { "customer_id": 100, "customer_name": "acme", "customer_email": "acme@google.com", "customer_address": "1 Main St" } } 
```
