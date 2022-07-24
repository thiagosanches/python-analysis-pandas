# python-analysis-pandas-postgre

# Pandas

Attempt to run some analysis using pandas. 

Take a look on [main.py](./main.py) file.

# PostgreSQL

Attempt to run some analysis using PostgreSQL.

After you have created the container, all the sub-sequent commands are assuming that you are running inside the container. To run inside the container: `docker exec -it <container_id> bash`.

### Create a postgresql container

```bash
docker run -d \
--name some-postgres \
-p 5432:5432 \
-e POSTGRES_PASSWORD=forabozo \
-e PGDATA=/var/lib/postgresql/data/pgdata \
-v /custom/mount:/var/lib/postgresql/data \
postgres
```

### Create a table that you want to import the data (csv)

```sql
CREATE TABLE data (
  id SERIAL,
  date DATE,
  time VARCHAR(255),
  request_ip VARCHAR(255),
  PRIMARY KEY (id)
)
```

### Import the CSV into the previous created table
This file can be huge!

You have to copy the `filtered-data.csv` into the mapped volume (`/custom/mount/pgdata/`).

It took less than 30 minutes to import a 4.5GB files with 86 million lines.

```sql
COPY data(date, time, request_ip)
FROM 'filtered-data.csv'
DELIMITER ','
CSV HEADER;
```
You can follow the operation using: `select * from pg_stat_progress_copy \watch 1`.

### Concatenate (if needed) date + time to create a timestamp column

```sql
SELECT request_ip, COUNT(request_ip) request_ip, 
to_timestamp(floor((extract('epoch' from TO_TIMESTAMP(concat(date, concat(' ', time)), 'YYYY/MM/DD/HH24:MI:ss')) / 300 )) * 300) AT TIME ZONE 'UTC' as interval_alias
FROM data GROUP BY request_ip, interval_alias
ORDER BY interval_alias ASC
LIMIT 5
```
You can use the `LIMIT 5` just to see if your data is fine.


### Export the previous query into a csv file

```sql
COPY (
    SELECT request_ip, COUNT(request_ip) request_ip, 
    to_timestamp(floor((extract('epoch' from TO_TIMESTAMP(concat(date, concat(' ', time)), 'YYYY/MM/DD/HH24:MI:ss')) / 300 )) * 300) AT TIME ZONE 'UTC' as interval_alias
    FROM data GROUP BY request_ip, interval_alias
    ORDER BY interval_alias ASC
) TO '/tmp/resampled-data-2.csv' (format csv, delimiter ';');
```

The file will be saved on `/tmp/resampled-data-2.csv`, you have to move to `/var/lib/postgresql/data/pgdata`.
