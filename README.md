# Elasticsearch Cluster Operational Review Checklist

A structured runbook for auditing the health, capacity, and hygiene of an Elasticsearch cluster. Work through the sections in order — each builds on the previous.

Most commands use the Dev Tools console syntax (`GET /_cat/...`). For shell execution, prefix with `curl -sk -u $ES_USER:$ES_PASS https://$ES_HOST:9200` or use an API key header.

---

## 1. Cluster Health & Baseline

Start here on every review. If the cluster is red or yellow, most other checks are secondary until that's resolved.

### 1.1 Overall cluster status

```
GET /_cluster/health
```

Look for: `status` (green/yellow/red), `number_of_nodes`, `unassigned_shards`, `initializing_shards`, `relocating_shards`, `active_shards_percent_as_number`, `task_max_waiting_in_queue_millis`.

Deeper breakdown per index (surfaces which indices are degraded):

```
GET /_cluster/health?level=indices&pretty
```

### 1.2 Pending tasks

Long-running or backed-up tasks indicate a struggling master.

```
GET /_cluster/pending_tasks
GET /_cat/pending_tasks?v
```

### 1.3 Current cluster settings (persistent + transient)

```
GET /_cluster/settings?include_defaults=false&flat_settings=true
```

Flag any transient settings still in place — these get lost on full cluster restart. Transient is deprecated in newer versions; anything there should be reviewed and either promoted to persistent or removed.

### 1.4 Version and license

```
GET /
GET /_license
GET /_xpack
```

Verify the license is current and all nodes run the same version (mixed-version clusters are only acceptable during rolling upgrades).

---

## 2. Node Inventory & Roles

### 2.1 Node list with roles

```
GET /_cat/nodes?v&h=name,ip,node.role,master,heap.percent,heap.current,heap.max,ram.percent,cpu,load_1m,load_5m,load_15m,disk.used_percent,version&s=name
```

Role column legend: `m` master-eligible, `d` data, `i` ingest, `r` remote_cluster_client, `l` ml, `t` transform, `v` voting_only, `c` coordinating_only (shown as `-`).

Checks:
- Dedicated master nodes should show only `m` (and ideally `v` for voting-only in large clusters).
- Data node heap should stay under 75% steady-state. If any node sits above 85%, it's at risk of circuit-breaker trips.
- No more than one master role active at a time; master-eligible count should be odd (3 or 5 typical).

### 2.2 Master node identification

```
GET /_cat/master?v
```

### 2.3 Node attributes (for shard allocation awareness)

```
GET /_cat/nodeattrs?v&s=node,attr
```

Useful for confirming zone/rack awareness tags are set correctly across data tiers.

### 2.4 Node stats — deeper view

```
GET /_nodes/stats/jvm,os,process,fs,thread_pool,breaker?human
```

Focus areas:
- **JVM heap**: `heap_used_percent` trend, `gc.collectors.old.collection_count` and `collection_time_in_millis` (frequent old GCs are a warning sign)
- **File descriptors**: `process.open_file_descriptors` vs `max_file_descriptors`
- **Circuit breakers**: any `tripped > 0` is a problem
- **Thread pools**: queued/rejected counts

---

## 3. Shard Allocation & Distribution

This is the core of your review given what you mentioned.

### 3.1 Shard count per node

```
GET /_cat/allocation?v&s=node
```

Columns to interpret: `shards`, `disk.indices`, `disk.used`, `disk.avail`, `disk.total`, `disk.percent`.

Red flags:
- Wildly uneven shard counts across nodes of the same tier (more than ~20% variance suggests allocation problems).
- `disk.percent` near the low watermark (default 85%) or high watermark (90%).
- Any `UNASSIGNED` row at the bottom with a non-zero shard count.

### 3.2 Unassigned shards — why?

```
GET /_cluster/allocation/explain
```

For a specific shard:

```
POST /_cluster/allocation/explain
{
  "index": "your-index-name",
  "shard": 0,
  "primary": true
}
```

Common decider reasons worth documenting: `NO` for `disk_threshold`, `awareness`, `filter`, `same_shard`, `max_retry` (retry exhaustion after repeated allocation failures — reset with `POST /_cluster/reroute?retry_failed=true`).

### 3.3 Oversharding check

The rule of thumb Elastic has published for years: **~20 shards per GB of JVM heap per data node**. Exceeding this significantly degrades cluster state overhead, master performance, and recovery time.

Calculate:

```
GET /_cat/nodes?v&h=name,node.role,heap.max,shards
GET /_cat/allocation?v&h=node,shards,disk.indices
```

If a node has 32GB heap and 800 shards, that's 25 shards/GB — getting close to the ceiling. Anything over 30 warrants remediation.

