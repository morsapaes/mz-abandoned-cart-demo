# mz-abandoned-cart-demo

![mz-abandoned-cart-demo](https://user-images.githubusercontent.com/21223421/143267063-2dbb1ec2-d48d-4ba5-8da8-f0d9ac1404e4.png)

Initial outline:

### Create Kafka source

```
CREATE SOURCE orders
FROM KAFKA BROKER 'redpanda:9092' TOPIC 'mysql.shop.orders'
FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://redpanda:8081'
ENVELOPE DEBEZIUM;
```

```
SHOW COLUMNS FROM orders;
```

```
    name      | nullable |   type    
--------------+----------+-----------
 id           | f        | bigint
 user_id      | t        | bigint
 order_status | t        | integer
 price        | t        | numeric
 created_at   | f        | text
 updated_at   | t        | timestamp
```


### Create materialized views

```
CREATE MATERIALIZED VIEW orders_view AS
SELECT * FROM orders;
```

```
CREATE MATERIALIZED VIEW abandoned_orders AS
    SELECT
        user_id,
        order_status,
        SUM(price) as revenue,
        COUNT(id) AS total
    FROM orders_view
    WHERE order_status=0
    GROUP BY 1,2;
```

```
select * from abandoned_orders;
```

### Create Postgres source

```
CREATE MATERIALIZED SOURCE "mz_source" FROM POSTGRES
CONNECTION 'user=postgres port=5432 host=postgres dbname=postgres password=postgres'
PUBLICATION 'mz_source';
```

```
CREATE VIEWS FROM SOURCE mz_source (users);
```

### Create Kafka sink

```
 CREATE MATERIALIZED VIEW high_value_orders AS
      SELECT
        users.id,
        users.email,
        abandoned_orders.revenue,
        abandoned_orders.total
      FROM users
      JOIN abandoned_orders ON abandoned_orders.user_id = users.id
      GROUP BY 1,2,3,4
      HAVING revenue > 2000; 
```


```
      CREATE SINK high_value_orderssink
        FROM high_value_orders
        INTO KAFKA BROKER 'redpanda:9092' TOPIC 'high-value-orders-sink'
        FORMAT AVRO USING
        CONFLUENT SCHEMA REGISTRY 'http://redpanda:8081';
```

