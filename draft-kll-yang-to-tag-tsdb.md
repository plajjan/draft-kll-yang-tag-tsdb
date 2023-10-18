---
stand_alone: true
title: Encoding YANG-Modeled Data for Storage in Tag-Centric Time Series Databases
category: standards track
workgroup: 

lang: en
kw:
  - Internet-Draft

pi:
  - toc
  
normative:

informative:

author:
-   ins: K. Larsson
    name: Kristian Larsson
    organization: Deutsche Telekom
    email: kll@dev.terastrm.net
    
contributor:

--- abstract 

This document proposes a standardized approach for representing YANG-modeled configuration and state data, for storage in Time Series Databases (TSDBs) that utilize a tag-centric model. It outlines procedures for translating YANG data representations to fit within the tag-centric structures of TSDBs and vice versa. This mapping ensures clear and efficient storage and querying of YANG-modeled data in TSDBs.

--- middle

# Introduction

The aim of this document is to define rules for representing configuration and state data defined using the YANG data modeling language [RFC7950] as time series using a tag centric model.

The majority of modern Time Series Databases (TSDBs) employ a tag-centric model. In this structure, data points are indexed primarily by tags, each consisting of a key-value pair. These tags facilitate efficient querying, aggregation, and filtering of data over time intervals. Such a model contrasts with the hierarchical nature of YANG-modeled data. The challenge, therefore, lies in ensuring that YANG-defined data, with its inherent structure and depth, can be seamlessly integrated into the flat, tag-based structure of most contemporary TSDBs.

This document seeks to bridge this structural gap, laying out rules and guidelines to ensure that YANG-modeled configuration and state data can be effectively stored, queried, and analyzed within tag-centric TSDBs.


# Specification of the Mapping Procedure

Instances of YANG data nodes are mapped to metrics. Only nodes that carry a value are mapped. This includes leafs and presence containers. The hierarchical path to a value, including non-presence containers and lists, form the path that is used as the name of the metric. The path is formed by joining YANG data nodes using `_`.

List keys are mapped into tags. The path to the list key is transformed in the same way as the primary name of the metric. Compound keys have each key part as as separate tag.

## Example: Packet Counters in IETF Interfaces Model

Consider the `in-unicast-pkts` leaf from the IETF interfaces model that captures the number of incoming unicast packets on an interface:

Original YANG Instance-Identifier:
```yang
/interfaces/interface[name='eth0']/statistics/in-unicast-pkts
```

Following the mapping rules defined:

1. The path components, including containers and list names, are transformed into the metric name by joining the node names with `_`. Special symbols, e.g. `-` are replaced with `_`.

Resulting Metric Name:
```
interfaces_interface_statistics_in_unicast_pkts
```

2. The list key "predicate", which in this case is the interface name (`eth0`), is extracted and stored as a separate label. The label key represents the complete path to the key.

Resulting Label:
```
interfaces_interface_name = eth0
```

3. The leaf value, which represents the actual packet counter, remains unchanged and is directly mapped to the value in the time series database.

For instance, if the packet counter reads `5,432,100` packets:

Value:
```
5432100
```

4. As part of the standard tags, a server identification string is also included. A typical choice of identifier might be the hostname. For this example, let's assume the device name is `router-01`:

Tag:
```
host = router-01
```

Final Mapping in the TSDB:

- Metric: `interfaces_interface_statistics_in_unicast_pkts`
- Value: `5432100`
- Tags:
  - `host` = `router-01`
  - `interfaces_interface_name` = `eth0`

## Mapping values

Leaf values are mapped based on their intrinsic type:

- All integer types are mapped to integers and retain their native representation
  - some implementations only support floats for numeric values
- decimal64 values are mapped to floats and the value should be rounded and truncated as to minimize the loss of information
- Enumeration types are mapped using their string representation.
- String types remain unchanged.

## Choice

Choice constructs from YANG are disregarded and not enforced during the mapping process. Given the temporal nature of TSDBs, where data spans across time, different choice branches could be active in a single data set, rendering validation and storage restrictions impractical.

## Host / device name

There is an implicit `host` tag identifying the server, typically set to the name of the host originating the time series data.

Instance data retrieved from YANG-based servers do not generally identify the server it originates from. As a time series database is likely going to contain data from multiple servers, the `host` tag is used to identify the source of the data.


# Querying YANG modeled time series data

The process of storing YANG-modeled data in tag-centric TSDBs, as defined in the previous sections, inherently structures the data in a way that leverages the querying capabilities of modern TSDBs. This chapter provides guidelines on how to construct queries to retrieve this data effectively.

## 1. **Basic Queries**

