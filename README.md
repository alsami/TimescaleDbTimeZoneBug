# Potential bug with time-zones

## Setup

### Create the database

```sql
CREATE TABLE IF NOT EXISTS TimeZoneBug (
    id VARCHAR(100) NOT NULL,
    timestamp_utc TIMESTAMP NOT NULL,
    i_value BIGINT,
    PRIMARY KEY(id, timestamp_utc)
);

SELECT create_hypertable('TimeZoneBug', 'timestamp_utc', chunk_time_interval => interval '1 week', if_not_exists => TRUE);

CREATE INDEX IF NOT EXISTS TimeZoneBug_compound_idx on TimeZoneBug (id, timestamp_utc DESC, i_value);
```

### Import data

You can import the data using the `data.csv` file.

## Reproduction

Below you can see two queries, that are only different in the bucket-size.
The datapoint at `2022-11-14 23:00:00.000000 +00:00` within the bucket of `12 hours` will result to `3132`. Whereas the datapoint at `2022-11-14 23:00:00.000000 +00:00` within the bucket of `1 day` will result to `3127`.

```sql
SELECT
	id,
	time_bucket('12 hours', timestamp_utc::timestamptz, 'Europe/Vienna') as time,
	max(i_value) AS i_value
FROM
	public.TimeZoneBug
WHERE
	timestamp_utc BETWEEN '2022-10-17 23:00:00' AND '2022-11-17 23:00:00' AND
	i_value IS NOT NULL AND
	id IN (
		'FE:C7:A5:36:EF:E9:AB:36:20480:0'
)
GROUP BY
	id,
	time
ORDER BY
	id,
	time;

SELECT
	id,
	time_bucket('1 day', timestamp_utc::timestamptz, 'Europe/Vienna') as time,
	max(i_value) AS i_value
FROM
	public.TimeZoneBug
WHERE
	timestamp_utc BETWEEN '2022-10-17 23:00:00' AND '2022-11-17 23:00:00' AND
	i_value IS NOT NULL AND
	id IN (
		'FE:C7:A5:36:EF:E9:AB:36:20480:0'
)
GROUP BY
	id,
	time
ORDER BY
	id,
	time;
```

When using the UTC values, meaning no time-zone provided as part of the query, the data-point at `2022-11-14 12:00:00.000000 +00:00` will be `3120` for within a bucket of `12 hours`. Now given the same query but for a bucket of `1day`, the data-point at `2022-11-14 12:00:00.000000 +00:00` will be also `3120`. 

```sql
SELECT
	id,
	time_bucket('12 hours', timestamp_utc::timestamptz) as time,
	max(i_value) AS i_value
FROM
	public.TimeZoneBug
WHERE
	timestamp_utc BETWEEN '2022-10-17 23:00:00' AND '2022-11-17 23:00:00' AND
	i_value IS NOT NULL AND
	id IN (
		'FE:C7:A5:36:EF:E9:AB:36:20480:0'
)
GROUP BY
	id,
	time
ORDER BY
	id,
	time;


SELECT
	id,
	time_bucket('1 day', timestamp_utc::timestamptz) as time,
	max(i_value) AS i_value
FROM
	public.TimeZoneBug
WHERE
	timestamp_utc BETWEEN '2022-10-17 23:00:00' AND '2022-11-17 23:00:00' AND
	i_value IS NOT NULL AND
	id IN (
		'FE:C7:A5:36:EF:E9:AB:36:20480:0'
)
GROUP BY
	id,
	time
ORDER BY
	id,
	time;
```