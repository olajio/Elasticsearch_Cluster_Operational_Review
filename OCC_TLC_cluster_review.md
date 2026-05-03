# OCC TLC Cluster Operational Review

**Cluster:** `2b3cd0da437d4742a1046c05537ea3c9` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 3 total nodes — 2 combined-role nodes (`himrst`: hot+ingest+master+remote+search+transform, **1.7 GB heap**, 320 GB disk each) + 1 voting-only tiebreaker (`mv`, 224 MB heap). 66 primaries / 132 total shards / 0 unassigned.

**This is a small all-in-one deployment with significant operational pressure.** The cluster is GREEN and functioning, but several metrics indicate it's running close to or beyond recommended limits. Let me walk through it.

---

## 1. Oversized shards & resharding recommendations

Using a 40 GB target (10–30 GB read-heavy / 30–50 GB write-heavy), here's the per-primary picture:

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `.ds-elastic-cloud-logs-8-2026.04.02-000001` | 1 | 45 GB | **45.0** | ⚠️ at write-heavy ceiling |
| `.ds-.monitoring-es-8-mb-2026.04.29-000017` | 1 | 2 GB | 2.0 | OK |
| `.ds-elastic-cloud-logs-8-2026.05.02-000002` | 1 | 2 GB | 2.0 | active backing index |
| `.ds-.monitoring-es-8-mb-2026.05.02-000019` | 1 | 1 GB | 1.0 | active backing index |
| `cdm_cpe_dictionary` | 1 | <1 GB | <1 | OK |
| `.ds-metricbeat-8.19.14-2026.04.15-000001` | 1 | <1 GB | <1 | OK |
| `.ds-metricbeat-8.19.13-2026.04.02-000001` | 1 | <1 GB | <1 | OK |
| `.ds-.monitoring-kibana-8-mb-*` | 1 | 0 GB | 0 | small monitoring data |
| Everything else | 1 | 0 GB | 0 | empty |

**The 45 GB shard on `.ds-elastic-cloud-logs-8-2026.04.02-000001` is right at the edge of the write-heavy ceiling.** It's not technically oversized by the 40 GB target rule (write-heavy can go to 50 GB), but combined with the heap pressure on these 1.7 GB nodes (see §5), this single shard is consuming a disproportionate fraction of cluster resources. The fact that the same primary count is used on the new active backing index (`-2026.05.02-000002`) suggests the rollover policy is keyed on time, not size — meaning if traffic increases, the next backing index could exceed 50 GB.

**Recommendation:** Don't reshard the existing index — that would be expensive and the index is already past its active write window. But review the index template / ILM rollover policy for `elastic-cloud-logs-8`. Consider:
- Adding a `max_primary_shard_size` rollover condition (e.g., 30 GB) to keep new backing indices smaller.
- Verifying the rollover frequency is appropriate — the gap from `2026-04-02` → `2026-05-02` is 30 days, which is a long window for a high-volume log datastream on small nodes.

For everything else, primary counts are appropriate (single primary for small indices). No resharding warranted.

## 2. Is the cluster oversharded?

**Yes, significantly.** This is the standout finding for this cluster.

- 132 total shards across 2 hot nodes = **66 shards per hot node**.
- Heap on each hot node: 1.7 GB.
- Ceiling at the 20-shards-per-GB-heap rule: **~34 shards per node**.
- **Current utilization: ~194% of the recommended limit.**

That's not "approaching" the threshold — the cluster is operating at nearly twice the recommended shard density per heap. This is consistent with the heap pressure observed on `instance-0000000000` (71% heap used) and the elevated RAM utilization (93% and 98% on the two hot nodes).