Also look at **shard size distribution**:

```
GET /_cat/shards?v&h=index,shard,prirep,state,docs,store,node&bytes=gb&s=store:desc
```

Target shard size: **20–50 GB** for time-series/logs, **10–30 GB** for search workloads. Flag:
- Primary shards > 50 GB (slow recovery, hot shards)
- Shards < 1 GB on non-rollover indices (waste of cluster state overhead)
- Empty shards (`docs=0`, `store` near-zero) — these still consume cluster state

### 3.4 Hot shard / hot node check

```
GET /_cat/shards?v&h=index,shard,prirep,node,docs,store&s=store:desc&bytes=gb
GET /_cat/indices?v&h=index,pri,rep,docs.count,store.size&s=store.size:desc&bytes=gb
```

If one index dominates the write/search load and its primary shards are all on two nodes, you have a hotspot. Check with:

```
GET /_nodes/stats/indices?human
```

Compare `indexing.index_total` and `search.query_total` across nodes — large imbalances point to hot shards.

### 3.5 Relocating and initializing shards

```
GET /_cat/recovery?v&active_only=true&h=index,shard,type,stage,source_node,target_node,files_percent,bytes_percent,translog_ops_percent
```

During a normal operational window these should be empty or resolving quickly. Persistent initialization indicates node flapping or allocation deciders blocking placement.

---

## 4. Index Hygiene

### 4.1 Full index inventory

```
GET /_cat/indices?v&h=health,status,index,pri,rep,docs.count,docs.deleted,store.size,pri.store.size,creation.date.string&s=store.size:desc&bytes=gb
```

Pay attention to:
- Yellow indices — usually `number_of_replicas` set higher than available nodes can host
- Red indices — missing primary shards, investigate immediately
- High `docs.deleted` ratio (>20% of `docs.count`) — indicates overdue force-merge or expensive update workload
- Very old indices with low query volume (candidates for deletion/archive)

### 4.2 Empty or near-empty indices

```
GET /_cat/indices?v&h=index,docs.count,store.size&s=docs.count:asc
```

Anything with zero documents should be questioned. Common causes: leftover from rollover, aborted reindex, template misconfiguration.

### 4.3 Replica configuration sanity

```
GET /_settings?filter_path=*.settings.index.number_of_replicas,*.settings.index.number_of_shards
```

Confirm replicas ≥ 1 on all production indices (single points of failure are unacceptable for federal workloads). Also flag indices with `number_of_shards` > number of data nodes — those can't distribute properly.

### 4.4 Index templates and component templates

```
GET /_index_template
GET /_component_template
GET /_cat/templates?v&s=name
```

Audit for:
- Legacy templates (`GET /_template`) that should be migrated to composable templates
- Overlapping priority values
- Templates with no matching indices (stale)
- Missing ILM policy references

### 4.5 Mapping explosion check

```
GET /<index>/_mapping
GET /<index>/_field_caps?fields=*
```

For any index with many dynamic fields, check `index.mapping.total_fields.limit` (default 1000) against actual field count. Approaching the limit causes rejection and often means dynamic mapping should be disabled or `dynamic: runtime` applied.

---

## 5. ILM, Rollover, and Data Streams

### 5.1 ILM policy overview

```
GET /_ilm/policy
GET /_ilm/status
```

`ilm_status` should be `RUNNING`. `STOPPED` or `STOPPING` means lifecycle transitions are paused.

### 5.2 Indices with ILM errors

```
GET /*/_ilm/explain?only_errors=true&filter_path=indices.*.step,indices.*.step_info,indices.*.failed_step
```

Any index surfaced here is stuck in a lifecycle step — common causes: rollover alias missing, insufficient nodes for shrink, ILM policy referencing a non-existent snapshot repository.

### 5.3 Rollover aliases

```
GET /_cat/aliases?v&s=alias
```

For rollover indices, confirm `is_write_index` is set on exactly one index per alias.

### 5.4 Data streams

```
GET /_data_stream
GET /_data_stream/*/_stats?human
```

Review backing index counts, generation numbers, and store size per stream.

---

## 6. Snapshots & Recovery

### 6.1 Registered repositories

```
GET /_snapshot/_all
POST /_snapshot/<repo>/_verify
```

Verify should succeed on every node. A failure here means the repo is effectively broken for that node.

### 6.2 SLM policies and recent executions

```
GET /_slm/policy
GET /_slm/stats
```

Check `snapshots_taken`, `snapshots_failed`, `policy.last_success`, `policy.last_failure`. Any policy where `last_failure` is more recent than `last_success` needs investigation.

### 6.3 Most recent snapshots per repo

```
GET /_snapshot/<repo>/_all?verbose=false
```

