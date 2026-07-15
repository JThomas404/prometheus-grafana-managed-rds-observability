# HCS DBaaS Observability Platform — Prometheus, Grafana and Custom Metrics Exporters for a Managed PostgreSQL Estate

## Table of Contents

- [Overview](#overview)
- [Real-World Business Value](#real-world-business-value)
- [Prerequisites](#prerequisites)
- [Project Folder Structure](#project-folder-structure)
- [Tasks and Implementation Steps](#tasks-and-implementation-steps)
- [Core Implementation Breakdown](#core-implementation-breakdown)
  - [Custom OC Exporter — Host-Level Metrics](#custom-oc-exporter--host-level-metrics)
  - [postgres_exporter Fleet — Database-Level Metrics](#postgres_exporter-fleet--database-level-metrics)
  - [Prometheus and Alerting Rules](#prometheus-and-alerting-rules)
  - [Grafana Dashboards](#grafana-dashboards)
  - [Microsoft Teams Alert Routing](#microsoft-teams-alert-routing)
  - [CI/CD Pipeline](#cicd-pipeline)
- [Local Testing and Debugging](#local-testing-and-debugging)
- [IAM Role and Permissions (Optional)](#iam-role-and-permissions-optional)
- [Design Decisions and Highlights](#design-decisions-and-highlights)
- [Errors Encountered and Resolved (Optional)](#errors-encountered-and-resolved-optional)
- [Skills Demonstrated](#skills-demonstrated)
- [Conclusion](#conclusion)

---

## Overview

This repository documents the design, implementation and operation of an observability platform built for a fleet of 137 or more managed PostgreSQL instances running on Huawei Cloud Stack (HCS). The instances are provisioned as Relational Database Service (RDS) resources and are fully managed by the platform, which means there is no SSH access, no SNMP agent and no standard node-level exporter available on the underlying virtual machines.

To close this visibility gap, a custom metrics exporter was built in Python to authenticate against the Huawei ManageOne Operation Center (OC) API, auto-discover every PostgreSQL RDS node from the platform's configuration management database, and expose host-level metrics (CPU, memory, disk, IOPS, replication status) as Prometheus gauges. This was combined with a per-instance `postgres_exporter` fleet for database-internal metrics, a Prometheus and Alertmanager stack for storage and alert evaluation, and four purpose-built Grafana dashboards for different operational audiences. Alerts are routed to Microsoft Teams with structured, runbook-style content, and all configuration is deployed through a dedicated Azure DevOps CI/CD pipeline that validates before it deploys.

The platform was designed, built and operated end to end, covering the authentication reverse-engineering required to reach the platform API, the exporter and reconciler code, the Prometheus alerting rule set, the Grafana dashboard design, and the CI/CD deployment pipeline.

## Real-World Business Value

Prior to this platform, the database operations team had no proactive visibility into the health of the underlying infrastructure hosting their PostgreSQL fleet. Because the RDS service is fully managed, the team relied on reactive reports from application teams to learn about CPU saturation, disk exhaustion or replication lag, typically after the issue had already affected a workload.

This platform changes that posture in several concrete ways:

- **Proactive detection.** Twenty-nine alerting rules across five groups provide sub-minute detection of saturation, disk exhaustion risk and replication lag across the entire fleet, rather than waiting for a downstream report.
- **Zero-touch fleet growth.** New RDS instances are auto-discovered from the platform's configuration management database and appear in metrics collection within sixty seconds, and gain a dedicated `postgres_exporter` unit within five minutes, with no manual configuration step.
- **Capacity planning ahead of overcommit.** A dedicated capacity dashboard exposes resource pool and storage backend allocation rates, allowing the team to anticipate hypervisor or storage overcommit before it degrades every instance sharing that pool.
- **Faster incident response.** Alerts carry structured, runbook-style annotations (what happened, current value, threshold, likely cause, remediation steps) directly in the Microsoft Teams notification, reducing the need to consult a separate wiki during an incident.
- **Safe, auditable change management.** Monitoring configuration is validated by `promtool` and Python syntax checks in a continuous integration stage before anything is deployed, and deployment is restricted to the main branch, giving the team a controlled path to adjust thresholds or dashboards without risking a Prometheus outage.

## Prerequisites

The platform assumes the following environment and access is available:

- A Linux bastion host with outbound HTTPS access to the ManageOne Operation Center API and to the Huawei Cloud RDS API.
- Python 3.6 or later, with the `pycryptodome` and `prometheus_client` packages installed.
- Prometheus 2.53 or later and Grafana installed on the bastion host, together with Alertmanager.
- Service account credentials for the ManageOne Operation Center, with sufficient read scope to query the configuration management database and per-node monitoring endpoints.
- A Huawei Cloud access key and secret key scoped to the RDS disk auto-expansion API, used for AK/SK request signing.
- A dedicated PostgreSQL monitoring role, provisioned per RDS instance, with the minimal privileges required by `postgres_exporter`.
- An Azure DevOps project with a variable group holding the platform credentials as secret variables, a secure file containing the SSH key used to reach the bastion host, and a self-hosted agent registered against that bastion host.
- A Microsoft Teams incoming webhook for alert delivery.

## Project Folder Structure

```
5-vdc-shared/0-dev/
├── prometheus/
│   ├── dev-configuration.yaml         # Prometheus scrape configuration (three jobs)
│   ├── dev-alerting-rules.yaml        # 29 alerting rules and recording rules
│   ├── custom-queries.yaml            # DBA-requested custom SQL collectors
│   ├── dev-rds-inventory.md           # RDS instance inventory documentation
│   └── metrics-and-thresholds.md      # Metric catalogue and threshold rationale
├── grafana/
│   ├── dev-dashboards.json            # PostgreSQL overview dashboard (DBA audience)
│   ├── dev-infra-dashboard.json       # Host-level fleet overview (platform/infra audience)
│   ├── dev-rds-detail-dashboard.json  # Per-instance drill-down (on-call audience)
│   └── hcs-infra-capacity-dashboard.json # Resource pool and storage capacity planning
├── alertmanager/
│   ├── dev-alertmanager.yml           # Alertmanager routing and receiver configuration
│   └── templates/
│       └── teams.tmpl                 # Microsoft Teams notification template
└── oc-exporter/
    ├── oc_exporter.py                 # Custom OC infrastructure exporter
    ├── oc-exporter.service            # Systemd unit with secret placeholders
    ├── pg_exporter_reconciler.py      # Fleet reconciler (auto-discovery)
    ├── pg-exporter-reconciler.service # Reconciler systemd unit
    ├── pg-exporter-reconciler.timer   # Systemd timer, five-minute interval
    ├── postgres_exporter.service.j2   # Template for per-RDS exporter units
    ├── pg-exporter-overrides.yaml     # Per-instance overrides (skip, pin, force)
    └── oc_auth_test.py                # Standalone SSO authentication test utility

pipelines/
└── 0-dev-monitoring.yml               # CI/CD pipeline (validate, then deploy over SSH)
```

## Tasks and Implementation Steps

1. **Reverse-engineer the platform authentication flow.** No API token or OAuth path exists for the ManageOne Operation Center, so the browser single sign-on flow was captured and reproduced, including RSA-OAEP encryption of the service account password.
2. **Build the auto-discovery layer.** A paginated query against the platform's configuration management database enumerates every PostgreSQL RDS node, filtered against a configurable list of tenants to exclude.
3. **Expose host-level metrics.** The custom exporter converts the discovered inventory and per-node monitoring data into Prometheus gauges, served on a dedicated exporter port.
4. **Add disk auto-expansion visibility.** A second, independently authenticated integration against the Huawei Cloud RDS API (AK/SK signed) exposes whether autoscaling is enabled, its ceiling and its trigger threshold, so disk alerts can be evaluated relative to the actual autoscaling policy rather than raw usage alone.
5. **Deploy a `postgres_exporter` fleet.** Rather than a single multi-target exporter, each RDS instance receives its own systemd-managed exporter process, keeping failures isolated and labelling clean.
6. **Automate exporter lifecycle with a reconciler.** A script running on a five-minute systemd timer compares the discovered RDS inventory against running exporter units and creates or removes units to match, using Jinja2-style templating and an override file for exceptions.
7. **Define the Prometheus scrape and alerting configuration.** Three scrape jobs (self-monitoring, per-instance PostgreSQL, and the OC exporter) feed twenty-nine alerting rules across five groups, plus recording rules that pre-join metrics across exporters.
8. **Design four audience-specific Grafana dashboards.** Separate dashboards were built for database administrators, platform/infrastructure engineers, on-call incident response and capacity planning, cross-linked for drill-down.
9. **Route alerts to Microsoft Teams.** A custom Alertmanager template renders structured alert annotations into readable Teams cards, and an acknowledgement endpoint on the exporter allows operators to mark an alert as being worked, recorded as a Grafana annotation.
10. **Build a validate-then-deploy CI/CD pipeline.** An Azure DevOps pipeline checks Prometheus configuration, alerting rule syntax, Grafana dashboard JSON and exporter Python syntax on every push or pull request, and only deploys to the bastion host on a push to the main branch.

## Core Implementation Breakdown

### Custom OC Exporter — Host-Level Metrics

The core of the host-level visibility problem is authentication. The ManageOne Operation Center does not accept a plaintext password submission over its login endpoint; it requires the password to be encrypted client-side with RSA-OAEP, matching the behaviour of the platform's own browser login page. The exporter fetches the current public key, encrypts the service account password, submits the login form, and follows the resulting single sign-on redirect chain to establish an authenticated session, extracting a CSRF token for subsequent API calls.

```python
from Crypto.Cipher import PKCS1_OAEP
from Crypto.Hash import SHA256
from Crypto.PublicKey import RSA

def _encrypt_password(self, pubkey_pem):
    """Encrypt password with RSA-OAEP SHA-256, return hex string."""
    key = RSA.import_key(pubkey_pem)
    cipher = PKCS1_OAEP.new(key, hashAlgo=SHA256)
    encrypted = cipher.encrypt(self.password.encode("utf-8"))
    return encrypted.hex()
```

Because the platform's brute-force protection issues a CAPTCHA challenge after a small number of consecutive login failures, an exponential backoff schedule (sixty seconds, then one hundred and twenty, then five hundred, capping at six hundred) governs re-authentication attempts, and a dedicated metric tracks consecutive authentication failures so the team is alerted before a lockout occurs.

Instance discovery queries the platform's configuration management database directly, rather than relying on a static inventory file:

```python
def discover_rds_instances(session):
    """Discover all PostgreSQL RDS nodes from the platform CMDB."""
    all_nodes = []
    page = 1
    while True:
        code, data = session.get(
            "/rest/ies/csmresmgrwebsite/v2/resources/cloud_rdspg_node"
            "?pageNo={}&pageSize={}".format(page, PAGE_SIZE)
        )
        if code != 200:
            break
        items = data.get("responseData", {}).get("data", [])
        for item in items:
            all_nodes.append({
                "res_id": item.get("resId", ""),
                "instance_name": item.get("CLOUD_RDSPG_INSTANCE_Name", {}).get("value", ""),
                "status": item.get("status", "unknown"),
                "tenant": item.get("tenantName", ""),
            })
        if len(items) < PAGE_SIZE:
            break
        page += 1

    all_nodes = [n for n in all_nodes if n["tenant"] not in SKIP_TENANTS]
    return all_nodes
```

A second data source, the Huawei Cloud RDS disk auto-expansion API, uses an entirely different signing scheme (SDK-HMAC-SHA256, based on AK/SK rather than a session cookie). This confirms a point worth stating plainly: two APIs on the same platform can require two different authentication mechanisms, and the exporter code treats them as independent integrations rather than assuming a shared auth layer.

The exporter also gathers infrastructure capacity data — per-hypervisor CPU and memory allocation and per-storage-backend capacity — from two further configuration management database endpoints, aggregated by cluster and exposed as pool-level gauges consumed by the capacity planning dashboard shown below.

### postgres_exporter Fleet — Database-Level Metrics

Each RDS instance runs its own `postgres_exporter` process rather than sharing a single multi-target exporter. This trades a larger process count for clean failure isolation: if one instance becomes unreachable, or its exporter crashes, every other instance in the fleet remains fully monitored. Each systemd unit is rendered from a template and hardened at the service level:

```ini
[Service]
Type=simple
User=root
Environment="DATA_SOURCE_NAME=postgresql://dbi_prometheus:{{PASSWORD}}@{{IP}}:{{PG_PORT}}/postgres?sslmode=disable&connect_timeout=5"
ExecStart=/usr/local/bin/postgres_exporter \
  --web.listen-address=:{{PORT}} \
  --extend.query-path=/opt/postgres_exporter/custom-queries.yaml \
  --no-collector.wal
Restart=on-failure
RestartSec=10
NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true
PrivateTmp=true
```

A five-minute reconciler timer keeps this fleet in sync with the discovered inventory automatically, and an override file allows an operator to skip an instance, pin its exporter to a fixed port for a stable dashboard URL, or force a different listener IP, without touching the reconciler's code.

Three custom SQL collectors were added at the database administration team's request, extending `postgres_exporter` beyond its built-in collectors for clock drift detection, WAL file accumulation and lock contention visibility. Each is flagged to run only against the primary node, avoiding replication-sensitive queries on standbys.

### Prometheus and Alerting Rules

Prometheus scrapes three job types at a fifteen-second interval: itself, the per-instance `postgres_exporter` fleet, and the single OC exporter process covering the whole host-level fleet. Because percentage-of-capacity alerts (for example, disk usage as a percentage of allocated capacity) require joining data from two different exporters, and Prometheus alerting rules cannot perform a cross-job join at evaluation time, a recording rule pre-computes the join:

```yaml
- record: rds_storage_total_bytes
  expr: |
    (pg_up * 0)
    + on(rds_instance) group_left()
      (max by (rds_instance) (oc_rds_disk_total_gigabytes) * 1024 * 1024 * 1024)
```

The `pg_up * 0` term preserves the full `postgres_exporter` label set while contributing a zero value, and the `group_left()` join enriches it with the OC-sourced capacity figure. Downstream alerts can then divide database size by this recorded series without a runtime cross-job join.

Every alert embeds a structured, runbook-style annotation rather than a bare threshold breach:

```yaml
- alert: OCRDSCriticalDisk
  expr: oc_rds_disk_usage_percent > 90
  for: 10m
  labels:
    severity: critical
    priority: P1
  annotations:
    summary: "[P1 CRITICAL] Disk > 90% on {{ $labels.rds_instance }}"
    description: |
      What: Disk usage on {{ $labels.node }} has exceeded 90% for 10 minutes.
      Current: {{ $value | printf "%.1f" }}%.
      Threshold: 90%.
      Impact: PostgreSQL will switch to read-only mode. All writes will fail.
      Likely causes: Unbounded table growth, WAL accumulation, failed vacuum.
    action: |
      1. Connect to the bastion host.
      2. Check database size: SELECT pg_size_pretty(pg_database_size(datname)) FROM pg_database;
      3. Check WAL: SELECT count(*) FROM pg_ls_waldir();
      4. If autoscaling is enabled, check oc_rds_disk_autoscaling_usage_percent.
      5. If autoscaling is disabled, request manual disk expansion.
```

### Grafana Dashboards

Four dashboards were built deliberately, rather than one dashboard attempting to serve every audience, because fifty or more metrics across a fleet of this size becomes unusable in a single view:

| Dashboard | Audience | Purpose |
|---|---|---|
| PostgreSQL Overview | Database administrators | Connection counts, database sizes, deadlocks, cache hit rate |
| Infrastructure Overview | Platform and infrastructure engineers | Host-level fleet CPU, memory, disk and IOPS |
| RDS Instance Detail | On-call engineer | Single-instance deep dive, all metrics, alert acknowledgement |
| HCS Infrastructure Capacity | Capacity planning | Resource pool allocation and storage headroom across clusters |

Colour thresholds are applied consistently across panels (for example, CPU and memory turn amber at seventy per cent and red above ninety per cent) and are set intentionally lower than the corresponding alert thresholds, so the visual state changes before an alert would fire.

The capacity dashboard below shows the Application Data Cluster (ADC) and Storage Data Cluster (SDC) resource pools, with allocated versus total vCPU and memory, and storage usage, tracked over a rolling window:

![HCS Infrastructure Capacity dashboard showing ADC and SDC resource pool CPU, memory and storage allocation](images/hcs-infra-cap.png)

The PostgreSQL database dashboard surfaces per-instance operational health, including version, connection ceiling, average CPU and memory usage, and open file descriptors:

![PostgreSQL Database dashboard showing general counters, CPU usage, memory usage and open file descriptors](images/psql-dash-1.png)

Further down the same dashboard, per-database statistics cover returned rows, idle session states, cache hit rate, background writer buffer activity, conflicts and deadlocks, temporary file usage, and checkpoint timing:

![PostgreSQL Database dashboard showing return data, idle sessions, cache hit rate, buffer activity, conflicts, deadlocks, temp files and checkpoint stats](images/psql-dash-2.png)

### Microsoft Teams Alert Routing

Alertmanager routes every firing and resolved alert to a Microsoft Teams incoming webhook, using a custom Go template that renders the structured `description` and `action` annotations into a formatted card with bold field labels, rather than the default plain-text rendering:

```go
{{ define "teams.text" -}}
{{ range .Alerts.Firing }}
**{{ .Annotations.summary }}**

**Instance:** {{ .Labels.rds_instance }}
**Priority:** {{ .Labels.priority }} {{ .Labels.severity }}

{{ .Annotations.description }}

**Troubleshooting Steps**
{{ .Annotations.action }}
{{ end }}
{{- end }}
```

The exporter also serves an acknowledgement endpoint, called from a link embedded in the dashboard and the Teams card itself, which writes a Grafana annotation and flips an acknowledgement gauge to suppress repeat notifications for that alert and instance pair until the underlying condition resolves.

### CI/CD Pipeline

The monitoring configuration is deployed through a dedicated Azure DevOps pipeline, separate from the Terraform infrastructure pipeline, because it iterates on a different cadence and requires different validation tooling. The pipeline runs in two stages:

```
Stage 1: Validate (runs on pull request and push)
├── promtool check config
├── promtool check rules
├── python3 json.load() against each Grafana dashboard export
└── python3 -m py_compile against the exporter source

Stage 2: Deploy (runs on push to main only)
├── Deploy Prometheus configuration, then restart the service
├── Deploy Grafana dashboards via the Grafana HTTP API
└── Render exporter secrets, then restart the exporter service
```

Secret values (platform credentials, cloud access keys, Grafana administrator password) are stored as secret variables in an Azure DevOps variable group and are substituted into the exporter's systemd unit only at deploy time, never committed to the repository. The substitution step fails the pipeline outright if any placeholder token remains unreplaced after substitution, preventing a service from starting with a literal, non-functional credential.

## Local Testing and Debugging

Validation was performed at three levels before any change reached the bastion host:

- **Static configuration checks**, run both in the pipeline and manually on the bastion prior to a restart:

  ```bash
  promtool check config /etc/prometheus/prometheus.yml
  promtool check rules /etc/prometheus/alerting-rules.yaml
  python3 -m py_compile oc_exporter.py
  ```

- **Exporter output inspection**, confirming the exporter is serving well-formed Prometheus text-format metrics before relying on Prometheus to scrape it:

  ```bash
  curl -s http://localhost:9199/ | head -20
  ```

- **Standalone authentication testing**, using a dedicated `oc_auth_test.py` utility to exercise the single sign-on flow in isolation from the full exporter, which made it possible to diagnose authentication failures (for example, a stale session cookie triggering a CAPTCHA page instead of a login form) without needing to wait for a full scrape cycle.

Dashboard changes were validated by importing the JSON export through the Grafana API against a non-production instance first, and cross-checking rendered panel values against a manual PromQL query in the Prometheus expression browser, before promoting the change through the pipeline.

## IAM Role and Permissions (Optional)

Huawei Cloud Stack does not use an AWS-style IAM role model, so access control here takes three complementary forms:

- **Least-privilege database access.** A dedicated PostgreSQL role, provisioned per instance specifically for monitoring, is used by `postgres_exporter`, rather than an administrative account. It is scoped only to the read access `postgres_exporter` and the custom collector queries require.
- **AK/SK-scoped cloud API access.** The Huawei Cloud access key and secret key used for disk auto-expansion queries are scoped to the RDS API surface only, and are stored exclusively as Azure DevOps secret variables, never in configuration files or source control.
- **Pipeline-level secret isolation.** All platform, database and Grafana credentials live in a single Azure DevOps variable group with secret-flagged values, injected into the exporter's systemd environment only at deploy time and validated against a leftover-placeholder check before the service is allowed to start. SSH access from the pipeline agent to the bastion host uses strict host-key verification with a dynamically captured key fingerprint, rather than disabling host-key checking.

## Design Decisions and Highlights

| Decision | Alternatives Considered | Rationale |
|---|---|---|
| Custom exporter against the platform's monitoring API, instead of `node_exporter` or CloudEye | Requesting VM-level access from the platform team (rejected, breaks the managed-service model); CloudEye integration (limited metric set, no auto-discovery) | The platform API exposes twenty-seven or more host-level metrics at sixty-second granularity with auto-discovery, which no standard exporter can reach on a fully managed RDS service |
| Reverse-engineered RSA-OAEP single sign-on flow | Plaintext password submission (rejected by the platform); requesting a service token (not available for this platform) | This is the only supported authentication path; the implementation includes exponential backoff to avoid triggering the platform's CAPTCHA-based brute-force protection |
| One `postgres_exporter` process per RDS instance | A single multi-target exporter | Trades additional process count for failure isolation and clean per-instance Prometheus labelling across a large fleet |
| Reconciler-driven fleet management on a five-minute timer | Manual systemd unit creation per instance; static Prometheus file-based service discovery | Eliminates configuration drift entirely; new instances are monitored without human intervention |
| Four audience-specific Grafana dashboards | A single combined dashboard | A dashboard covering fifty or more metrics across the entire fleet is unusable for any one audience; separation keeps each dashboard fast and relevant, with cross-links for drill-down |
| Recording rule for cross-exporter joins | Runtime cross-job joins in alerting expressions | Prometheus alerting rules cannot natively join two different jobs at evaluation time without a pre-computed series |
| Microsoft Teams webhook over PagerDuty or email | Email (slow, easily missed); PagerDuty or Opsgenie (additional vendor cost and approval) | Zero additional licensing, delivered directly into the tool the operations team already works in |
| Separate monitoring pipeline from the Terraform infrastructure pipeline | A single combined pipeline | Monitoring configuration iterates on a different cadence and needs different validation tooling (`promtool` rather than a Terraform plan); a dashboard change now deploys in under two minutes |
| Static threshold disk alerts over `predict_linear` extrapolation | Linear prediction of time-to-disk-exhaustion | Bursty workloads such as batch extract-transform-load jobs produced short spikes that extrapolated into false positives; a simple threshold with a hold duration proved more reliable in practice |

## Errors Encountered and Resolved (Optional)

| Issue | Root Cause | Resolution |
|---|---|---|
| Single sign-on authentication failed silently, with no session cookie returned | A leftover session cookie from a prior failed attempt caused the platform to return a CAPTCHA page instead of a login form | The cookie jar and HTTP opener are reset on every authentication attempt |
| CAPTCHA lockout after three consecutive authentication failures | The platform's brute-force protection cannot be solved by a headless exporter | Implemented exponential backoff (60s, 120s, 300s, capped at 600s), with a metric that alerts the team once consecutive failures exceed two |
| Cross-job metric joins failed inside alerting rules | The two exporters used different label sets for the same logical instance | Added a consistent `rds_instance` label to both scrape jobs and introduced a recording rule using `group_left()` |
| Disk-fill prediction alerts fired on healthy instances | `predict_linear` extrapolated short, bursty write spikes into an imminent-exhaustion forecast | Disabled the predictive rule and replaced it with static threshold alerts at eighty and ninety per cent, each with a hold duration |
| Grafana dashboard import failed on redeployment | The dashboard JSON `id` field conflicted with the existing dashboard already stored in Grafana | The pipeline strips the `id` field from the payload and sets `overwrite: true` before import |
| Platform session expired mid-scrape cycle | The platform invalidates sessions after roughly twenty-five minutes of inactivity | Session time-to-live was set to twenty minutes in the exporter, refreshing proactively ahead of expiry |
| Tenant filtering silently failed for some nodes | The tenant field was `None` for certain nodes, breaking a direct string comparison | Replaced the filter with a null-safe comparison |
| Teams notifications rendered as unreadable raw text | The default Alertmanager template does not format multi-line annotation text | Built a custom Go template using `reReplaceAll` to bold structured field names |
| The exporter systemd service started with literal placeholder text in place of a credential | An Azure DevOps variable group was missing a secret, or an environment variable name did not match | Added a fail-safe check that scans the rendered file for any remaining placeholder pattern and exits the pipeline with an error before the service is allowed to start |
| SSH connections from the pipeline agent to the bastion host failed host-key verification on first deployment | The pipeline initially disabled host-key checking entirely, which was then hardened and began rejecting unknown hosts | Replaced with strict host-key checking and a dynamic key scan performed at the start of the job, run once against a trusted bootstrap |

## Skills Demonstrated

- **Observability engineering**: designed and implemented a full metrics pipeline (collection, storage, alerting, visualisation, notification) from first principles against a platform with no native exporter support.
- **API integration and authentication engineering**: reverse-engineered a browser-based single sign-on flow requiring client-side RSA-OAEP encryption, and implemented a second, independently signed integration (AK/SK, SDK-HMAC-SHA256) against the same platform vendor's separate API surface.
- **Python systems programming**: built a production exporter and a fleet reconciler, including pagination, retry and backoff logic, template rendering, and a lightweight HTTP endpoint for alert acknowledgement.
- **PromQL and Prometheus internals**: authored recording rules to solve cross-exporter joins, and twenty-nine alerting rules with tuned thresholds, hold durations and structured, actionable annotations.
- **Grafana dashboard design**: designed four dashboards for four distinct audiences, with consistent threshold colour semantics and drill-down navigation between fleet and instance views.
- **CI/CD and infrastructure automation**: built a validate-then-deploy Azure DevOps pipeline with syntax and configuration checks ahead of a controlled, SSH-based deployment, including a fail-safe secret substitution check.
- **Security-conscious operational design**: least-privilege database roles, secret isolation through pipeline variable groups, strict SSH host-key verification, and exponential backoff to avoid tripping a third-party brute-force protection mechanism.
- **Incident response tooling**: structured, runbook-quality alert content delivered directly into the team's existing communication channel, with an in-dashboard acknowledgement workflow.

## Conclusion

This platform closes a genuine visibility gap on a managed database service by building the integrations the platform itself does not provide. The emphasis throughout was on removing manual toil (auto-discovery, fleet reconciliation, validate-before-deploy), on making the operational experience during an incident as fast as possible (structured alerts, Teams delivery, one-click acknowledgement), and on keeping the system trustworthy over time (disabling alerts that produced false positives rather than letting them be ignored, and validating configuration in two separate stages before it reaches a running service). The result is a fleet of over one hundred and thirty seven PostgreSQL instances monitored from a single, auto-discovering pipeline, with proactive alerting that did not exist before this work.

---

### Suggested Repository Names

The following names describe the platform accurately without relying on any employer or internal identifier. Pick whichever best matches your existing portfolio naming convention:

1. `hcs-dbaas-observability-platform`
2. `postgresql-fleet-monitoring-hcs`
3. `prometheus-grafana-managed-rds-observability`
4. `hcs-metrics-exporter-and-dashboards`
5. `dbaas-monitoring-pipeline-huawei-cloud`
6. `rds-fleet-observability-toolkit`

If you intend to reference the employer by name in a private or access-controlled repository, `absa-dbaas-monitoring-platform` is also accurate, but is not recommended for a public-facing portfolio repository
.