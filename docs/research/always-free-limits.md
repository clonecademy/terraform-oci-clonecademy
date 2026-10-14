# Always Free limits per module service in us-sanjose-1

Research for issue #2 (parent map #1). Facts read from primary sources on 2026-10-13. Sources are Oracle documentation (`docs.oracle.com`, `oracle.com` price list) and the `oracle/terraform-provider-oci` and `oracle/oci-go-sdk` repositories. Every row cites its source; anything not confirmed in a primary source is marked **UNCERTAIN**.

## Summary

- **Always Free is a billing allowance, not a resource type.** On a Pay As You Go (PAYG) tenancy nothing hard-stops a paid resource. Most limits are *counted by billing*, not enforced by a service limit, so module input validation is the only guard (plus compartment quotas, see [Enforcement](#enforcement-compartment-quotas)).
- **The only free-flagged shapes** are `VM.Standard.E2.1.Micro` (compute), `MySQL.Free` and `HeatWave.Free` (MySQL). Everything else is a normal resource whose usage is netted against a monthly or standing allowance.
- **No `is_free_tier` argument exists** on any resource the module family manages (none appears in the argument references read for instance, volume, VCN, bucket, vault, key, bastion, log, topic, budget and MySQL DB system). The Console "Always Free-eligible" label is UI only.
- **us-sanjose-1** is a live OC1 commercial region with **1 availability domain** (3 fault domains). It is the tenancy home region, which is where every Always Free resource must live.
- **Biggest uncertainty:** the Arm A1 allowance. The live docs say 2 OCPU / 12 GB, but the oracle.com price list still says 3,000 OCPU-hours and 18,000 GB-hours per paid tenancy (4 OCPU / 24 GB). See [A1 conflict](#a1-allowance-conflict).

## Key numbers for validation

| Service | Cap to validate against | Source |
|---|---|---|
| Compute E2.1.Micro | 2 instances | [Always Free](#sources) |
| Compute A1.Flex | 2 OCPU and 12 GB total (use this, not 4/24) | [Always Free](#sources) |
| Boot plus block volume | 200 GB combined; 5 backups combined | [Always Free](#sources) |
| Object Storage (PAYG) | 10 GB Standard + 10 GB Infrequent + 10 GB Archive; 50,000 requests/month | [Always Free](#sources) |
| VCN | No charge for VCNs; Free Tier cap of 2 does NOT apply on PAYG (limit 10/region) | [Always Free](#sources), [Service Limits](#sources) |
| MySQL HeatWave | 1 DB system, `MySQL.Free`, 50 GiB, standalone, 1-node `HeatWave.Free` | [MySQL docs](#sources) |
| Bastion | Free; 5 bastions/region, 20 sessions/bastion | [Price list](#sources), [Service Limits](#sources) |
| Vault | Software keys free; 20 HSM key versions free; 150 secrets; no private vault | [Always Free](#sources) |
| Logging | First 10 GB/month free (shared with VCN flow logs) | [Price list](#sources), [Always Free](#sources) |
| Notifications | 1 million HTTPS and 1,000 email deliveries per month | [Price list](#sources) |
| Budgets | No price-list entry; 1 target per budget, 1 budget per compartment or tag | [Create budget](#sources), [Provider](#sources) |

## Compute

Always Free compute lives in the home region only (`us-sanjose-1`).

### VM.Standard.E2.1.Micro (AMD)

| Item | Value |
|---|---|
| Limit | Up to 2 instances; 1/8 OCPU (burstable), 1 GB memory, one VNIC, up to 50 Mbps internet |
| Identified by | Shape name `VM.Standard.E2.1.Micro`; the Console labels it "Always Free-eligible" |
| Enforced | Yes. Service limit `standard-e2-micro-core-count` = 2 per AD for PAYG (quota family `compute-core`, AD scope). Unit counting (1 per instance) is **UNCERTAIN**; the docs list the limit as "2" next to "up to two instances". |
| Availability domain | "Can only be created in one availability domain" in multi-AD regions; us-sanjose-1 has one, so no placement choice |
| us-sanjose-1 | Available as a home-region shape (see [Regional availability](#regional-availability)); capacity is not guaranteed |
| Images | Oracle Linux Cloud Developer 8, Oracle Linux, Ubuntu, CentOS (those labelled Always Free Eligible) |

### VM.Standard.A1.Flex (Arm)

| Item | Value |
|---|---|
| Limit | First 1,500 OCPU-hours and 9,000 GB-hours per month, equal to 2 OCPUs and 12 GB for Always Free tenancies; one instance of 2 OCPU, or two instances of 1 OCPU |
| Identified by | Shape name only. There is no free-flagged shape and the allowance is a monthly hour pool. The Always Free page points to the Compute section instead of an eligible label for A1. |
| Enforced | **No, on PAYG.** The service limit is `standard-a1-core-count` = 16 and `standard-a1-memory-count` = 96 GB (commercial realm, PAYG or Trial). Usage above 2 OCPU / 12 GB is billed, not blocked. |
| Minimum boot volume | 47 GB per the Always Free page (Console default 50 GB); see [Block Volume](#block-volume-and-boot-volume) |
| Idle reclamation | Always Free instances may be reclaimed if, over 7 days, 95th-percentile CPU < 20%, network < 20% and (A1 only) memory < 20% |
| Images | Oracle Linux Cloud Developer (needs at least 8 GB memory), Oracle Linux, Ubuntu |
| us-sanjose-1 | Allowed in any AD except South Korea North (Chuncheon). Capacity ("out of host capacity") is the known failure mode. |

#### A1 allowance conflict

- Current Always Free docs: 1,500 OCPU-hours and 9,000 GB-hours, "equivalent to 2 OCPUs and 12 GB" ([Always Free Resources](#sources)).
- The oracle.com price list still says "Each paid tenancy gets the first 3,000 OCPU hours and 18,000 GB hours per month for free" for A1, shared across bare metal, VM and container instances ([Price list](#sources)).
- Secondary reporting (not a primary source) says Oracle halved the allowance around June 2026 without an announcement, and that PAYG users above 2 OCPU / 12 GB may now be billed ([linuxiac](#sources)).
- **Decision for validation:** cap at 2 OCPU and 12 GB. Treat 4 OCPU / 24 GB as unsafe. **UNCERTAIN** whether a PAYG tenancy is credited the larger 3,000/18,000 pool on its invoice; confirm in Cost Analysis once an A1 instance has run for a month.

## Block volume and boot volume

| Item | Value |
|---|---|
| Limit | 200 GB combined boot and block storage; 5 volume backups combined; home region only |
| Identified by | Counted only. No flag. A volume outside the home region is billed from the first GB. |
| Enforced | Not for the 200 GB. PAYG service limit is 100 TB aggregated per AD. Compartment quota `total-storage-gb` (family `block-storage`, AD scope) and `backup-count` (regional) can cap it. |
| Sizes | Provider: block volumes 50 GB to 32 TB; boot volume `boot_volume_size_in_gbs` minimum 50 GB. Always Free page says minimum boot volume is 47 GB and Console default is 50 GB. Validate with 50 GB minimum. |
| Layouts | 4 volumes at 50 GB, or one 50 GB boot volume plus 150 GB block volume |
| us-sanjose-1 | Home region, so eligible |

Gotchas:
- Paid and free volumes mix silently. When free capacity appears (a free volume is deleted), the service converts an existing paid volume to Always Free. The reverse also holds: the 201st GB is billed.
- Volume performance (`vpus_per_gb`) above Balanced (10) is documented as a performance choice; whether higher VPUs are billed outside the 200 GB allowance is **UNCERTAIN** (no primary source found). Keep the default of 10.
- Backups: the docs say the 6th backup fails on an Always Free tenancy; that a PAYG tenancy bills it instead is inferred, not stated. Do not attach a backup policy that can exceed 5 retained backups.

## VCN and networking

| Item | Value |
|---|---|
| Limit | Free Tier tenancies (no paid status, no trial credit): up to 2 VCNs. PAYG default: 10 VCNs per region (UC: 50). |
| Identified by | Counted. Service limit `vcn-count` (quota family `vcn`, regional scope). |
| Per-VCN limits | 1 internet gateway, 1 NAT gateway, 1 service gateway, 300 subnets, 300 route tables, 300 security lists, 1,000 NSGs (identical on PAYG) |
| Outbound data | 10 TB per month free; inbound free |
| us-sanjose-1 | Available |

Notes:
- VCNs, subnets, route tables, security lists and NSGs are not listed as billable on the price list. Whether NAT gateways and reserved public IPs carry a charge was **not verified**; the module should not assume either is billed or free until confirmed against the networking pricing page (HTTP 403 for automated fetch).
- Provider: `cidr_blocks` allows /16 to /30; setting `is_ipv6enabled = true` makes Oracle allocate a /56 GUA prefix unless `is_oracle_gua_allocation_enabled = false`.
- Port 25 outbound is blocked by default.
- VCN flow logs count against the shared 10 GB/month Logging allowance (see [Logging](#logging)).

## Object Storage

| Item | Value |
|---|---|
| Limit (paid account) | 10 GB Standard, 10 GB Infrequent Access, 10 GB Archive, plus 50,000 API requests per month |
| Limit (Always Free only, trial expired) | 20 GB combined across tiers, 50,000 requests per month |
| Identified by | Counted only (no flag). Tier is set by the bucket's `storage_tier` (immutable) and `auto_tiering`. |
| Enforced | No. Regional limit is 10,000 buckets per tenancy. |
| us-sanjose-1 | Available (Always Free storage is not stated as home-region-only, but treat as home region) |

Gotchas:
- A PAYG tenancy gets 10 GB per tier, not 20 GB of Standard. Putting more than 10 GB in Standard is billed.
- `versioning = Enabled` keeps old versions, and every version counts toward the 10 GB.
- Requests above 50,000/month are outside the allowance. Whether replication is billed was not checked.
- Terraform bucket defaults: `storage_tier` Standard, `versioning` Disabled, `access_type` `NoPublicAccess`, `auto_tiering` Disabled.

## MySQL HeatWave

| Item | Value |
|---|---|
| Limit | One Always Free DB system per tenancy in the home region (Commercial realm, free or paid) |
| Identified by | **Shape name.** DB system shape `MySQL.Free`; HeatWave cluster shape `HeatWave.Free`. API: `CreateDbSystem` with `shapeName = MySQL.Free`. |
| Fixed configuration | 50 GiB data and log storage; standalone only; automatic backup with 1-day retention (enabled, cannot be disabled); PITR off; manual backups unavailable |
| HeatWave | Optional 1-node `HeatWave.Free` cluster; Lakehouse and AutoML supported; GenAI vector stores not supported |
| Not available | High availability, read replicas, Database Management and Ops Insights integrations, SLA, Oracle Support |
| Enforced | Yes, in effect: creation targets an AD with a non-zero limit for `MySQL.Free`. Creation with that shape fails if the limit is zero. The exact limit name is **UNCERTAIN** (not published on the Service Limits page); look it up in the Console (Limits, Quotas and Usage) or with `oci limits value list`; the service name is not confirmed. |
| Network | Requires a VCN and a **private** subnet; up to 5 NSGs |
| us-sanjose-1 | Home region eligible. A region-by-region Always Free MySQL table was not retrievable, so confirm with `oci mysql shape list` (see [Regional availability](#regional-availability)). |

Gotchas:
- A second DB system with `MySQL.Free` is not creatable; a second system with any other shape is paid.
- The Always Free page quotes "an extra 50 GB of backup storage"; the price list has separate "Free" SKUs for ECPU, HeatWave, storage and backup storage.
- The provider docs list no free argument; the shape name is the only signal. The CLI/Terraform path does not create a HeatWave cluster with the DB system; the cluster is a separate resource (`oci_mysql_heat_wave_cluster`). Whether `HeatWave.Free` is accepted as its `shape_name` is **UNCERTAIN** (not shown in the provider docs).
- Always Free DB systems are always upgraded to the latest version in each maintenance cycle.

## Bastion

| Item | Value |
|---|---|
| Limit | Service is free for free and paid accounts. Limits: 5 bastions per region, 20 sessions per bastion. |
| Identified by | Not applicable. No price-list charge; `bastion_type = "standard"`. |
| Enforced | Service limits only (not billing). |
| us-sanjose-1 | Regional service. Availability in the region is **UNCERTAIN** (see [Regional availability](#regional-availability)). |

Provider notes: `max_session_ttl_in_seconds` has no documented numeric bounds in the Terraform docs. The bastion needs a target subnet (`target_subnet_id`).

## Vault

| Item | Value |
|---|---|
| Software-protected keys | Free, unlimited key versions |
| HSM-protected keys | First 20 key versions free, across any number of keys or vaults |
| Secrets | 150 free secrets; 40 versions per secret (20 active, 20 pending deletion). Limit for PAYG is 5,000 secrets and 60 versions (30/30). |
| Virtual private vault | Not part of Always Free. Billed per hour; includes 1,000 key versions. |
| Identified by | Counted, plus a **type** chosen at creation. `vault_type`: `DEFAULT`, `VIRTUAL_PRIVATE`, `EXTERNAL` (Go SDK enum). Key `protection_mode`: `HSM`, `SOFTWARE`, `EXTERNAL`. Neither can be changed after creation. |
| Enforced | Not for the 20 HSM versions. Virtual vault service limit is 10 vaults and 100 key versions (HSM 100 on PAYG). |
| Key deletion | `time_of_deletion` must be 7 to 30 days after the request (default 30) |
| us-sanjose-1 | Regional; availability is **UNCERTAIN** (see [Regional availability](#regional-availability)) |

Gotchas:
- **The provider default for `protection_mode` is `HSM`.** Leaving it unset consumes the 20 free HSM key versions, then bills per version. Modules should default to `SOFTWARE`.
- Each rotation adds a key version. Asymmetric (RSA, ECDSA) key versions count as two toward limits.
- `DEFAULT` is the only free vault type. `VIRTUAL_PRIVATE` (hourly charge) must never be selectable in the free modules.
- Whether key versions pending deletion still count or are billed was not confirmed.

## Logging

| Item | Value |
|---|---|
| Limit | First 10 GB of log storage per month is free (price list); VCN flow logs on Free Tier are "up to 10GB per month shared across OCI Logging" |
| Identified by | Counted by volume. No flag. |
| Enforced | Regional limits only: 100 log groups (`log-group-count`), 500 log objects (`log-count`), 100 agent configurations. These are the same on PAYG and trial. |
| Retention | Provider: `retention_duration` is 30 to 180 days in 30-day steps |
| us-sanjose-1 | Regional; availability **UNCERTAIN** (see [Regional availability](#regional-availability)) |

Gotchas: stored volume above 10 GB per month is billed. The unit price was blanked on the fetched price list. Longer retention raises stored volume. The Always Free page lists no Logging quantity of its own beyond the flow-log sentence. Logging Analytics is a separate product with separate pricing.

## Notifications

| Item | Value |
|---|---|
| Limit | First 1 million delivery operations (HTTPS) and first 1,000 email deliveries per month free |
| Identified by | Counted by volume. |
| Enforced | `topic-count` = 50 per region on Always Free tenancies, 100 on PAYG. Subscriptions: 10 per topic (active or pending), 100 pending per tenancy. |
| Provider | Topic `name` is unique across the tenancy and not updatable; create is rate-limited to 60 per minute |
| us-sanjose-1 | Regional; availability **UNCERTAIN** (see [Regional availability](#regional-availability)) |

Gotcha: the price list lists only HTTPS delivery operations and emails as free; other protocols (for example SMS) have no stated allowance, so treat them as billable.

## Budgets

| Item | Value |
|---|---|
| Limit | Budgets are not on the Always Free page, the price list or the Service Limits page. No charge is documented. **UNCERTAIN** whether a number is published; absence is not a guarantee. |
| Rules | One budget per compartment ("can't create a budget in a compartment that already has a budget") and one per tag. A budget has exactly one target. |
| Created in | The root compartment (needs `manage usage-budgets in tenancy`), regardless of target |
| Amount | 1 to 999,999,999,999 in the rate-card currency; `reset_period` is `MONTHLY` only |
| Alerts | Multiple alert rules per budget (no maximum stated). Threshold is a percentage up to 10,000 or an absolute amount. Email text max 1,000 characters. Actual-spend alerts evaluated every 24 hours; forecast alerts need at least 3 days of data and are evaluated hourly. |
| Enforced | Never stops spending. Alerts only. |
| us-sanjose-1 | Global/root-compartment service; not region-limited |

Provider: `targets` must contain exactly one item; `target_type` is `COMPARTMENT` or `TAG`; `processing_period_type` is `INVOICE`, `MONTH` or `SINGLE_USE`.

Design consequence: the "free stack" budget and the "whole tenancy" budget are two budgets on two different compartments (the stack compartment and the root compartment), which the one-per-compartment rule allows. A root-compartment budget also tracks child compartments, so spend is double-counted between them by design.

## Regional availability

- **us-sanjose-1** (key `SJC`, realm OC1, region status Live) has **1 availability domain**, with 3 fault domains per AD ([Regions](#sources), [Public cloud regions](#sources)).
- It is the tenancy home region, so the home-region constraint on Always Free compute, block volume and MySQL is met.
- **UNCERTAIN, not verified:** a per-service Always Free availability table for San Jose. Oracle points to a "Data Regions" page, which is a JavaScript application that the research tooling could not render. Services in the module family are regional and part of the commercial OC1 realm; confirm in the tenancy with:
  - `oci iam availability-domain list` (expect one AD)
  - `oci mysql shape list --compartment-id <ocid>` (expect `MySQL.Free`)
  - `oci limits resource-availability get --service-name compute --limit-name standard-e2-micro-core-count --availability-domain <ad> --compartment-id <ocid>` (and the A1 equivalents)
  - Console: Governance and Administration, Limits, Quotas and Usage.

## Enforcement: compartment quotas

Because most limits are billing-counted, hard caps come from compartment quotas (set in the root compartment, scope is per AD or per region):

| Resource | Quota family | Quota name | Scope |
|---|---|---|---|
| A1 OCPUs | `compute-core` | `standard-a1-core-count` | AD |
| A1 memory (GB) | `compute-memory` | `standard-a1-memory-count` | AD |
| E2.1.Micro | `compute-core` | `standard-e2-micro-core-count` | AD |
| Boot and block volume GB | `block-storage` | `total-storage-gb` | AD |
| Volume backups | `block-storage` | `backup-count` | Region |
| VCNs | `vcn` | `vcn-count` | Region |

Example form from the docs: `set compute-core quota standard3-core-count to 480 in compartment MyCompartment`. The A1 statements above are the same pattern (the page lists only other shapes as examples). Quota names for MySQL, Vault, Object Storage and Logging were not retrieved (**UNCERTAIN**).

## Pay As You Go gotchas (what silently becomes billable)

1. **A1 above 2 OCPU / 12 GB**: not blocked (limit 16 OCPU / 96 GB), billed per hour. Possible 4/24 pool on the price list is unconfirmed.
2. **Any resource outside the home region**: Always Free applies in the home region only; block volumes elsewhere "incur regular charges".
3. **201st GB of block/boot storage** and **6th volume backup**.
4. **Object Storage Standard above 10 GB**, versioned objects, and requests above 50,000/month.
5. **HSM key versions above 20**; the provider default `protection_mode` is `HSM`. Virtual private vaults.
6. **Logging above 10 GB/month**, including VCN flow logs.
7. **A second MySQL DB system**, or any shape other than `MySQL.Free`; enabling HA or a larger HeatWave cluster is a different paid shape.
8. **Outbound data above 10 TB/month.**
9. **Load balancers (beyond the one Always Free flexible LB at 10 Mbps), reserved IPs unattached, NAT gateway** are out of scope for the module family, but note the Always Free flexible load balancer only applies to tenancies created on or after 15 Dec 2020.
10. **Mixing paid and free volumes** converts silently in both directions as capacity changes.
11. **Upgrade effect on Object Storage:** an Always Free-only tenancy gets 20 GB combined; a PAYG tenancy gets 10 GB per tier.
12. **Idle reclamation** can delete Always Free instances (7-day, 20% rule); it is not a billing event but an availability risk.

## Terraform identification summary

| Resource | Free signal | Provider argument |
|---|---|---|
| `oci_core_instance` | Shape name only | `shape = "VM.Standard.E2.1.Micro"` / `VM.Standard.A1.Flex` with `shape_config` (ocpus, memory_in_gbs) |
| `oci_core_volume` / boot volume | Counted only | `size_in_gbs` (50 minimum), `vpus_per_gb` (0, 10, 20, 30-120) |
| `oci_core_vcn` | Counted only | none |
| `oci_objectstorage_bucket` | Counted only | `storage_tier`, `versioning` |
| `oci_mysql_mysql_db_system` | Shape name | `shape_name = "MySQL.Free"`, `is_highly_available = false`, `data_storage_size_in_gb = 50` |
| `oci_bastion_bastion` | Free service | `bastion_type = "standard"` |
| `oci_kms_vault` / `oci_kms_key` | Type and mode | `vault_type = "DEFAULT"`, `protection_mode = "SOFTWARE"` |
| `oci_logging_log` | Counted only | `retention_duration` (30-day steps, max 180) |
| `oci_ons_notification_topic` | Counted only | none |
| `oci_budget_budget` | Free (no entry) | `targets` length exactly 1, `reset_period = "MONTHLY"` |

## Sources

- Always Free Resources (live, copyright 2026): https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
- Free Tier overview (trial end, home region, A1 limit): https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier.htm
- Limits by Service (compute, block, VCN, MySQL, Vault, Logging, Notifications, Bastion): https://docs.oracle.com/en-us/iaas/Content/General/Concepts/servicelimits.htm
- MySQL HeatWave, Always Free Service definition: https://docs.oracle.com/en-us/iaas/mysql-database/doc/features-mysql-heatwave-service.html
- MySQL HeatWave, Creating an Always Free DB System: https://docs.oracle.com/en-us/iaas/mysql-database/doc/creating-always-free-db-system.html
- Oracle Cloud price list (Logging, Notifications, Bastion, Vault, MySQL Free, A1 note): https://www.oracle.com/cloud/price-list/ (fetched as https://www.oracle.com/devops/logging/pricing/; unit prices are blanked in the fetched copy)
- Regions and availability domains: https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm
- Public cloud regions (San Jose Live): https://www.oracle.com/cloud/public-cloud-regions/
- Budgets overview: https://docs.oracle.com/en-us/iaas/Content/Billing/Concepts/budgetsoverview.htm
- Creating a budget: https://docs.oracle.com/en-us/iaas/Content/Billing/Tasks/create-budget.htm
- Vault and key overview: https://docs.oracle.com/en-us/iaas/Content/KeyManagement/Concepts/keyoverview.htm
- Compute quotas: https://docs.oracle.com/en-us/iaas/Content/Quotas/Concepts/resourcequotas_topic-Compute_Quotas.htm
- Block Volume quotas: https://docs.oracle.com/en-us/iaas/Content/Quotas/Concepts/resourcequotas_topic-Block_Volume_Quotas.htm
- Networking quotas: https://docs.oracle.com/en-us/iaas/Content/Quotas/Concepts/resourcequotas_topic-Networking_Quotas.htm
- Overview of compartment quotas: https://docs.oracle.com/en-us/iaas/Content/Quotas/Concepts/resourcequotas.htm
- Terraform provider docs (raw): https://github.com/oracle/terraform-provider-oci/tree/master/website/docs/r (`core_instance`, `core_volume`, `core_vcn`, `objectstorage_bucket`, `kms_vault`, `kms_key`, `bastion_bastion`, `logging_log`, `ons_notification_topic`, `budget_budget`, `mysql_mysql_db_system`)
- OCI Go SDK, vault type enum and key deletion window: https://github.com/oracle/oci-go-sdk/tree/master/keymanagement (`create_vault_details.go`, `schedule_key_deletion_details.go`)
- Secondary (context only, not a primary source), A1 allowance reduction: https://linuxiac.com/oracle-quietly-cuts-free-tier-ampere-a1-resources-in-half/

## Open questions to resolve before finalising module validation

1. Does a PAYG tenancy get a 1,500/9,000 or a 3,000/18,000 A1 pool on its invoice? (Check Cost Analysis after a month of A1 usage.)
2. Name of the `MySQL.Free` and `HeatWave.Free` service limits and whether `HeatWave.Free` is accepted by `oci_mysql_heat_wave_cluster`.
3. Per-service availability of Bastion, Vault, Logging, Notifications and MySQL Free in us-sanjose-1 (confirm with the CLI commands above).
4. Whether NAT gateways, reserved public IPs and higher `vpus_per_gb` are free.
5. Whether keys pending deletion are billed.
6. Whether Budgets carry any published cap on budgets per tenancy or alert rules per budget.