Confirm cadence matches RPO requirements. For FedRAMP/FISMA environments, document retention against the SSP.

---

## 7. Security & Access Governance

This overlaps with the API key inventory work you've already done, but belongs in a review.

### 7.1 Users and roles

```
GET /_security/user
GET /_security/role
GET /_security/role_mapping
```

### 7.2 API key inventory

```
GET /_security/api_key?owner=false
```

Flag: keys without expiration, keys with cluster-wide `all` privileges, keys owned by departed staff, keys unused in > 90 days (cross-reference `creation` date with audit logs).

### 7.3 Audit log status

```
GET /_cluster/settings?include_defaults=true&filter_path=*.xpack.security.audit*
```

For federal workloads, `xpack.security.audit.enabled: true` should be persistent, with appropriate `include`/`exclude` events documented.

### 7.4 TLS / transport encryption

```
GET /_ssl/certificates
```

Review expiration dates. Anything expiring in < 90 days should be on the renewal backlog.

---

## 8. Performance & Workload Indicators

### 8.1 Hot threads (only when investigating a specific issue)

```
GET /_nodes/hot_threads?threads=5&interval=500ms
```

### 8.2 Search and indexing stats

```
GET /_nodes/stats/indices/search,indexing,refresh,flush,merge?human
```

Look at `refresh.total_time_in_millis`, `flush.total_time_in_millis`, `merges.current`, `merges.total_stopped_time_in_millis`. Stopped merges indicate throttling under indexing pressure.

### 8.3 Thread pool rejections

```
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected,completed&s=rejected:desc
```

Any non-zero `rejected` for `write`, `search`, or `get` pools warrants follow-up. Persistent rejections mean the cluster is under-provisioned for the workload or a bulk client is misbehaving.

### 8.4 Field data and query cache

```
GET /_nodes/stats/indices/fielddata,query_cache,request_cache?human
```

High `evictions` on `query_cache` or `request_cache` suggests cache size pressure; high `fielddata.memory_size_in_bytes` on text fields (without `keyword` subfield use) is a classic anti-pattern.

---

## 9. Cross-Cluster Search (CCS) — if applicable

Given your CCS Health Check project for 2026, the review should include:

### 9.1 Remote cluster connectivity

```
GET /_remote/info
```

Check `connected: true` for every remote, `num_nodes_connected` matches expectation, and `initial_connect_timeout` / `skip_unavailable` are set per your standards.

### 9.2 CCS query test

```
GET /remote_cluster_name:index-pattern-*/_search?size=0
```

A failure here (while `_remote/info` shows connected) usually indicates a role-mapping or remote-index-privilege gap.

### 9.3 Role privileges for CCS

```
GET /_security/role/<role_name>
```

Confirm the role includes `remote_indices` privileges where expected.

---

## 10. Logging, Deprecations & Upgrade Readiness

### 10.1 Deprecation log

```
GET /_migration/deprecations
```

Resolve every `critical` item before the next major upgrade. `warning` items should be tracked and scheduled.

### 10.2 Cluster-level issues

Review anything flagged under `cluster_settings`, `node_settings`, `index_settings`, and `ml_settings` in the deprecations output.

### 10.3 Feature usage

```
GET /_xpack/usage
```

Helpful for confirming which features (Watcher, ML, Transforms, Frozen tier) are actually in use versus licensed-but-dormant.

---

## 11. Documentation Deliverables

At the end of the review, the output artifacts should include:

- Cluster health summary (status, node count, shard count, total store size)
- Oversharding assessment (shards-per-GB-heap per node, with remediation recommendations)
- List of unassigned / red / yellow indices with root cause
- List of empty or undersized shards flagged for deletion or rollover tuning
- ILM policies with errors or stuck indices
- SLM policies with recent failures
- Expiring TLS certs and unused / stale API keys
- Deprecation warnings sorted by severity
- Capacity projection: days-to-full based on current disk trend (pull from Monitoring or compute from `/_cat/allocation` snapshots over time)

---

## Appendix A: One-shot Health Snapshot

For a quick "is anything on fire" check, run these six in sequence:

```
GET /_cluster/health
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,disk.used_percent
GET /_cat/allocation?v
GET /_cluster/allocation/explain
GET /_cat/pending_tasks?v
GET /_cat/thread_pool?v&h=node_name,name,active,queue,rejected&s=rejected:desc
```

## Appendix B: Useful `_cat` Flags

- `?v` — include column headers
- `?help` — list all available columns for that endpoint
- `?h=col1,col2` — show only specific columns
- `?s=col:desc` — sort
- `?bytes=gb` / `?bytes=mb` — force byte units (default is human-readable mixed)
- `?format=json` — machine-readable output for scripting
