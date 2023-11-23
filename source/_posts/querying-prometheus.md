---
title: Querying Prometheus
date: 2023-11-23 16:13:33
tags:
cover: "/covers/querying-prometheus.svg"
---

Explain PromQL in simple SQL that you are familiar with.

>This blog is work-in-progress. But you can still read and discuss it with me.

Planned content:
- select data
- operations
- format in storage
- joins, group and set operation
- get distributed
- extension on types and operations
- counter, histogram and summary

# Select data
Prometheus collects data in an unreliable way. The data process pipeline has considered and brings lots of "special logic" to make those unreliable data look reasonable and intuitive. But as the price, that logic might not be straightforward as a traditional SQL-based database. This section includes lookback,  null-handling, offset, interval.

Notice that those mechanisms always take effort together. To keep the explanation dry, other cases are ignored when explaining one (E.g., "offset" section assumes there is no "lookback").

## offset
This is the simplest one. Offset is used to "bias" the data's timestamp -- every data point in Prometheus has a related timestamp, as well as the query itself. In SQL, the table (Prometheus names it "metric") looks just like this:

```sql
CREATE TABLE metric (
ts, timestamp(3) NOT NULL,
    v, double,
    tag_1 string,
    tag_2 string,
    ...
    PRIMARY KEY (tag_1, tag_2, ...),
);
```
The `offset` provides a way to **push backward** (toward an earlier time) the time range of querying:
```sql
SELECT ... FROM metric WHERE ts >= (START - OFFSET) and ts < (END - OFFSET);
```
For example, if we query data between `2023-11-14T18:00:00Z` and `2023-11-14T22:00:00Z`, and with "1h" offset, the equal SQL would be something like below:


```sql
SELECT ... FROM metric WHERE ts >= '2023-11-14T17:00:00Z' AND ts < '2023-11-14T21:00:00Z'
```
Another point is the offset can be negative. In this case, you are just "minus a negative"

## interval
You might be wondering why the results from Prometheus step are aligned to the given resolution but not the data collection interval. Here is an example result part I get with `7s` resolution:

```plaintext
[   ...,
    [1700635385,"1"],
    [1700635392,"1"],
    [1700635399,"1"],
    [1700635406,"1"],
    [1700635413,"1"],
    ...
]
```

That's because Prometheus doesn't generate timestamp directly from the data's timestamp, but in a reversed way, where the timestamp in the result is determined first, then finds values suited for that timestamp and feeds them into calculate operators. The `interval` is one of the key parameters for determining the timestamps in result. It works very simply -- if we use a `for` loop to simulate this behavior, it would be
```cpp
for (ts = start; ts < end; ts += interval) {}
```

## lookback
Also, unlike SQL where filter filters the "exact" what you require, PromQL will take its liberty to find data out the range of your query and present them to you. This behavior is called "lookback", looking at a larger range and fetching data for calculation.
For example, if the current timestamp we are calculating is `1700000000` (`2023-11-14T22:13:20Z`), and lookback delta is '5m' (300s). The filter is not `WHERE ts = 1700000000`, but this:

```sql
SELECT ... FROM metric WHERE ts >= 1699999700 AND ts <= 1700000000;
```

That is, Prometheus will take up to 5 minutes of data into consideration when calculating one data point.

## null handling

TODO

## don't repeat
Though Prometheus will try its best to find data, but it won't grab a repeated dataset for calculation, where all the points are the same, i.e., it won't repeat on one point to fill absent timestamps.

