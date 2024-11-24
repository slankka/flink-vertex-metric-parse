## DDL of the source

Filesystem connector is originally supported by flink.

We can use it to read the jobmanager archived directory.

* monitor-internal is optional, it will continuously running if sets.

```SQL
create table archived_logs (
    archive ARRAY<ROW(path STRING, json STRING)>,
   `file.name` STRING NOT NULL METADATA
) with (
   'connector' = 'filesystem',
   'format' = 'json',
   'path' = 'hdfs:///completed-jobs/',
   'source.monitor-interval' = '60 SEC'
);
```

## DDL of the print sink

To debug or look the result, we can just sink to console. The output will be located at taskmanager console. 

```SQL
create table archived_log_taskmanager(
  vertexId STRING,
  taskmanager ARRAY<ROW(host STRING, taskmanagerid STRING)>
) WITH  (
'connector' = 'print'
);
```

## Simple DML to verfiy the source
I used CROSS JOIN with UNNEST to unnest the `archive` json array.

```SQL
INSERT INTO archived_log_taskmanager
SELECT JSON_VALUE(t.json, '$.id') AS vertexId, 
ARRAY [ROW (JSON_VALUE(t.json, '$.taskmanagers[0].host'),
 JSON_VALUE(t.json, '$.taskmanagers[0].["taskmanager-id"]'))] AS taskmanager
 from archived_logs CROSS JOIN UNNEST(archived_logs.archive) as t(path, json) where
 t.path  like '%taskmanagers';
```

output:
```
+I[cbc357ccb763df2852fee8c4fc7d55f2, [+I[kkan129131:46379, container_e87_1724243239726_1412_01_000002]]]
+I[f6dc7f4d2283f4605b127b9364e21148, [+I[kkan129131:46379, container_e87_1724243239726_1412_01_000002]]]
+I[cbc357ccb763df2852fee8c4fc7d55f2, [+I[kkan130204:44749, container_e87_1724243239726_1892_01_000002]]]
```

## DDL of the database sink
After the verify, my ambition is to collect metrics as much as possible:

I used a analytic database Apache Doris, note: any database such as mysql would be fine.

**Precondition**: Just create the physic table with actual columns in your database.

```SQL
create table taskmanager_container_history
(
    APP_ID         STRING null COMMENT 'parsed application_id',
    JOB_ID         STRING null COMMENT 'id from path',
    VERTEX_ID      STRING null COMMENT 'id',
    CONTAINER_ID   STRING null COMMENT 'taskmanager-id',
    CONTAINER_HOST STRING null COMMENT 'host',
    NAME           STRING NULL COMMENT 'name',
    START_TIME     STRING NULL COMMENT 'start-time',
    END_TIME       STRING NULL COMMENT 'end-time',
    DURATION       STRING NULL COMMENT 'duration',
    STATUS         STRING NULL COMMENT 'status'
) with (
  'connector' = 'doris',
  'fenodes' = 'FENODES:FE_HTTP_PORT',
  'table.identifier' = 'schema.table',
  'jdbc-url' = 'jdbc:mysql://FENODES:FE_PORT/schema?useUnicode=true&characterEncoding=UTF8',
  'username' = 'username',
  'password' = 'password',
  'sink.enable-delete' = 'true',
  'sink.properties.format' = 'csv',
  'sink.properties.read_json_by_line' = 'true',
  'sink.properties.partial_columns' = 'false',
  'sink.label-prefix' = 'doris_label'
);
```

## Little concepts:

**The Yarn Application Id**

It's magic, application id contains in container id.
The application-id of `container_e87_1724243239726_1892_01_000002` is `application_1724243239726_1892`.

**The Host**

Why the hosts of taskmanagers are incompleted?

The answer is, flink definition of hostname FQDN:

```
org.apache.flink.runtime.taskmanager.TaskManagerLocation.DefaultHostNameSupplier#getHostName

If the FQDN is the textual IP address, then the hostname is also the IP address
If the FQDN has only one segment (such as "localhost", or "host17"), then this is used as the hostname.
If the FQDN has multiple segments (such as "worker3.subgroup.company.net"), then the first segment (here "worker3") will be used as the hostname
```

Note: the reverse resolve of taskmanager hostname are enabled by default. And it's recommend to enable.

`jobmanager.retrieve-taskmanager-hostname`=`true`

## The final DML job

**NOTE**
Be careful, the `JSON_QUERY` and `JSON_VALUE` are different while using `REGEXP_EXTRACT`.

```SQL
INSERT INTO taskmanager_container_history
SELECT appid, jobId, vertexId, taskManagerId, hosts,name, startTime, endtime, duration, status from (
SELECT 
JSON_VALUE(t.json, '$.name') AS name,
archived_logs.`file.name` AS jobId,
JSON_VALUE(t.json, '$.id') AS vertexId, 
COALESCE(JSON_QUERY(t.json, 'lax $.taskmanagers[*]["host"]'), '[]') as hosts,
COALESCE(JSON_QUERY(t.json, 'lax $.taskmanagers[*]["status"]' ), '[]') as status,
COALESCE(JSON_QUERY(t.json, 'lax $.taskmanagers[*]["start-time"]' ), '[]') as startTime,
COALESCE(JSON_QUERY(t.json, 'lax $.taskmanagers[*]["end-time"]' ), '[]') as endtime,
COALESCE(JSON_QUERY(t.json, 'lax $.taskmanagers[*]["duration"]' ), '[]')  as duration,
COALESCE(JSON_QUERY(t.json, 'lax $.taskmanagers[*]["taskmanager-id"]'), '[]') as taskManagerId,
CONCAT('application', REGEXP_EXTRACT(JSON_VALUE(t.json, '$.taskmanagers[0]["taskmanager-id"]'), 'container_\w+(_\d+_\d+)_\d+_\d+', 1)) as appid
from archived_logs CROSS JOIN UNNEST(archived_logs.archive) as t(path, json)  where t.path like '%taskmanagers'
) q

```

Because of the complexity of the json structure, I only extract taskmanager properties as json array string.
Less work left to convert array to data rows again. I haven't found simple way to unnest the data.

## How to execute Flink SQL
Flink SQL client, Flink Table API, Cloud Infrastructure interface.

## How to use the data

for example:
**Taskmanager log**

To generate a full url points to taskmanager log:

**INPUT**
* yarn job history serivce link prefix: `http://hist.yarn.slankka.com:19888/jobhistory/logs/`
* the job creator username: `slankka`
* the container hosts suffix: `.yarn.slankka.com`

*unfortunately neigher flink archived log provides job creator username, nor Flink filesystem connector supports user group information metadata at this moment.*

**OUTPUT**
```
http://hist.yarn.slankka.com:19888/jobhistory/logs/kkan129131.yarn.slankka.com:46379/container_e87_1724243239726_1412_01_000002/container_e87_1724243239726_1412_01_000002/slankka
```





