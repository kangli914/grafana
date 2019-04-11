# Grafana Work to build monitoring using Docker, Telegraf, Influxdb, Grafana and etc.


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

### Inverting table from column to row 
data are stored in different rows of table with same key (runid: 15390) but wanting to achieve to display in one row of table in grafana (columns --> row):
![alt](https://github.com/kangli914/grafana/blob/master/pic/table_col2row.png "table1")
Tips:
* make Time-Series Table with 'Options': Time series to columns (instead of rows)
* standards about 'time', 'value' and 'metric':
 - time: query must return a column named time that returns either a SQL datetime or any numeric datatype representing unix epoch
 - metric(s): You may return a column named metric that is used as metric name for the value column. If you return multiple value columns and a column named metric then this column is used as prefix for the series name
 - value: Any column except time and metric are treated as a value column 
* in this case:
 - make key (e.g. runid) as 'time' (so that when inverting from columns to rows. all data with same runid will be in one row)
 - timestamp as 'value' (so that it will become value after inverting)
 - takename as 'metric'(series name or header name after inverting)  
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