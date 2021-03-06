# Grafana Work to build monitoring using Docker, Telegraf, Influxdb, Grafana, PostgresSQL and etc.
Grafana is an open-source Data visualization & Monitoring with support for Graphite, InfluxDB, Prometheus, Elasticsearch and many more databases.
An a performance guys, I have been looking for an platform for beautiful analytics displaying and monitoring that without writting JavaScript in browsers.
This is the one, the solutions!!! (maybe later, too earlier to conclude - wanting to see what ElasticSearch stack offers - next)

## Getting Started
Follow the following instructions to setup a Telegraf, Influxdb, Grafana in docker:

### Installing
Follow the instructions given below:
* Explicitly create volumes grafana-volume & influxdb-volume and network monitoring
* Since volumes were created so the following dummy docker run will set appropriate users (e.g. admin, telegraf)  and create database (e.g. telegraf) in influxdb  
```
docker run --rm \
-e INFLUXDB_DB=telegraf 
-e INFLUXDB_ADMIN_ENABLED=true \
-e INFLUXDB_ADMIN_USER=admin -e INFLUXDB_ADMIN_PASSWORD=admin \
-e INFLUXDB_USER=telegraf -e INFLUXDB_USER_PASSWORD=telegraf \
-v influxdb-volume:/var/lib/influxdb \
influxdb /init-influxdb.sh 
```
* Spin up InfluxDB, Grafana containers using docker compose file - follow [Setup](https://towardsdatascience.com/get-system-metrics-for-5-min-with-docker-telegraf-influxdb-and-grafana-97cfd957f0ac)
* Install a Telegraf agent for Windows - Sample agent configuration file is here [telegraf](https://github.com/kangli914/grafana/blob/master/telegraf.conf)
```
[[outputs.influxdb]]
  urls = ["http://localhost:8086"]
  database = "telegraf"
  username = "telegraf"
  password = "telegraf"
```
## Configure Dashboard 
Create first dashboard using Telegraf agent metics
![alt](https://github.com/kangli914/grafana/blob/master/pic/dashboard.png "Dashboard")

## InfluxDB
* How 'win_cpu' table (in telegraf database) was stored in InfluxDB as time time series data - 'time' as primary key
* All fields in 'win_cpu' table mapped to 'Counters' definitions in 'Telegraf.Conf' file
![alt](https://github.com/kangli914/grafana/blob/master/pic/influxdb.png "influxdb")

## Miscellaneous Notes
Some notes while building dashboard something like below. Grafana + data from PostgresSQL Database
![alt](https://github.com/kangli914/grafana/blob/master/pic/examples.png "Dashboard")

### Table (type): Inverting table from column to row 
Sometimes, data are stored in different rows of table with same key (runid: 15390) but wanting to achieve to display in one row of table in grafana (columns --> row):
![alt](https://github.com/kangli914/grafana/blob/master/pic/table_col2row.png "table1")
Tips:
* make Time-Series Table with 'Options': Time series to columns under 'Table Transform' (instead of rows)
* standards about 'time', 'value' and 'metric':
1. time: query must return a column named time that returns either a SQL datetime or any numeric datatype representing unix epoch
2. metric(s): You may return a column named metric that is used as metric name for the value column. If you return multiple value columns and a column named metric then this column is used as prefix for the series name
3. value: Any column except time and metric are treated as a value column 
* In this case:
1. make key (e.g. runid) as 'time': think of this as timestamps on X-Axes although it's not as long as it's non-string type (so that when inverting from columns to rows. all data with same runid will be in one row)
2. timestamp as 'value' (so that it will become value after inverting)
3. takename as 'metric'(series name or header name after inverting)  
```
SELECT 
  S.runid AS time,
  S.timestamp AS value,
  concat_ws(' - ', S.taskname, S.status) AS metric
FROM stateinfo S
``` 
![alt](https://github.com/kangli914/grafana/blob/master/pic/table_col2row2.png "table2")
Full Query
```
D: 'Action' column:
SELECT 
  S.runid AS time,
  S.timestamp AS value,
  concat_ws(' - ', S.taskname, S.status) AS metric
FROM stateinfo S
WHERE $__unixEpochFilter(S.timestamp / 1000)
  AND S.databag::jsonb -> 'request_details' ->> 'requesting_application' LIKE ${appcode} 
  AND S.databag::jsonb -> 'request_details' ->> 'provider_job_id' LIKE ${providerid}
  AND S.runid IN(${runid}) 
  AND S.status NOT LIKE '%STASHED%'
  AND S.status LIKE '%ACTION%'
GROUP BY S.runid, S.timestamp, concat_ws(' - ', S.taskname, S.status)
ORDER BY S.timestamp ASC

C: 'Sub Task' column:
SELECT 
  S.runid AS time,
  S.timestamp AS value,
  concat_ws(' - ', S.taskname, S.status) AS metric
FROM stateinfo S
WHERE $__unixEpochFilter(S.timestamp / 1000)
  AND S.databag::jsonb -> 'request_details' ->> 'requesting_application' LIKE ${appcode} 
  AND S.databag::jsonb -> 'request_details' ->> 'provider_job_id' LIKE ${providerid}
  AND S.runid IN(${runid}) 
  AND S.status NOT LIKE '%STASHED%'
  AND S.status NOT LIKE '%ACTION%'
GROUP BY S.runid, S.timestamp, concat_ws(' - ', S.taskname, S.status)
ORDER BY S.timestamp ASC

B: 'IngestRequest' column:
SELECT
  R.runid AS time,
  concat_ws(' - ', 'IngestRequest', '') AS metric,
  extract(epoch from to_timestamp(R.databags::jsonb -> 'request' ->> 'request_time', 'YYYY-MM-DD HH24:MI:SS.US')) * 1000 AS value
FROM runinfo R
WHERE $__unixEpochFilter(R.startdatetime / 1000)
  AND R.databags::jsonb -> 'request' ->> 'requesting_application' LIKE ${appcode} 
  AND R.databags::jsonb -> 'request' ->> 'provider_job_id' LIKE ${providerid} 
  AND R.runid IN(${runid}) 
  AND R.runstatus LIKE '%COMPLETE%'
GROUP BY time, value, metric
ORDER BY time ASC

A: 'Execution Date' column:
SELECT
  R.runid AS time,
  'Date' AS metric,
  extract(YEAR from to_timestamp(R.databags::jsonb -> 'request' ->> 'request_time', 'YYYY-MM-DD HH24:MI:SS.US'))*10000 + 
  extract(MONTH from to_timestamp(R.databags::jsonb -> 'request' ->> 'request_time', 'YYYY-MM-DD HH24:MI:SS.US'))*100 +
  extract(DAY from to_timestamp(R.databags::jsonb -> 'request' ->> 'request_time', 'YYYY-MM-DD HH24:MI:SS.US'))
  AS value
FROM runinfo R
WHERE $__unixEpochFilter(R.startdatetime / 1000)
  AND R.databags::jsonb -> 'request' ->> 'requesting_application' LIKE ${appcode} 
  AND R.databags::jsonb -> 'request' ->> 'provider_job_id' LIKE ${providerid} 
  AND R.runid IN(${runid}) 
  AND R.runstatus LIKE '%COMPLETE%'
GROUP BY time, value, metric
ORDER BY time ASC

```

### Graph (type): Using as Bar chart
Want to compare several data points in a BAR chat view:
![alt](https://github.com/kangli914/grafana/blob/master/pic/graph_bar.png "graphbar")
Tips:
* Use 'Series' for X-Axis:
![alt](https://github.com/kangli914/grafana/blob/master/pic/series.png "graphbar")
* Query below also include joining 2 tables on common key so that it can access info across tables:
```
FROM runinfo R JOIN stateinfo S ON R.runid = S.runid
```
* if use time series, be smart about use the 'time' (S.runid in this case) as key on X-Axes
```
SELECT
  S.runid AS time,
  '2018 August' AS metric,
  (S.timestamp - 1536249466000) / 1000 AS value
FROM stateinfo S
```
Full Query:
```
C: August
SELECT
  S.runid AS time,
  /*concat_ws(' | ', '2018 August', S.runid) AS metric,*/
  '2018 August' AS metric,
  (S.timestamp - 1536249466000) / 1000 AS value
FROM stateinfo S
WHERE /*$__unixEpochFilter(S.timestamp / 1000)
  AND */S.runid = 2230
  AND S.status LIKE '%ACTION_SUCCEEDED%'
GROUP BY time, value, metric
ORDER BY time ASC

B: January
SELECT
  R.runid AS time,
  /*concat_ws(' | ', '2019 January', R.runid) AS metric,*/
  '2019 January' AS metric,
  (S.timestamp - extract(epoch from to_timestamp(R.databags::jsonb -> 'request' ->> 'request_time', 'YYYY-MM-DD HH24:MI:SS.US')) * 1000) / 1000 AS value
FROM runinfo R JOIN stateinfo S ON R.runid = S.runid
WHERE /*$__unixEpochFilter(S.timestamp / 1000)
  AND */
  R.databags::jsonb -> 'request' ->> 'requesting_application' LIKE ${appcode} 
  AND R.databags::jsonb -> 'request' ->> 'provider_job_id' LIKE '%Perftestjob%'
  AND R.runid = 12104
  AND S.status LIKE '%ACTION_SUCCEEDED%'
GROUP BY time, value, metric
ORDER BY time ASC

A: March
SELECT
  R.runid AS time,
  /*concat_ws(' | ', '2019 March', R.runid) AS metric,*/
  '2019 March' AS metric,
  (S.timestamp - extract(epoch from to_timestamp(R.databags::jsonb -> 'request' ->> 'request_time', 'YYYY-MM-DD HH24:MI:SS.US')) * 1000) / 1000 AS value
FROM runinfo R JOIN stateinfo S ON R.runid = S.runid
WHERE /*$__unixEpochFilter(S.timestamp / 1000)
  AND */
  R.databags::jsonb -> 'request' ->> 'requesting_application' LIKE ${appcode} 
  AND R.databags::jsonb -> 'request' ->> 'provider_job_id' LIKE '%Perftestjob%'
  AND R.runid = 15410
  AND S.status LIKE '%ACTION_SUCCEEDED%'
GROUP BY time, value, metric
ORDER BY time ASC
```

### Build some nested template variables for look-up
Some times one template variable will need to chain into another template variable.
For example, out come of Runid variable will be dynamically populated based on the selections on 2 variables provider & code
![alt](https://github.com/kangli914/grafana/blob/master/pic/templatingV2.png "templating2")
```
select distinct runid from runinfo where $__unixEpochFilter(startdatetime / 1000) AND 
databags::jsonb -> 'request' ->> 'provider_job_id' like $providerid AND 
databags::jsonb -> 'request' ->> 'requesting_application' like $appcode
```
