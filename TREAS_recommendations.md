# Cluster Operational Review

**Cluster:** `17d55ba912e640cdae0979d36e9e5795` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 17 total nodes — 3 master (`mr`, 1.7 GB heap), 12 hot/ingest/remote/search/transform (`hirst`, 29.6 GB heap, ~1.5 TB disk each), 2 frozen (`f`, 14.7 GB heap). 103 primaries / 256 total shards / 0 unassigned.

This deployment looks dramatically different from a typical production cluster. Let me walk through it.

---

## 1. Oversized shards & resharding recommendations

**There are no oversized shards in this cluster — there is essentially no data.**

Every single index has either zero documents or, in the case of `cdm_cpe_dictionary` (1.49M docs) and a few Kibana/system indices, store sizes that round to 0 GB. The largest user-data shard footprint anywhere is 250 MB (split between `cdm_cpe_dictionary` and one SLM history datastream backer on `instance-0000000008` and `instance-0000000011`).

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_cpe_dictionary` | 1 | <1 GB (≈250 MB) | <1 | OK |
| Every other `cdm_*` and `.ds-cdm_*` | 1 | 0 GB | 0 | empty |
| `.kibana_*`, `.security*`, `.tasks`, `.transform-*` | 1 | 0 GB | 0 | system-tiny |

**No resharding is warranted on the data side**, because there's no data to reshard. The only sizing signal in the cluster is structural — every CDM index is at `pri=1, rep=1`, which is the right default for 0 GB content. If/when data starts flowing in, you'll want to revisit primary counts based on actual ingest volume against the same 40 GB/shard target.

## 2. Is the cluster oversharded?

**By the standard heuristic, no. By the "shard-to-data ratio" smell test, very much yes — but only because the cluster is empty.**

- 256 total shards across 12 hot nodes = **~21 shards per hot node**. Against a ceiling of ~592 shards/node (20 shards per GB of 29.6 GB heap), you're at ~3.6% of the safe limit.
- 103 primaries holding effectively 0 GB of real data means the cluster state is carrying mappings, settings, and shard metadata for indices that contain nothing.

The 21 shards/node figure is dominated by two things:
1. **Enrich indices replicated to every hot node.** You have 5 enrich policies (`cdm_config_t0`, `cdm_software_directive`, `cdm_csm_stig`, `cdm_software_dictionary_eol`, `cdm_vuln_cve`) × 12 nodes = ~60 shards just for enrich. That's roughly a quarter of all shards in the cluster, all for ingest pipeline lookups.
2. **Empty CDM and Kibana alerting indices** that are pre-provisioned but never written to.

If the cluster is in pre-production / awaiting data load (which the evidence strongly suggests — see §5), this state is fine. If it's been running this way in steady state for weeks, then yes, you're paying cluster-state overhead for a lot of empty containers.

## 3. Empty indices & their sources

This is where this cluster really stands out: **57 of the ~58 user-visible indices are empty**. The only index with meaningful document count is `cdm_cpe_dictionary` (1.49M docs from CPE seed data). Even `data_dictionary` is small reference data, and the rest is at 0 docs.

Categorized:

**Kibana Alerting framework (14 indices)** — `.internal.alerts-*-default-000001` covering security, ML anomaly detection (×2), observability (metrics/logs/uptime/threshold/slo/apm), streams, stack, transform health, dataset quality, attack discovery, default. Auto-created by Kibana, expected to be empty until rules fire. Leave them alone.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. New in 8.16+, used when migrating SIEM detection rules from other vendors (Splunk, etc.). Empty unless you're actively running a migration.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Provisioned by APM when integration loads, populated only when APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Created by Kibana's SLO feature; empty if no SLOs are defined.

**CDM trending datastream backers (10 indices)** — all `.ds-cdm_*_trending-2026.03.25-000001`. Every single one is the *first* generation backing index (`-000001`), all created on **2026-03-25** within seconds of each other. None has rolled over, none has data. This is the smoking gun — see §5.

**CDM application indices (24 indices)** — `cdm_vuln`, `cdm_vuln_remediated`, `cdm_vuln_trending-6-000001`, `cdm_cyhy_*`, `cdm_scuba_m365_*`, `cdm_scuba_gws_*`, `cdm_asm_nmi`, `cdm_vdp`, `cdm_threat`, `cdm_hwam`, `cdm_software_dictionary`, `cdm_kev_dictionary`, `cdm_stig_dictionary`, `cdm_cve_dictionary`, `cdm_directive_dictionary`, `cdm_benchmark_vuln`, `cdm_system_boundary_fisma_1163_rollup`, `cdm_cyhy_host_scans`, `cdm_cyhy_domain*`, `cdm_cyhy_vuln`. **All zero docs.** Notably, even the dictionary indices that are normally seeded by external feeds (`cdm_cve_dictionary`, `cdm_stig_dictionary`, `cdm_software_dictionary`, `cdm_kev_dictionary`) are empty here, suggesting the dictionary refresh jobs haven't run against this cluster either.

**Other (1):** `.kibana_locks-000001`. Notably absent: there are no `.elastic-connectors*` or `.ent-search-*` indices in this deployment, which is good — it means this cluster skipped Enterprise Search. Since Enterprise Search is **removed in 9.0**, this cluster is in good shape for that upgrade.

**Frozen tier:** Both `instance-0000000018` and `instance-0000000019` show **0 shards, 0 bytes of disk indices**. Yet both report `disk.used_percent: 95.57%`. That's a quirk of how ESS reports overcommitted frozen-tier disk — the number doesn't reflect anything to worry about while the nodes hold no content.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index — `cdm_*`, `.ds-cdm_*`, `.internal.alerts-*`, `.kibana_*`, system indices — has `rep=1`. Even with the cluster being mostly empty, every index is configured for HA replication.

There are no `partial-*` searchable-snapshot mounts either, which is consistent with the frozen nodes carrying no shards: nothing has aged into the frozen tier yet.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This cluster looks like a **freshly provisioned environment that has not yet started receiving data**, or has had its data wiped. The evidence is overwhelming and consistent:

**The 2026-03-25 creation timestamp pattern.** Every empty CDM index — both regular indices and `.ds-` datastream backers — has a `creation.date.string` between `2026-03-25T19:50:17` and `2026-03-25T20:07:13`. That's a ~17-minute window roughly six weeks ago. This isn't a slow drift; it's a single bulk provisioning event, almost certainly an Index Template / Component Template installation or a Terraform / API deploy that ran setup, created the empty indices, and stopped there. A healthy ingesting cluster would show staggered creation timestamps reflecting rollover events and ad-hoc reseeding over time.

**Enrich policies were rebuilt later.** The enrich indices have timestamps in their names like `1774472199442` (which is **2026-04-25**), suggesting `_enrich/policy/.../_execute` ran a month *after* the index provisioning. This is consistent with a deployment script that runs setup, provisions indices, then later kicks off enrich rebuilds — but the actual ingestion never followed.

**No ILM activity.** The `.ds-ilm-history-7-*` datastreams are tiny (single digits of docs in the most recent backing index). ILM has done almost nothing on this cluster because there's nothing to manage. Trending datastreams are still all on backing index `-000001` six weeks after creation — no rollovers have occurred because no documents have arrived to age out.

**Heap utilization is bimodal across hot nodes.** Look at the heap percentages: `instance-0000000013` at 2%, `instance-0000000011` at 7%, vs `instance-0000000008` at 59%. With nearly identical shard counts (21–22 per node) and effectively no data, this variance has to come from JVM warmup state and Lucene-internal caches associated with the few shards that *do* hold something (the enrich shards, `cdm_cpe_dictionary`, Kibana system indices). On a real production cluster you'd want this more uniform; here it's just noise.

**Master node CPU.** `instance-0000000004` (master-eligible) shows `load_1m=3.20, load_5m=4.09, load_15m=4.07` — sustained load on a 1.7 GB heap node. The other masters (`instance-0000000003` and `instance-0000000020`, which is the elected master) are quieter at ~1.5 and ~1.0 respectively. Worth a look at what task is keeping that node busy on an otherwise idle cluster — could be a stuck transform, an enrich rebuild loop, or just background master tasks dealing with cluster-state churn from empty index housekeeping.

**Frozen disk percent reads 95.57% on both nodes despite 0 indices.** This is an ESS reporting artifact (overcommit math on `disk.total` vs `disk.avail`), not a real signal here. It would be a real signal if the frozen tier actually held shards — but in this deployment, both frozen nodes are completely empty.

**No `partial-*` shards.** No frozen searchable snapshots have been mounted. Either ILM hasn't aged anything into frozen yet (likely, since there's no data) or the frozen-tier policy isn't yet wired up.

**Index naming.** `cdm_vuln_trending-6-000001` uses six-digit zero-padding for the rollover number. If you have shared tooling across deployments that targets these by exact name pattern, the glob length may matter. Worth a quick check that index templates and downstream consumers match.

---

## Suggested actions

1. **Verify expected state.** Is this cluster supposed to be ingesting? If yes, this is the headline finding — every CDM ingestion path is silent. Six weeks of empty trending datastreams (still on backing index `-000001`) suggests no documents have ever arrived.
2. **If this cluster is intentionally pre-prod / standby:** the current state is fine. No remediation needed.
3. **If this cluster should be receiving data:** check the upstream Logstash / pipeline configuration pointing at this cluster. Given the past CDM pipeline silent-failure pattern (zero-document ingestion with `Connection reset by peer` errors), this looks like the same failure mode at scale — every CDM target is silent. I'd start with the Logstash output config and any CCS / cross-cluster replication settings if data is supposed to be flowing in from another cluster.
4. **Investigate the master node load** on `instance-0000000004`. With no real workload, sustained 4.0 load on a small master node is unexpected.
5. **Investigate the dictionary indices.** `cdm_cve_dictionary`, `cdm_stig_dictionary`, `cdm_kev_dictionary`, `cdm_software_dictionary` should all be populated by their respective seed/sync jobs (NVD CVE feed, STIG library, CISA KEV catalog, software inventory). All four are empty. These are usually pulled by a separate scheduled job, not by Logstash — worth checking whether those refresh jobs even target this cluster.

Happy to dig deeper into any of these — particularly the ingestion-path diagnosis if this cluster is supposed to be live, or specific index template inspection to confirm the empty CDM indices are using the mappings you'd expect data to flow into.