The shard count breaks down as:
1. **Kibana system indices** — `.kibana_*`, `.security*`, `.tasks`, `.transform-*`, `.async-search`, `.inference`, `.kibana_alerting_cases`, `.kibana_security_session`, `.kibana_security_solution`, `.kibana_locks`, etc. = ~14 shard pairs = ~28 shards.
2. **Kibana Alerting framework** — full `.internal.alerts-*-default-000001` family with rep=1 = ~30 shards.
3. **Kibana SIEM rule migrations / APM / SLO** — ~16 shards.
4. **ILM/SLM/event-log/deprecation log datastream backing indices** — ~30+ shards spread across multiple weeks.
5. **Elastic Cloud Logs and Monitoring datastreams** — the actual data-bearing ones, ~10 shards.
6. **Legacy `.monitoring-es-7-*` and `.monitoring-kibana-7-*`** — the old Stack Monitoring v7 indices = 4 shards.
7. **`cdm_cpe_dictionary`** — 2 shards.

Most of these are tiny or empty system indices, but **every single one occupies a shard slot on these undersized nodes**. The cluster could comfortably halve its shard count by deleting empty system indices that aren't strictly needed (see §3) — but the more durable fix is sizing.

**The recommended remediation is to scale up the hot nodes.** Doubling heap to 3.4 GB raises the ceiling to ~68 shards/node, which would put you back inside guidance. A 4 GB or 8 GB heap node tier would give meaningful margin.

## 3. Empty indices & their sources

There are roughly **30+ empty indices** in the cluster. They break down by origin:

**Kibana Alerting framework (15 indices)** — full `.internal.alerts-*-default-000001` family covering security, ML anomaly detection (×2), observability (metrics/logs/uptime/threshold/slo/apm), streams, stack, transform health, dataset quality, attack discovery, default. Auto-created by Kibana 2026-04-01, expected to be empty until rules fire. Leave them alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. Empty unless actively running a SIEM migration.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Empty until APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Empty if no SLOs are defined.

**Recently-rolled-over backing indices (1 index)** — `.ds-metricbeat-8.19.13-2026.05.02-000002`. Created today, no docs yet — normal.

**Legacy v7 monitoring (2 indices)** — `.monitoring-es-7-2026.04.01`, `.monitoring-kibana-7-2026.04.01`. Single-day captures from the day the cluster was deployed (2026-04-01), then Stack Monitoring presumably switched to the v8 datastream format. These are vestigial. **Candidate for deletion.**

**Other:** `.kibana_locks-000001`. No `.elastic-connectors*` or `.ent-search-*` indices — good, this cluster skipped Enterprise Search.

**No CDM application data.** Unlike the other deployments reviewed, this cluster has *no* CDM application indices at all (no `cdm_vuln`, `cdm_cyhy_*`, `cdm_scuba_*`, etc.). The only `cdm_*` index is `cdm_cpe_dictionary` (the CPE seed dictionary). This is a different kind of deployment than the other CDM-focused clusters — it appears to be primarily a logging/monitoring deployment with `cdm_cpe_dictionary` present perhaps for cross-cluster reference.

