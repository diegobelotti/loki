---
title: "XXXX: Index Gateway Sharding"
---

# Index Gateway Sharding

**Author:** Christian Haudum (christian.haudum@grafana.com)

**Date:** 01/2023

**Sponsor(s):** @chaudum

**Type:** Feature

**Status:** Draft

**Related issues/PRs:**

**Thread from [mailing list](https://groups.google.com/forum/#!forum/lokiproject):**

---

## Background

This document tries to come up with a proposal on how to do a better sharding of data on the index gateways so we are able to scale the service horizontally to fulfill the increased need for metadata queries of big tenants.

The index gateway service can be run in "simple mode", where an index gateway instance is responsible for handling, storing and returning requests for all indices for all tenants, or in "ring mode", where an instance is responsible for a subset of tenants instead of all tenants.

On top of that, in order to achieve redundancy as well as spreading load, the index gateway ring uses a replication factor of 3.

This means, before an index gateway client makes a request to the index gateway server, it first hashes the tenant ID and then requests a replication set for that hash from the index gateway ring. Due to the fixed replication factor (RF), the replication set contains three server addresses. On every request, a random server from that list is picked to then execute the request on.

## Problem Statement

The current strategy of sharding by tenant ID and having a replication factor of 3 fails in the long run, because even when running lots of index gateways, only a maximum of three could be utilized by a single tenant. 

Another problem is that the RF is fixed and the same for all tenants, independent of their actual size in terms of log volume or query rate.

## Goals

The goal of this document is to find a better sharding mechanism for the index gateway, so that there are no boundaries for scaling the service horizontally.

* The sharding needs to account for the "size" of a tenant.
* A single tenant needs to be able to utilize more than three index gateways.

## Proposals

### Proposal 0: Do nothing

If we do not improve the sharding mechanism for the index gateways and leave it as it is, it will become more and more difficult to serve metadata queries for large tenant in a reasonable amount of time, proportionally to the demand for these queries.

### Proposal 1: Dynamic replication factor

Instead of using a fixed replication factor of 3, the RF can be derived from the amount of active members in the index gateway ring. That means that the RF would be a percentage of the available gateway instances. For example, a ring with 12 instances and 30% replication utilization would result in a RF of 3 (`floor(12*0.3)`). Scaling up to 18 instances would result in a RF of 5.

This approach would solve the problem of horizontal scaling. However, it does not solve the problem of different tenant sizes. It also fails to ensure replication for availability when running a small number of instances, unless there is a fixed lower value for the RF.

### Proposal 2: Shard by matchers and per-tenant sharding factor

To achieve the goal of distributing metadata queries of a single tenant across multiple index gateways requires to use a different hashing than just the tenant ID. On the one hand, the hash key needs to be deterministic for the same query over time, on the other hand, the resulting hash keys must not spread queries in a way, that data that is downloaded on the index gateways when queries are executed, ends up redundantly on different servers. So while the current sharding solution with a RF of 3 provides full redundancy on 3 instances, the proposal needs to aim for partial redundancy across a subset of index gateway instances.

Considerations for choosing an appropriate value for the hash key function:

* The hash key of the tenant ID guarantees deterministic targeting of the index gateway server assuming the token assignment in the ring does not change.
* Adding the label matchers from the query would result in too many different hash key, resulting in queries to be spread out across all available index gateway instances and therefore high data redundancy. However, the hash key is deterministic.
* Adding a random shard ID (e.g. `shard-0`, `shard-1`, ... `shard-n`) to the tenant ID allows to utilize a certain amount of `n` instances. The amount of shards can be implemented as a per-tenant override. However, this approach results in non-deterministic hash keys.

Out of these considerations, a combination of them like this would be a suitable hashing mechanism:

```
HASH($tenant + HASH($matchers) % $sharding_factor)
```



## Other Notes
