# Bi-directional cluster link: Active-Active

This is just an adaptation of the original guide by Tomas Almeida: https://github.com/tomasalmeida/cluster-schema-linking-examples/tree/main/bidirectional-link 

## Start the clusters

```shell
    docker compose up -d

    docker compose logs -f
``` 

Tail the logs expecting everything to start.

Two CP clusters (ZK+Broker+SR+C3) are running:

*  Left Control Center available at [http://localhost:19021](http://localhost:19021/)
*  Right Control Center available at [http://localhost:29021](http://localhost:29021/)
*  Center Control Center available at [http://localhost:39021](http://localhost:39021/)
*  Left Schema Register available at [http://localhost:8085](http://localhost:8085/)
*  Right Schema Register available at [http://localhost:8086](http://localhost:8086/)
*  Center Schema Register available at [http://localhost:8087](http://localhost:8087/)

## Create the topic `test` skipping the schema registry setup to keep it simple

```shell
#curl -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8085/subjects/product-value/versions

#curl -v -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data @data/product.avsc http://localhost:8086/subjects/product-value/versions  

docker compose exec leftKafka kafka-topics --bootstrap-server leftKafka:19092 --topic test --create --partitions 1 --replication-factor 1

docker compose exec centerKafka kafka-topics --bootstrap-server centerKafka:39092 --topic test --create --partitions 1 --replication-factor 1

docker compose exec rightKafka kafka-topics --bootstrap-server rightKafka:29092 --topic test --create --partitions 1 --replication-factor 1
```

## Create the schema linking skipping the schema registry setup to keep it simple 

### Create schema linking from left to right

```shell
#docker compose exec leftSchemaregistry bash -c '\
    echo "schema.registry.url=http://rightSchemaregistry:8086" > /home/appuser/config.txt'
    
#docker compose exec leftSchemaregistry bash -c '\
    schema-exporter --create --name left-to-right-sl --subjects "product-value" \
    --config-file ~/config.txt \
    --schema.registry.url http://leftSchemaregistry:8085 \
    --subject-format "left.product-value" \
    --context-type NONE'
```

### Create schema linking from right to left

```shell
#docker compose exec rightSchemaregistry bash -c '\
    echo "schema.registry.url=http://leftSchemaregistry:8085" > /home/appuser/config.txt'

#docker compose exec rightSchemaregistry bash -c '\
    schema-exporter --create --name right-to-left-sl --subjects "product-value" \
    --config-file ~/config.txt \
    --schema.registry.url http://rightSchemaregistry:8086 \
    --subject-format "right.product-value" \
    --context-type NONE'
```

## Create the cluster linking

### Create cluster linking from left to right

```shell
docker compose exec rightKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
cluster.link.prefix=left.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl.properties'

docker compose exec rightKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'

docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 \
    --create --link bidirectional-linkAC \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
```

**# Create cluster linking from left to center**
```shell
docker compose exec centerKafka bash -c '\
echo "\
bootstrap.servers=leftKafka:19092
link.mode=BIDIRECTIONAL
cluster.link.prefix=left.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl.properties'

docker compose exec centerKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl-offset-groups.json'

docker compose exec centerKafka \
    kafka-cluster-links --bootstrap-server centerKafka:39092 \
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl.properties \
    --consumer-group-filters-json-file /home/appuser/cl-offset-groups.json
```

**# Create cluster linking from right to left**
```shell
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
link.mode=BIDIRECTIONAL
cluster.link.prefix=right.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl1.properties'
docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl1-offset-groups.json'
docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-linkAC \
    --config-file /home/appuser/cl1.properties \
    --consumer-group-filters-json-file /home/appuser/cl1-offset-groups.json
```

**# Create cluster linking from center to left**
```shell
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=centerKafka:39092
link.mode=BIDIRECTIONAL
cluster.link.prefix=center.
consumer.offset.sync.enable=true
consumer.offset.sync.ms=1000
" > /home/appuser/cl2.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl2-offset-groups.json'

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-linkAB \
    --config-file /home/appuser/cl2.properties \
    --consumer-group-filters-json-file /home/appuser/cl2-offset-groups.json
```shell

**check for link**

docker compose exec leftKafka \
 kafka-cluster-links --bootstrap-server leftKafka:19092  --list
docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --list
docker compose exec centerKafka \
    kafka-cluster-links --bootstrap-server centerKafka:39092 --list


docker compose exec leftKafka \
 kafka-cluster-links --bootstrap-server leftKafka:19092  --list
docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --list
docker compose exec centerKafka \
    kafka-cluster-links --bootstrap-server centerKafka:39092 --list


Create the mirror topic

```shell
docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic left.product \
    --link bidirectional-link \
    --bootstrap-server rightKafka:29092        
``` 

```shell
docker compose exec rightKafka \
    kafka-mirrors --create \
    --source-topic number \
    --mirror-topic left.number \
    --link bidirectional-link \
    --bootstrap-server rightKafka:29092 
```
### Create cluster linking from right to left

```shell
docker compose exec leftKafka bash -c '\
echo "\
bootstrap.servers=rightKafka:29092
link.mode=BIDIRECTIONAL
cluster.link.prefix=right.
consumer.offset.sync.enable=true
" > /home/appuser/cl2.properties'

docker compose exec leftKafka bash -c '\
echo "{\"groupFilters\": [{\"name\": \"*\",\"patternType\": \"LITERAL\",\"filterType\": \"INCLUDE\"}]}" > /home/appuser/cl2-offset-groups.json'

docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092 \
    --create --link bidirectional-link \
    --config-file /home/appuser/cl2.properties \
    --consumer-group-filters-json-file /home/appuser/cl2-offset-groups.json
``` 

Create the mirror topic

```shell
docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic product \
    --mirror-topic right.product \
    --link bidirectional-link \
    --bootstrap-server leftKafka:19092
``` 

```shell
docker compose exec leftKafka \
    kafka-mirrors --create \
    --source-topic number \
    --mirror-topic right.number \
    --link bidirectional-link \
    --bootstrap-server leftKafka:19092
``` 

//get this working:
seq 1 1000000 | onlyOdds    kafka-console-producer --topic number --bootstrap-server leftkafka:19092 --producer.config


## Checking the link is the same

```shell
 docker compose exec leftKafka \
    kafka-cluster-links --bootstrap-server leftKafka:19092  --link bidirectional-link --list
docker compose exec rightKafka \
    kafka-cluster-links --bootstrap-server rightKafka:29092 --link  bidirectional-link --list
```

Verifying the results:

- **Link name:** 'bidirectional-link' -> same name for both results, when the links were created, the same name was given to them
- **link ID:** same id in both links
- **remote cluster ID and local cluster ID:** are shown in a crossed way

## Test time!

### Active-active


producing to number topic in simple way 
left cluster:
 docker compose exec leftKafka bash
 [appuser@leftKafka ~]$ kafka-console-producer --bootstrap-server leftKafka:19092 --topic number

Consuming

kafka-console-consumer --bootstrap-server leftKafka:19092 \
        --group left-group \
        --from-beginning \
        --include ".*number" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true

right cluster:
docker compose exec rightKafka bash
kafka-console-producer --bootstrap-server rightKafka:29092 --topic number

1. Producer produces to left cluster **(top left terminal)**

```shell
docker compose exec leftSchemaregistry kafka-avro-console-producer \
    --bootstrap-server leftKafka:19092 \
    --topic product \
    --property value.schema.id=1 \
    --property schema.registry.url=http://leftSchemaregistry:8085 \
    --property auto.register=false \
    --property use.latest.version=true
```

Enter the messages to be produced:

```
    { "product_id": 1, "product_name" : "riceLeft"} 
    { "product_id": 2, "product_name" : "beansLeft"} 
```


2. Consumer consumes from left cluster **(bottom left terminal)**

```shell
docker compose exec leftSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server leftKafka:19092 \
        --property schema.registry.url=http://leftSchemaregistry:8085 \
        --group left-group \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```


3. Producer produces to right cluster **(top right terminal)** 
```shell
docker compose exec rightSchemaregistry kafka-avro-console-producer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --property value.schema.id=1 \
    --property schema.registry.url=http://rightSchemaregistry:8086 \
    --property auto.register=false \
    --property use.latest.version=true
```

Enter the messages to be produced:

```
 { "product_id": 3, "product_name" : "riceRight"} 
 { "product_id": 4, "product_name" : "beansRight"} 

    { "product_id": 1, "product_name" : "riceRight"} 
    { "product_id": 2, "product_name" : "beansRight"} 
```


4. Consumer consumes from right cluster **(bottom right terminal)**

```shell
docker compose exec rightSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server rightKafka:29092 \
        --property schema.registry.url=http://rightSchemaregistry:8086 \
        --group right-group \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```

```shell
kafka-console-consumer --bootstrap-server leftKafka:19092 --group left_group --include ".*number"
``````

### Disaster Mode

Now run consumers with same group id: **(just two side by side terminals)** 

```shell
docker compose exec leftSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server leftKafka:19092 \
        --property schema.registry.url=http://leftSchemaregistry:8085 \
        --group disaster-group \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```

```shell
docker compose exec rightSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server rightKafka:29092 \
        --property schema.registry.url=http://rightSchemaregistry:8086 \
        --group disaster-group \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```

What do you notice?

Count with at least once: Expect duplicates! (offsets are not sync'ed fast enough)

But if you close the right consumer and produce to it:

```shell
docker compose exec rightSchemaregistry kafka-avro-console-producer \
    --bootstrap-server rightKafka:29092 \
    --topic product \
    --property value.schema.id=1 \
    --property schema.registry.url=http://rightSchemaregistry:8086 \
    --property auto.register=false \
    --property use.latest.version=true
```

Entering:

```
{"product_id":3,"product_name":"tomato"}
```

You should see it on left consumer. Now close the left consumer and open the right consumer again:

```shell
docker compose exec rightSchemaregistry \
        kafka-avro-console-consumer --bootstrap-server rightKafka:29092 \
        --property schema.registry.url=http://rightSchemaregistry:8086 \
        --group disaster-group \
        --from-beginning \
        --include ".*product" \
        --property print.timestamp=true \
        --property print.offset=true \
        --property print.partition=true \
        --property print.headers=true \
        --property print.key=true \
        --property print.value=true
```

You shouldn't see anything. There was time for offsets to synch.

## Clean Up

```shell
docker compose down -v
```