**Frozen tier:** None. No frozen nodes, no `partial-*` shards. Two-node hot cluster only (plus the voting tiebreaker).

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index has `rep=1`. There are no `partial-*` searchable-snapshot mounts (which is correct — there's no frozen tier here).

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This cluster has multiple yellow-flag indicators that combined suggest it's running too lean. Walking through them:

**Heap pressure on the elected master.** `instance-0000000000` (the elected master, also serving as a hot data node — `himrst` combined role) shows 71% heap used (1.2 GB of 1.7 GB). The other `himrst` node is at 52%. The voting-only tiebreaker is at 76% of its tiny 224 MB heap. With 30% headroom on a 1.7 GB heap, GC pressure is likely on the edge of becoming user-visible. A heap dump under the active write load is probably already doing 300-500 MB worth of work per major GC.

**RAM exhaustion.** Both hot nodes are at **93% and 98% RAM utilization**. This is the most concerning metric in the report. While Elasticsearch JVMs allocate a fixed heap and the rest is OS file cache (which is healthy to be high), 98% RAM on `instance-0000000001` leaves essentially no margin for a memory spike. If the OS hits memory pressure and starts evicting file cache aggressively, query latency will degrade rapidly. This is consistent with the over-shard-density observation — many small shards each consume some Lucene memory overhead even when they're empty.

**Tiebreaker node CPU is sustained-high.** `tiebreaker-0000000002` shows `load_1m=5.20, load_5m=5.46, load_15m=5.87` on a 224 MB heap. Voting-only nodes shouldn't be doing meaningful work — they exist purely for split-brain prevention. Sustained load of 5+ on a tiebreaker is unusual and worth diagnosing. Could be system noise (the load metric reflects the entire instance, not just Elasticsearch), or could indicate the tiebreaker is thrashing on something. If it's the host showing load (not the JVM), this is an ESS cloud-side issue.

**Active write load is real.** Both hot nodes show `write_load.forecast = 0.0078` — small in absolute terms, but non-trivial for nodes this size. The active backing indices (`.ds-elastic-cloud-logs-8-2026.05.02-000002` at 4.9M docs ingested in ~24 hours, `.ds-.monitoring-es-8-mb-2026.05.02-000019` at 1.5M docs) confirm steady write traffic. The CPU on `instance-0000000000` shows 14% — also consistent with active indexing.

**The 45 GB closed-window backing index is sitting on a 1.7 GB heap node.** `.ds-elastic-cloud-logs-8-2026.04.02-000001` (94M docs, 45 GB primary) is a closed write window — no longer being written to — but it still has its segments cached and indexed for query. With both replica and primary present, that's 90 GB of total disk on a 320 GB node (~28% of capacity for one shard pair). On a 1.7 GB heap, the off-heap memory needed to keep this shard's segments query-ready is non-trivial and likely contributing to the RAM pressure.

**Disk usage is healthy.** Both hot nodes at ~16.9% (54 GB used of 320 GB). Plenty of disk headroom — the constraint here is heap and shard count, not storage.

**ILM and SLM are active.** Multiple `.ds-ilm-history-7-*` and `.ds-.slm-history-7-*` backing indices going back to 2026-04-01. ILM is rolling things over and snapshot lifecycle is producing history records — both are healthy signals.

**No master-only nodes.** This deployment uses the combined `himrst` role for the data nodes and a single voting-only tiebreaker. That means master-eligible work shares the JVM with data work, which is fine for small deployments but contributes to the heap pressure noted above. If you scale up, splitting out dedicated master nodes (3 small `m`-only nodes) would reduce the burden on the data nodes' heap.

**No CDM ingestion pattern visible.** Unlike the other deployments, this cluster doesn't appear to be running CDM workloads. The data here is `elastic-cloud-logs`, `metricbeat`, and `monitoring` datastreams — i.e., this looks like a **logging/observability cluster** rather than a CDM data cluster. The `cdm_cpe_dictionary` presence might be for some reference lookup, or might be vestigial from a template that includes it.

---

## Suggested actions (priority ordered)

1. **Scale up the hot tier — this is the headline recommendation.** Two 1.7 GB heap nodes hosting 132 shards is operating at ~194% of safe shard density. RAM at 93–98% leaves no spike margin. Doubling heap to 3.4 GB or moving to 4 GB nodes would put the cluster comfortably back inside Elastic's recommended limits. This is operational risk, not just inefficiency.
2. **Investigate the tiebreaker CPU load** (`tiebreaker-0000000002` at sustained ~5 load average on a 224 MB heap). Voting-only nodes shouldn't see this kind of work. Could be host-level noise reported by the load metric, or could indicate something is exercising the voting node unexpectedly.
3. **Tighten the `elastic-cloud-logs-8` ILM rollover policy** to include a `max_primary_shard_size: 30gb` condition. The current 45 GB primary is at the write-heavy ceiling, and if traffic increases, the next backing index could exceed safe limits.
4. **Delete the legacy v7 monitoring indices** — `.monitoring-es-7-2026.04.01` and `.monitoring-kibana-7-2026.04.01`. They're single-day vestiges from cluster setup. Tiny win but reduces shard count.
5. **No data-loss exposure** — every index has rep=1.
6. **Confirm whether `cdm_cpe_dictionary` is needed here.** If this is purely a logging cluster, the 1.49M-doc dictionary index might be deletable, freeing resources.

Happy to dig deeper into any of these — particularly to help size the right hot-tier upgrade, design the ILM policy update for `elastic-cloud-logs-8`, or diagnose the tiebreaker CPU.
