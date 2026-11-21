# Elastic Stack Operations

Technical documentation and operational runbooks for administering, upgrading, monitoring, and troubleshooting the Elastic Stack.

## Documentation

| Document | Description |
| --- | --- |
| [Elasticsearch & Kibana Rolling Upgrade Runbook](docs/elasticsearch/rolling-upgrade-runbook.md) | Field runbook based on an Elasticsearch 9.2.1 to 9.5.6 upgrade, including pre-flight checks, rolling node upgrades, Kibana, Fleet, verification, recovery, and lessons learned. |

## Scope

The repository currently focuses on self-managed Elastic Stack operations, especially:

- Elasticsearch cluster administration and rolling upgrades
- Kibana availability and saved-object migrations
- Fleet Server and Elastic Agent lifecycle management
- Snapshot, recovery, and troubleshooting procedures
- Operational security and configuration hygiene

## Important

The runbook is based on a specific environment: a three-node, multi-role Elasticsearch cluster on a RHEL-family Linux distribution using RPM packages and `yum`. Commands and sequencing must be checked against your own node roles, package installation methods, security configuration, and the target Elastic release.

The runbook includes a publication note with validation points against current Elastic documentation. Read those notes before using it in production. Vendor documentation takes precedence where procedures differ.

## Repository structure

```text
.
├── README.md
└── docs/
    └── elasticsearch/
        └── rolling-upgrade-runbook.md
```

## Official Elastic references

- [Plan your upgrade](https://www.elastic.co/docs/deploy-manage/upgrade/plan-upgrade)
- [Upgrade Elasticsearch](https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/elasticsearch)
- [Upgrade Kibana](https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/kibana)
- [Upgrade Fleet-managed Elastic Agents](https://www.elastic.co/docs/reference/fleet/upgrade-elastic-agent)
- [Snapshot and restore](https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore)
- [Troubleshoot upgrades](https://www.elastic.co/docs/troubleshoot/elasticsearch/troubleshooting-upgrades)

## Contributions

Corrections and additional runbooks should include the Elastic Stack version, deployment type, operating system/package method, prerequisites, validation commands, recovery considerations, and links to the relevant official documentation.
