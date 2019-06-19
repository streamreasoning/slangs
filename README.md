## KSQL ([src](https://docs.confluent.io/current/ksql/docs/tutorials/examples.html))

### Prerequisites
* git
* docker
* docker compose

### Set up the environment
* clone the demo-scene repository form the official conflunet github: `git clone https://github.com/streamreasoning/DEBS-2019.git`
* go to ksql workshop forlder: `cd kafka`
* run the docker containers: `docker-compose up -d`
* open the ksql console: `docker-compose exec ksql-cli ksql http://ksql-server:8088`

### Tumbling Windows ([src](https://www.confluent.io/stream-processing-cookbook/ksql-recipes/detecting-analyzing-suspicious-network-activity))

**NOTE**: this examples involves the `pageviews` topic

Create the `pageviews` stream from the topic `pageviews` and use `viewtime` as timestamp for the tuples

```
CREATE STREAM pageviews
WITH (KAFKA_TOPIC='pageviews',
    VALUE_FORMAT='AVRO',
    TIMESTAMP='viewtime');
```

Count the number of pageviews per pageid **in the last minute**

```
CREATE TABLE pageid_30s AS
SELECT 
	pageid, 
	count(*) AS view_count
FROM pageviews
WINDOW TUMBLING (SIZE 30 SECONDS)
GROUP BY pageid;
```

Check the content of the new topic

```
SELECT 
	pageid, 
	view_count 
FROM pageid_30s;
```

### Sliding Windows
**NOTE**: this examples involves the `pageviews` topic

Count the number of pageviews per pageid **in the last minute** and **update the results every 10 secs**

```
CREATE TABLE pageid_30s_hop AS
SELECT 
	pageid, 
	count(*) AS view_count
FROM pageviews
WINDOW HOPPING (SIZE 30 SECONDS, ADVANCE BY 10 SECONDS)
GROUP BY pageid;
```

Check the content of the new topic

```
SELECT 
	pageid, 
	view_count 
FROM pageid_30s_hop;
```

### Session Windows ([src](https://www.confluent.io/stream-processing-cookbook/ksql-recipes/using-event-time-from-iot-sensor-readings))

**NOTE**: this examples involves the `pageviews` topic

Count the number of pageviews per pageid **in the last minute** and **update the results every 10 secs** with a session inactivity gap of 60 seconds

```
CREATE TABLE pageid_per_session AS
SELECT 
	pageid,
   TIMESTAMPTOSTRING(windowStart(), 'yyyy-MM-dd HH:mm:ss') AS window_start,
   TIMESTAMPTOSTRING(windowEnd(), 'yyyy-MM-dd HH:mm:ss') AS window_end,    
   count(*) AS view_count
FROM pageviews
WINDOW SESSION (30 SECONDS)
GROUP BY pageid;
```

Check the content of the new topic

```
SELECT
	window_start,
	window_end,
	pageid, 
	view_count 
FROM pageid_per_session;
```

### Stream-Table Join (KSQL only supports INNER and LEFT joins between a stream and a table - [src](https://docs.confluent.io/current/ksql/docs/developer-guide/join-streams-and-tables.html#stream-stream-joinss))

**NOTE**: this examples involves the `pageviews` and the `users` topics

Create the `users` table containing information about the users

```
CREATE TABLE users
  (registertime BIGINT,
   gender VARCHAR,
   regionid VARCHAR,
   userid VARCHAR,
   interests array<VARCHAR>,
   contactinfo map<VARCHAR, VARCHAR>)
WITH (
	KAFKA_TOPIC='users',
    VALUE_FORMAT='JSON',
    KEY = 'userid');
```

Create a new table `users_5prt` with 5 partitions

``` 
CREATE TABLE users_5part
WITH (PARTITIONS=5) AS
SELECT * FROM USERS; 	
```

**NOTE**: in order to join a stream and a table, the number of partitions maust be the same
 
Create a new stream `pageviews_transformed` from the `pageviews` topic

```
CREATE STREAM pageviews_transformed
  WITH (TIMESTAMP='viewtime',
        PARTITIONS=5,
        VALUE_FORMAT='JSON') AS
  SELECT viewtime,
         userid,
         pageid,
         TIMESTAMPTOSTRING(viewtime, 'yyyy-MM-dd HH:mm:ss.SSS') AS timestring
  FROM pageviews
  PARTITION BY userid;
```

Create the `pageviews_enriched` stream <


```
CREATE STREAM pageviews_enriched AS
  SELECT pv.viewtime,
         pv.userid AS userid,
         pv.pageid,
         pv.timestring,
         u.gender,
         u.regionid,
         u.interests,
         u.contactinfo
  FROM pageviews_transformed pv
  LEFT JOIN users_5part u ON pv.userid = u.userid;
```

Check the content of the new topic

```
SELECT * FROM pageviews_enriched;
```


### Stream-Stream Join (KSQL supports INNER, LEFT OUTER, and FULL OUTER joins between streams - [src](https://docs.confluent.io/current/ksql/docs/developer-guide/join-streams-and-tables.html#stream-table-joins))

Imagine that the `users` topic are constantly updated with new information.

Create the `users_str` stream from the topic `users` and use `registertime` as timestamp for the tuples

```
CREATE STREAM users_str
WITH (KAFKA_TOPIC='users',
    VALUE_FORMAT='AVRO',
    TIMESTAMP='registertime');
```

Create a new stream of the users with 5 partition

```
CREATE STREAM users_str_5p
WITH (VALUE_FORMAT='JSON',
    PARTITIONS=5,
    TIMESTAMP='registertime') AS 
	 SELECT registertime,
      userid,
      regionid,
      gender,
      TIMESTAMPTOSTRING(registertime, 'yyyy-MM-dd HH:mm:ss.SSS') AS timestring
    FROM users_str;
 ```

Create the stream `pageviews_enriched_str` by joining (LEFT OUTER JOIN)  the `pageviews_transformed ` stream with the `users_str_5p` stream.
The new stream will contain all the information of pages and users, even if a page is not yet been visited.

```
CREATE STREAM pageviews_enriched_full AS
  SELECT pv.*, u.*
  FROM pageviews_transformed pv
  LEFT OUTER JOIN users_str_5p u
  WITHIN 30 SECONDS
  ON (pv.userid = u.userid);                       
```

Check the content of the new topic

```
SELECT * FROM pageviews_enriched_full;
```



