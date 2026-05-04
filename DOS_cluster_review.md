# DOS Cluster Operational Review

**Cluster:** `0f1a7441b2a5412bb330ef592c3f71bc` (Elastic Cloud, ESS) · **Version:** 8.19.14 · **Status:** GREEN · **Date:** 2026-05-03

**Topology:** 12 total nodes — 3 master (`mr`, 1.7 GB heap), 6 hot/ingest/remote/search/transform (`hirst`, 29.6 GB heap, ~1.5 TB disk each), 3 frozen (`f`, 3.7 GB heap, 600 GB disk each). 111 primaries / 242 total shards / 0 unassigned.

This deployment looks essentially idle from a data perspective. The walkthrough follows.

---

## 1. Oversized shards & resharding recommendations

**There are no oversized shards in this cluster — there is essentially no data.**

Every single index has either zero documents or a primary store that rounds to 0 GB. The only index with meaningful document count is `cdm_cpe_dictionary` (1.49M docs, ~250 MB on disk). Everything else — every CDM application index, every trending datastream, every Kibana alerting index — is empty.

| Index | Primaries | Primary store | GB / primary | Verdict |
|---|---:|---:|---:|---|
| `cdm_cpe_dictionary` | 1 | <1 GB (≈250 MB) | <1 | OK |
| Every other `cdm_*` and `.ds-cdm_*` | 1 | 0 GB | 0 | empty |
| `.kibana_*`, `.security*`, `.tasks`, `.transform-*` | 1 | 0 GB | 0 | system-tiny |

**No resharding is warranted on the data side**, because there is no data to reshard. Every CDM index is at `pri=1, rep=1`, which is the right default for 0 GB content. If/when data starts flowing in, primary counts should be revisited based on actual ingest volume against the same 40 GB/shard target.

## 2. Is the cluster oversharded?

**By the standard heuristic, no. By the "shard-to-data ratio" smell test, yes — but only because the cluster is empty.**

- 242 total shards across 6 hot nodes = **~40 shards per hot node**. Against a ceiling of ~592 shards/node (20 shards per GB of 29.6 GB heap), the cluster is at ~6.8% of the safe limit.
- 111 primaries holding effectively 0 GB of real data means the cluster state is carrying mappings, settings, and shard metadata for indices that contain nothing.

The 40 shards/node figure is dominated by two things:

1. **Enrich indices replicated to every hot node.** There are 5 enrich policies (`cdm_config_t0`, `cdm_software_directive`, `cdm_csm_stig`, `cdm_software_dictionary_eol`, `cdm_vuln_cve`) × 6 nodes = ~30 shards just for enrich. That is roughly 12% of all shards in the cluster, all for ingest pipeline lookups.
2. **Empty CDM and Kibana alerting indices** that are pre-provisioned but never written to.

Notable: this cluster also has `.monitoring-kibana-7-2026.03.23` present (legacy Stack Monitoring index from one day shortly after provisioning), suggesting Stack Monitoring was briefly enabled then disabled — only one day of data was captured.

## 3. Empty indices & their sources

This is where this deployment really stands out: **almost every user-visible index is empty**. The only indices with meaningful document count are `cdm_cpe_dictionary` (1.49M docs from CPE seed data) and `data_dictionary` (3,858 docs of small reference data); everything else is at 0 docs or trivial system content.

Categorized:

**Kibana Alerting framework (15 indices)** — the full `.internal.alerts-*-default-000001` family covering security, ML anomaly detection (×2), observability (metrics/logs/uptime/threshold/slo/apm), streams, stack, transform health, dataset quality, attack discovery, default. Auto-created by Kibana, expected to be empty until rules fire. They should be left alone — Kibana recreates them if deleted.

**Kibana SIEM rule migrations (2 indices)** — `.kibana-siem-rule-migrations-prebuiltrules`, `.kibana-siem-rule-migrations-integrations`. New in 8.16+, used when migrating SIEM detection rules from other vendors (Splunk, etc.). Empty unless a migration is actively running.

