##  YugabyteDB CDC demo: Streaming changes to downstream databases


### Step 1 - clone the repo


```cd ~/Github ; gh repo clone sumitdatabase/yb-dbz-blog```


--- 

### Step 2 - Bring up 9 containers 
- Zookeeper
- Kafka, Kafka-connect, Kafka-ui
- yb-master, yb-tserver
- pg 
- elastic 
- mysql



```docker-compose -f docker-compose-init.yaml up -d --force-recreate```

---

### Step 3 - create a source YB table. Do not enter any data.


```docker exec -it yb-tserver bash -c "ysqlsh -h yb-tserver"```


```create table demo ( id int primary key, name text, email varchar(100)) ;```

---

### Step 4 - create a CDC stream and update the yb-source.json with the stream id returned

```cdc=$(docker exec -it yb-master bash -c "yb-admin --master_addresses yb-master create_change_data_stream ysql.yugabyte")```


```stream_id=$(echo $cdc | cut -d: -f2 | sed 's/ //g')```


```sed "s/<insert_your_cdc_stream_id_here>/$stream_id/g" yb-source.json >temp.json```

```mv temp.json yb-source.json```

---

### Step 5 - Start replicating changes from this stream for demo table. Ensure the stream_id from above is entered in yb-source.json

```curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @yb-source.json```

---

### Step 6 - Deploy sink connectors. Postgres, MySQL, Elastic.

```curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @jdbc-sink-pg.json```

```curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @jdbc-sink-mysql.json```

```curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @elastic-sink.json```


---

### Step 7 - Insert test records into source

```docker exec -it yb-tserver bash -c "ysqlsh -h yb-tserver"```

```INSERT INTO demo VALUES (1, 'Vaibhav', 'foo@bar.com');```

---

### Step 8 - verify data in CDC sinks

```docker exec -it pg sh -c "psql -U postgres postgres```

```select * from demo ;```

```docker exec -it mysql sh -c "mysql -u root -pdebezium mysql"```

```select * from test;```

```curl 'localhost:9200/dbserver1.public.demo/_search?pretty'```

---