To retrieve all data points related to incoming unicast packets from the IETF interfaces model:

- **InfluxQL**:
  ```sql
  SELECT * FROM interfaces_interface_statistics_in_unicast_pkts
  ```

- **PromQL**:
  ```promql
  interfaces_interface_statistics_in_unicast_pkts
  ```

## 2. **Filtering by Tags**

To retrieve incoming unicast packets specifically for the interface `eth0`:

- **InfluxQL**:
  ```sql
  SELECT * FROM interfaces_interface_statistics_in_unicast_pkts WHERE interfaces_interface_name = 'eth0'
  ```

- **PromQL**:
  ```promql
  interfaces_interface_statistics_in_unicast_pkts{interfaces_interface_name="eth0"}
  ```

Similarly, to filter by device / host name:

- **InfluxQL**:
  ```sql
  SELECT * FROM interfaces_interface_statistics_in_unicast_pkts WHERE host = 'router-01'
  ```

- **PromQL**:
  ```promql
  interfaces_interface_statistics_in_unicast_pkts{host="router-01"}
  ```

## 3. **Time-based Queries**

- **InfluxQL**:
  ```sql
  SELECT * FROM interfaces_interface_statistics_in_unicast_pkts WHERE time > now() - 24h
  ```

Prometheus fetches data based on the configured scrape interval and retention policies, so time-based filters in PromQL often center around the range vectors. For data over the last 24 hours:

- **PromQL**:
  ```promql
  interfaces_interface_statistics_in_unicast_pkts[24h]
  ```

## 4. **Aggregations**

To get the average number of incoming unicast packets over the last hour:

- **InfluxQL**:
  ```sql
  SELECT MEAN(value) FROM interfaces_interface_statistics_in_unicast_pkts WHERE time > now() - 1h GROUP BY time(10m)
  ```

- **PromQL**:
  ```promql
  avg_over_time(interfaces_interface_statistics_in_unicast_pkts[1h])
  ```

## 5. **Combining Filters**

To retrieve the sum of incoming unicast packets for `eth0` on `router-01` over the last day:

- **InfluxQL**:
  ```sql
  SELECT SUM(value) FROM interfaces_interface_statistics_in_unicast_pkts WHERE interfaces_interface_name = 'eth0' AND host = 'router-01' AND time > now() - 24h
  ```

- **PromQL**:
  ```promql
  sum(interfaces_interface_statistics_in_unicast_pkts{interfaces_interface_name="eth0", host="router-01"})[24h]
  ```

## 6. **Querying Enumeration Types**

In YANG models, enumerations are defined types with a set of named values. The `oper-status` leaf in the IETF interfaces model is an example of such an enumeration, representing the operational status of an interface.

For instance, the `oper-status` might have values such as `up`, `down`, or `testing`.

To query interfaces that have an `oper-status` of `up`:

- **InfluxQL**:
  ```sql
  SELECT * FROM interfaces_interface_oper_status WHERE value = 'up'
  ```

- **PromQL**:
  ```promql
  interfaces_interface_oper_status{value="up"}
  ```

Similarly, to filter interfaces with `oper-status` of `down`:

- **InfluxQL**:
  ```sql
  SELECT * FROM interfaces_interface_oper_status WHERE value = 'down'
  ```

- **PromQL**:
  ```promql
  interfaces_interface_oper_status{value="down"}
  ```

This approach allows us to effectively query interfaces based on their operational status, leveraging the enumeration mapping within the TSDB.

# Requirements on time series databases

This document specifies a mapping to a conceptual representation, not a particular concrete interface. To effectively support the mapping of YANG-modeled data into a tag-centric model, certain requirements must be met by the Time Series Databases (TSDBs). These requirements ensure that the data is stored and retrieved in a consistent and efficient manner.

## Support for String Values

Several YANG leaf types carry string values, including the `string` type itself and all its descendants as well as enumerations which are saved using their string representation.

The chosen TSDB must support the storage and querying of string values. Not all TSDBs inherently offer this capability, and thus, it's imperative to ensure compatibility.

## Sufficient Path Length

YANG data nodes, especially when representing deep hierarchical structures, can result in long paths. When transformed into metric names or labels within the TSDB, these paths might exceed typical character limits imposed by some databases. It's essential for the TSDB to accommodate these potentially long names to ensure data fidelity and avoid truncation or loss of information.

## High Cardinality

Given the possibility of numerous unique tag combinations (especially with dynamic values like interface names, device names, etc.), the chosen TSDB should handle high cardinality efficiently. High cardinality can impact database performance and query times, so it's essential for the TSDB to have mechanisms to manage this efficiently.