**APM system indices (3 indices)** — `.apm-source-map`, `.apm-custom-link`, `.apm-agent-configuration`. Provisioned by APM when the integration loads, populated only when APM agents send data.

**SLO observability (3 indices)** — `.slo-observability.summary-v3.5`, `.slo-observability.summary-v3.5.temp`, `.slo-observability.sli-v3.5`. Created by Kibana's SLO feature; empty if no SLOs are defined.

**CDM trending datastream backers (10 indices)** — all `.ds-cdm_*_trending-2026.03.23-000001`. Every single one is the *first* generation backing index (`-000001`), all created on **2026-03-23** within a tight window. None has rolled over, none has data. This is the smoking gun — see §5.

**CDM application indices (24 indices)** — `cdm_vuln`, `cdm_vuln_remediated`, `cdm_vuln_trending-6-0000001`, `cdm_csm_trending-6-0000001`, `cdm_swam_trending-6-0000001`, `cdm_hwam_trending-6-0000001`, `cdm_device_scan_trending-6-0000001`, `cdm_scan_trending-6-0000001`, `cdm_system_boundary_trending-6-0000001`, `cdm_cyhy_*`, `cdm_scuba_m365_*`, `cdm_scuba_gws_*`, `cdm_asm_nmi`, `cdm_vdp`, `cdm_threat`, `cdm_hwam`, `cdm_software_dictionary`, `cdm_kev_dictionary`, `cdm_stig_dictionary`, `cdm_cve_dictionary`, `cdm_directive_dictionary`, `cdm_benchmark_vuln`, `cdm_system_boundary_fisma_1163_rollup`, `cdm_cyhy_host_scans`, `cdm_cyhy_domain*`, `cdm_cyhy_vuln`. **All zero docs.** Notably, even the dictionary indices that are normally seeded by external feeds (`cdm_cve_dictionary`, `cdm_stig_dictionary`, `cdm_software_dictionary`, `cdm_kev_dictionary`, `cdm_directive_dictionary`) are empty here, suggesting the dictionary refresh jobs haven't run against this cluster either.

**Other (1):** `.kibana_locks-000001`. Notably absent: there are no `.elastic-connectors*` or `.ent-search-*` indices in this deployment, which is good — it means this cluster skipped Enterprise Search. Since Enterprise Search is **removed in 9.0**, this cluster is in good shape for that upgrade.

**Frozen tier:** All three frozen nodes (`instance-0000000012`, `instance-0000000013`, `instance-0000000014`) show **0 shards, 0 bytes of disk indices**. They report `disk.used_percent: 90.08%` — that is an ESS reporting artifact (overcommit math on `disk.total` vs `disk.avail`), not a real signal here. It would be a real signal if the frozen tier actually held shards — but in this deployment, all three frozen nodes are completely empty. With no data ever aged into frozen, three frozen nodes is significant unused capacity.

## 4. Indices with zero replicas in the hot tier

**There are zero indices with `rep=0`.** Every single index — `cdm_*`, `.ds-cdm_*`, `.internal.alerts-*`, `.kibana_*`, system indices — has `rep=1`. Even with the cluster being mostly empty, every index is configured for HA replication.

There are no `partial-*` searchable-snapshot mounts either, which is consistent with the frozen nodes carrying no shards: nothing has aged into the frozen tier yet.

**No data-loss risk from missing replicas.**

## 5. Other operational insights

This cluster looks like a **freshly provisioned environment that has not yet started receiving data**, or has had its data wiped. The evidence is overwhelming and consistent:

**The 2026-03-23 creation timestamp pattern.** Every empty CDM index — both regular indices and `.ds-` datastream backers — has a `creation.date.string` between `2026-03-23T16:30:11` (Kibana alerting) and `2026-03-23T18:49:37` (CDM trending). That is a window of about 2.5 hours roughly six weeks ago. This is not a slow drift; it is a single bulk provisioning event, almost certainly an Index Template / Component Template installation or a Terraform / API deploy that ran setup, created the empty indices, and stopped there. A healthy ingesting cluster would show staggered creation timestamps reflecting rollover events and ad-hoc reseeding over time.

**Enrich policies were rebuilt later.** The enrich indices have timestamps in their names like `1774288205013` (which is **2026-03-23**, same as provisioning) and `1777580955791` for `.enrich-cdm_vuln_cve` (which is **2026-04-30**, a week before the snapshot). This suggests `_enrich/policy/.../_execute` ran multiple times on different schedules. The `.enrich-cdm_vuln_cve` rebuild on 2026-04-30 is interesting — it implies *something* is running enrich rebuilds, even though no actual data is flowing in to be enriched.

**No ILM activity beyond housekeeping.** The `.ds-ilm-history-7-*` datastreams are tiny. The most recent backing index is `.ds-ilm-history-7-2026.04.22-000003` with 0 docs. ILM has done almost nothing on this cluster because there is nothing to manage. Trending datastreams are still all on backing index `-000001` six weeks after creation — no rollovers have occurred because no documents have arrived to age out.

**Master node CPU is concerning.** `instance-0000000003` (master-eligible, not the elected master) shows `load_1m=5.98, load_5m=6.95, load_15m=6.85` — sustained load of ~6-7 on a 1.7 GB heap node. With essentially no real workload in the cluster, this is unexpected and worth investigating. Could be a stuck transform, a runaway enrich job loop, or background master tasks dealing with cluster-state churn from the empty index housekeeping. The elected master (`instance-0000000004`) and the third master (`instance-0000000005`) are quieter (~1.5 and ~0.5 respectively).

**Frozen tier is unused but expensive.** Three frozen nodes (each 600 GB at 90% reported usage but 0 actual shards) is significant overprovisioning if nothing is ever going to age there. If this cluster is intentionally pre-prod and will eventually have data flowing through ILM into frozen, leave them alone. If not, the frozen tier could potentially be scaled down to save cost.

**Index naming.** `cdm_vuln_trending-6-0000001` uses seven-digit zero-padding for the rollover number. If shared tooling across deployments targets these by exact name pattern, the glob length may matter. Worth a quick check that index templates and downstream consumers match.

---

## Suggested actions

1. **Verify expected state.** Is this cluster supposed to be ingesting? If yes, this is the headline finding — every CDM ingestion path is silent. Six weeks of empty trending datastreams (still on backing index `-000001`) suggests no documents have ever arrived.
2. **If this cluster is intentionally pre-prod / standby:** the current state is fine. No remediation needed beyond the master CPU investigation below.
3. **If this cluster should be receiving data:** check the upstream Logstash / pipeline configuration pointing at this cluster. Given the past CDM pipeline silent-failure pattern (zero-document ingestion with `Connection reset by peer` errors), this looks like the same failure mode at scale — every CDM target is silent. Start with the Logstash output config and any CCS / cross-cluster replication settings if data is supposed to be flowing in from another cluster.
4. **Investigate the master node load** on `instance-0000000003`. With no real workload, sustained 6-7 load on a small master node is unexpected.
5. **Investigate the dictionary indices.** `cdm_cve_dictionary`, `cdm_stig_dictionary`, `cdm_kev_dictionary`, `cdm_software_dictionary`, `cdm_directive_dictionary` should all be populated by their respective seed/sync jobs (NVD CVE feed, STIG library, CISA KEV catalog, software inventory). All five are empty. These are usually pulled by a separate scheduled job, not by Logstash — worth checking whether those refresh jobs even target this cluster.
6. **Right-size the frozen tier.** Three frozen nodes for zero frozen shards is overprovisioned. Either confirm a near-term plan to use them, or scale down to one (matching the typical pattern in the team's other deployments).
