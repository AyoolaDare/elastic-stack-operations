# Elasticsearch & Kibana Rolling Upgrade Runbook

Nov 21, 2026



> **Publication status: Operational field notes. Review the validation points below before using this procedure in production.**
>
> This document records a real Elasticsearch 9.2.1 to 9.5.6 upgrade and the issues encountered. It is not a substitute for the official Elastic upgrade procedure. The notes below identify places where the original procedure needs environment-specific validation.

## Contents

- [1. Purpose and scope](#1-purpose-and-scope)
- [2. Upgrade order and golden rules](#2-upgrade-order-and-golden-rules)
- [3. Phase 0: Pre-upgrade health check](#3-phase-0-pre-upgrade-health-check)
- [4. Phase 1: Repository and package preparation](#4-phase-1-repository-and-package-preparation)
- [5. Phase 2: Backup and snapshot](#5-phase-2-backup-and-snapshot)
- [6. Phase 3: Rolling Elasticsearch node upgrade](#6-phase-3-rolling-elasticsearch-node-upgrade)
- [7. Phase 4: Kibana upgrade](#7-phase-4-kibana-upgrade)
- [8. Phase 5: Fleet Server and Elastic Agents](#8-phase-5-fleet-server-and-elastic-agents)
- [9. Phase 6: Post-upgrade verification](#9-phase-6-post-upgrade-verification)
- [10. Rollback and recovery](#10-rollback-and-recovery)
- [11. Troubleshooting](#11-troubleshooting-errors-encountered)
- [12. Dos and Don'ts summary](#12-dos-and-donts-summary)
- [13. Security hygiene and follow-up tasks](#13-security-hygiene-and-follow-up-tasks)

## Important validation notes

1. **Elasticsearch node upgrade order:** The original procedure says to upgrade non-master nodes first and the elected master last. Current Elastic guidance orders nodes by role: data tiers first, then nodes that are neither data nor master-eligible, then master-eligible nodes. A node with multiple roles belongs to the first applicable group. Because the example environment gives every node master, data, and ingest roles, validate the exact ordering against the [official Elasticsearch upgrade guide](https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/elasticsearch) before execution. Do not rely on the elected-master shortcut alone.

2. **Yellow cluster during a rolling upgrade:** The runbook contains both a requirement to wait for green before moving to the next node and a note that temporary yellow health can occur while versions are mixed. Elastic's documented process says to wait for shard allocation to finish and the cluster to return green before upgrading the next node. If the cluster remains yellow, inspect allocation explanations and the official [rolling-upgrade troubleshooting guidance](https://www.elastic.co/docs/troubleshoot/elasticsearch/troubleshooting-upgrades). Do not treat yellow as automatically safe or automatically expected.

3. **Elastic Agent upgrade method depends on installation type:** Fleet-managed upgrades are not supported for Linux RPM or DEB installations. Those agents must be upgraded using their package method. Upgrade Fleet Server before moving agents to a newer minor version. Check the [Fleet-managed Agent upgrade documentation](https://www.elastic.co/docs/reference/fleet/upgrade-elastic-agent) and identify the installation type of each agent before starting.

4. **Kibana is not a rolling upgrade:** Shut down all older Kibana instances connected to the cluster before starting the upgraded instances. Kibana must be upgraded after Elasticsearch and aligned to its version. See the [Kibana upgrade guide](https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/kibana).

5. **TLS verification:** The examples below use `curl -k`, which disables certificate verification. For routine production use, prefer `--cacert <CA_PATH>` with the correct trusted CA. Keep `-k` for temporary diagnostics only, where the risk is understood.

6. **Backup and rollback:** Elasticsearch does not support in-place downgrade after an upgrade begins. A rollback requires rebuilding/restoring as appropriate from a compatible pre-upgrade snapshot. Kibana rollback after saved-object migration requires restoring the Kibana feature state from a pre-upgrade snapshot, with all Kibana instances stopped. Review [snapshot and restore](https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore) and [Kibana rollback](https://www.elastic.co/docs/deploy-manage/upgrade/deployment-or-cluster/kibana-roll-back) before relying on the rollback section.

7. **Ingestion and detection availability:** “No downtime for ingestion” is an operational objective, not a guarantee. Kibana downtime affects Kibana-based detection-rule execution. Validate data arrival and detection execution separately, and record any coverage gap.

The procedure below is retained as the original operational sequence. Where it differs from current vendor guidance or your deployment's actual role layout, stop and resolve the difference before execution.

---

## 1. Purpose and scope

This runbook documents a rolling upgrade of an Elasticsearch cluster, Kibana, Fleet Server and Elastic Agents within the 9.x line, with the objective of maintaining ingestion availability. It was written from a real upgrade from 9.2.1 to 9.5.6 and includes the mistakes made along the way.

**Assumed environment**

- 3-node cluster where every node holds the master, data and ingest roles, with one replica per shard
- RHEL-family Linux, RPM packages, managed with `yum`
- HTTPS and security enabled on port 9200; Kibana on a separate host or on one of the nodes
- Elastic Agent managed through Fleet

**Placeholders used in this document**

| Placeholder | Meaning |
| --- | --- |
| `<ES_HOST>` | Address of any healthy Elasticsearch node |
| `<NODE_A>`, `<NODE_B>`, `<NODE_C>` | The three Elasticsearch nodes |
| `<REPO>` | Name of your snapshot repository |
| `<TARGET>` | Target version, for example `9.5.6-1` |
| `<OLD>` | Version you are upgrading from |

All `curl` examples use `-u elastic` with no password so the shell prompts for it. Never type the password into the command line.

## 2. Upgrade order and golden rules

The components must be upgraded in this order, and never the other way round.

1. **Elasticsearch**, one node at a time, non-master nodes first and the elected master last.
2. **Kibana**, only after every Elasticsearch node is on the target version.
3. **Fleet Server**, then the **Elastic Agents** it manages.

**Golden rules**

- The cluster must be `green` with 0 unassigned shards before you touch any node, and again before you move to the next one.
- Kibana must never be newer than the oldest Elasticsearch node. A newer Kibana refuses to connect and detection rules stop running.
- Elasticsearch cannot be downgraded after a node has been upgraded and rejoined. A verified snapshot is your only way back.
- Restart only one node at a time. With one replica, two nodes down means missing shard copies.
- Re-enable shard allocation immediately after each node rejoins. Forgetting this was the main cause of problems in the original upgrade.
- Check the packaging step on every host. Repository configuration differed between hosts and caused most of the delay.

## 3. Phase 0: Pre-upgrade health check

Do not start unless every check below passes. Each command can be run from any node. The settings check is the one most likely to surprise you.

**3.1 Cluster health**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/health?pretty"
```

| Field | Expected | If not |
| --- | --- | --- |
| `status` | `green` | Stop. See the yellow cluster entry in section 11. |
| `number_of_nodes` | 3 | A node is down. Find out why before anything else. |
| `unassigned_shards` | 0 | Find the reason with the allocation explain call below. |
| `unassigned_primary_shards` | 0 | Red-level problem. Do not upgrade. |

**3.2 Node versions and roles**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/nodes?v&h=name,ip,version,master,node.role,heap.percent,cpu,disk.used_percent"
```

All nodes should show the same version (`<OLD>`). Note which node has `*` in the `master` column. That node is upgraded last.

**3.3 Cluster settings, including leftover allocation overrides**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/settings?flat_settings=true&pretty"
```

`cluster.routing.allocation.enable` must **not** appear under `persistent` or `transient`. If it is set to `primaries`, someone left it behind from an earlier maintenance step. Clear it:

```
curl -k -u elastic -X PUT "https://<ES_HOST>:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{"persistent":{"cluster.routing.allocation.enable":null},"transient":{"cluster.routing.allocation.enable":null}}'
```

Expect `"acknowledged":true`. Then find out who set it, because it will come back if a script or colleague re-applies it.

**3.4 Why shards are unassigned, if any**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/allocation/explain?pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_cat/shards?v&h=index,shard,prirep,state,unassigned.reason&s=state:desc" | grep -v STARTED
```

The first call explains one random unassigned shard. Name a specific shard in the request body to explain a particular one. The second call lists only shards that are not started.

**3.5 Deprecations and system feature migration**

```
curl -k -u elastic "https://<ES_HOST>:9200/_migration/deprecations?pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_migration/system_features?pretty"
```

Any deprecation with `"level": "critical"` blocks the upgrade. `warning` entries do not. Every system feature should report `NO_MIGRATION_NEEDED`. If one reports `MIGRATION_NEEDED`, run `POST _migration/system_features` and wait for it to finish first.

**3.6 Index creation versions**

```
curl -k -u elastic "https://<ES_HOST>:9200/_all/_settings?filter_path=*.settings.index.version.created"
```

For an upgrade within the same major version, every index should already be a 9.x index and no reindex is needed.

**3.7 Active recoveries and pending tasks**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/recovery?v&active_only=true"
curl -k -u elastic "https://<ES_HOST>:9200/_cat/pending_tasks?v"
```

Both should be empty. Wait for any rebalancing or recovery to finish.

**Do**

- Run all of these immediately before starting, not the day before.
- Record the output so you can compare later.

**Don't**

- Don't start an upgrade on a yellow cluster. A restart can leave a shard with no copy.
- Don't assume the cluster is healthy because it was earlier. Replica allocation can silently stop.

## 4. Phase 1: Repository and package preparation

Do this on **every** Elasticsearch host and on the Kibana host, before you stop anything. It was the biggest source of delay in the original upgrade: the package was available on one host and "not found" on others.

**4.1 Check the repository on each host**

```
ls /etc/yum.repos.d/
cat /etc/yum.repos.d/elastic-9.x.repo
yum repolist all | grep -i -E 'elastic|kibana|logstash'
```

Expected repository file:

```
[elastic-9.x]
name=Elastic repository for 9.x packages
baseurl=https://artifacts.elastic.co/packages/9.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```

Keep `gpgcheck=1`. If the repository is present but disabled, use `--enablerepo=elastic-9.x` on the install command instead of editing the file.

**4.2 Confirm the target version is visible**

```
sudo yum clean all
sudo yum makecache
yum --showduplicates list available elasticsearch | grep <TARGET>
yum --showduplicates list available kibana | grep <TARGET>
```

Expect a line such as `elasticsearch.x86_64  <TARGET>  elastic-9.x`. Do this for every host. If one host shows no available packages, fix it now.

**4.3 If a host cannot see the package**

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `No match for argument` and only installed packages listed | Repository missing or disabled on that host | Add the repo file, or use `--enablerepo` |
| `makecache` hangs or times out | Host cannot reach the Elastic repository | Test with `curl -sI https://artifacts.elastic.co/packages/9.x/yum/repodata/repomd.xml`. Check proxy settings. |
| Package listed on some hosts, not others | Stale metadata cache, or a different repo file | Run `yum clean all` and compare repo files between hosts |
| Version is hidden | A `versionlock` or `exclude=` line | Check with `yum versionlock list` and `grep -ri exclude /etc/yum.conf /etc/yum.repos.d/` |

The lines about the system not being registered with an entitlement server are harmless and unrelated to Elastic.

**4.4 Offline fallback**

If a host cannot reach the repository, download the RPM and its checksum on a machine that can, copy both over, verify the checksum, then install locally. Use the same RPM on all nodes.

```
shasum -a 512 -c elasticsearch-<TARGET>-x86_64.rpm.sha512
sudo yum localinstall ./elasticsearch-<TARGET>-x86_64.rpm
```

**Do**

- Pin the exact version in the install command.
- Confirm availability on every host before stopping any node.

**Don't**

- Don't run a bare `yum update elasticsearch`. It installs whatever is newest in the repository, which may not be your target.
- Don't disable `gpgcheck` to get past an error.

## 5. Phase 2: Backup and snapshot

This phase is your only way back, because an upgraded Elasticsearch node cannot be downgraded. Confirm both the snapshot and the config backups exist before you stop anything.

**5.1 Confirm a recent, successful snapshot**

```
curl -k -u elastic "https://<ES_HOST>:9200/_snapshot?pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_snapshot/<REPO>/_current?pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_cat/snapshots/<REPO>?v&s=end_epoch:desc&h=id,status,start_time,end_time,indices,failed_shards"
```

The newest snapshot must have status `SUCCESS` and `failed_shards` of 0. If none is recent, take one and wait for it to finish:

```
curl -k -u elastic -X PUT "https://<ES_HOST>:9200/_snapshot/<REPO>/pre_upgrade_$(date +%F)?wait_for_completion=false"
```

Make sure it includes the system indices (`.kibana*`, `.security*`, `.fleet*`) by confirming the snapshot covers feature states.

**5.2 Back up configuration on each host**

```
sudo cp -a /etc/elasticsearch /root/elasticsearch-config-backup-$(date +%F)
sudo cp -a /etc/kibana /root/kibana-config-backup-$(date +%F)
```

The Kibana backup matters most. The encryption keys in `kibana.yml` (`xpack.security.encryptionKey`, `xpack.encryptedSavedObjects.encryptionKey`, `xpack.reporting.encryptionKey`) must stay the same through the upgrade, or encrypted saved objects such as alerting rules and connectors become unreadable.

**5.3 Record the current state**

```
rpm -q elasticsearch kibana
sudo yum history list elasticsearch kibana
```

The yum history gives you the transaction numbers you will need if you have to undo a package change.

**Do**

- Check the snapshot's own state, not just that the command returned.
- Keep a copy of `kibana.yml` somewhere safe.

**Don't**

- Don't upgrade while a snapshot is still running.
- Don't assume the existing snapshot schedule covers you. Verify the last successful run.

## 6. Phase 3: Rolling Elasticsearch node upgrade

Upgrade one node at a time. Do the two non-master nodes first and the elected master last. Repeat steps 6.1 to 6.6 for each node and do not start the next node until step 6.6 passes.

**6.0 Pick the order**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/nodes?v&h=name,ip,version,master"
```

The node with `*` is the elected master. Upgrade it last. Use the address of a node that is **not** the one you are restarting for every `curl` call during that node's turn.

**6.1 Flush**

```
curl -k -u elastic -X POST "https://<ES_HOST>:9200/_flush"
```

Expect a `_shards` summary with `failed` at 0. If you can, pause heavy ingestion first. This is optional on a replicated cluster.

**6.2 Restrict allocation to primaries**

```
curl -k -u elastic -X PUT "https://<ES_HOST>:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{"persistent":{"cluster.routing.allocation.enable":"primaries"}}'
```

This stops Elasticsearch from rebuilding replicas while the node is down. It is also the setting that caused the most trouble: if it is left on, no replicas are ever allocated and the cluster stays yellow (or worse) indefinitely. Step 6.5 removes it. Treat 6.2 and 6.5 as a pair.

**6.3 Stop, upgrade and start the node** (run on that node)

```
sudo systemctl stop elasticsearch
sudo yum install elasticsearch-<TARGET> --enablerepo=elastic-9.x
rpm -q elasticsearch
sudo systemctl daemon-reload
sudo systemctl start elasticsearch
sudo journalctl -u elasticsearch -f
```

Press Ctrl+C when the log shows the node has joined the cluster. `rpm -q` must print the target version before you start the service. Use the `--enablerepo` flag only if the repository is disabled on that host.

If the install says the package is already installed but the node still reports the old version, the old process is still running against the new files. Restart it with `systemctl restart elasticsearch`.

**6.4 Confirm the node rejoined on the new version**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/nodes?v&h=name,ip,version,master,uptime"
```

The upgraded node must show the target version and a short uptime. A cluster with mixed versions is normal until the last node is done.

**6.5 Re-enable allocation. Do this immediately.**

```
curl -k -u elastic -X PUT "https://<ES_HOST>:9200/_cluster/settings" -H 'Content-Type: application/json' -d '{"persistent":{"cluster.routing.allocation.enable":null}}'
```

Then confirm the override is gone from the settings output.

**6.6 Wait for green**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/health?wait_for_status=green&timeout=120s&filter_path=status,unassigned_shards,initializing_shards,relocating_shards"
curl -k -u elastic "https://<ES_HOST>:9200/_cat/recovery?v&active_only=true"
```

With a large cluster (thousands of shards) recovery can take a long time. That is normal and safe. Wait. During a rolling upgrade a replica may not be placed on an older-version node while its primary sits on a newer one, so a short yellow period before the last node is done is expected. It should clear once all nodes match.

**6.7 Check configuration leftovers on each node**

```
sudo find /etc/elasticsearch \( -name '*.rpmnew' -o -name '*.rpmsave' \)
```

Compare any file found with the live config. Do not overwrite your live `elasticsearch.yml` or `jvm.options`.

**What to expect during the master's turn**

When the elected master is stopped, the remaining two nodes elect a new one within seconds. API calls to the stopped node fail briefly. This is normal on a healthy three-node cluster, so use the address of another node.

**Do**

- Restart exactly one node at a time.
- Check the node's version before re-enabling allocation.
- Verify the allocation setting is cleared after every node, not only the last.

**Don't**

- Don't restart a node while the cluster is yellow or red.
- Don't re-enable allocation blindly if a node failed to rejoin. Read its log first.
- Don't run the upgrade on the next node because "the last one worked". Wait for green each time.

## 7. Phase 4: Kibana upgrade

Upgrade Kibana only after the last Elasticsearch node is on the target version and the cluster is green. Doing it earlier fails and takes detection offline.

**7.1 Pre-checks**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/nodes?v&h=name,version,master"
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/health?filter_path=status,unassigned_shards"
rpm -q kibana
sudo cp -a /etc/kibana /root/kibana-config-backup-$(date +%F)
```

All nodes must show the target version, the status must be `green`, and the config backup must exist.

**7.2 Upgrade**

```
sudo systemctl stop kibana
sudo yum install kibana-<TARGET> --enablerepo=elastic-9.x
rpm -q kibana
sudo systemctl start kibana
sudo journalctl -u kibana -f
```

If you run more than one Kibana instance against the same cluster, stop all of them first and upgrade them together. Mixed Kibana versions on the same indices can corrupt the saved-object migration.

**7.3 What to expect on first start**

- Kibana migrates its saved objects. This can take several minutes. Do not restart it while it runs.
- New system indices carrying the new version in their name may appear.
- Warnings about renamed settings (for example a setting replaced by a new name) are not errors. Fix them in `kibana.yml` later.
- The log ends with a line saying Kibana is available.

**7.4 Verify**

Open Kibana in the browser and check the following.

- Security detection rules are enabled and running, and new alerts are created.
- Dashboards and Discover load.
- Stack Management shows no critical items in the Upgrade Assistant.
- Fleet shows the agents.

**7.5 Configuration leftovers**

```
sudo find /etc/kibana \( -name '*.rpmnew' -o -name '*.rpmsave' \)
sudo diff /etc/kibana/kibana.yml /etc/kibana/kibana.yml.rpmnew
```

Your live `kibana.yml` is kept. The `.rpmnew` file is the package default, written alongside it for comparison. Do not copy it over your live file.

**Do**

- Check that the encryption keys in `kibana.yml` are unchanged after the upgrade.
- Remember that Kibana being down means detection rules are not running. Keep the window short and tell your team.

**Don't**

- Don't upgrade Kibana first "to get it done". It will refuse to connect and sit in an error loop.
- Don't restart Kibana during the first-start migration.

## 8. Phase 5: Fleet Server and Elastic Agents

Upgrade Fleet Server first, then the agents it manages. An agent cannot run a newer version than the Fleet Server it reports to.

**8.1 Steps**

1. In Kibana, open **Fleet** and then **Agents**.
2. Select the Fleet Server agent and choose to upgrade it to the target version. Wait until it reports healthy on the new version.
3. Select a small batch of the remaining agents and upgrade them. Watch for any that go unhealthy or offline.
4. Continue in larger batches once the first batch is stable.
5. Check the upgrade progress in the agent list, not only the success message.

**8.2 Check ingestion is still flowing**

After each batch, confirm new documents are still arriving for your main data sources. In Kibana Discover, look at the most recent timestamp for each key data view (for example Windows event logs, Sysmon, firewall and email gateway logs). A gap that starts at the upgrade time points to an agent that did not come back.

**8.3 Common issues**

| Symptom | Likely cause | What to do |
| --- | --- | --- |
| Agent stuck on "Updating" | Cannot download the new agent binary | Check that the agent can reach the artifact source. Check the agent log. |
| Agent goes offline after upgrade | Service did not restart cleanly | Restart the agent service on that host and check its log. |
| Agent newer than Fleet Server | Wrong order | Upgrade Fleet Server first. Roll the agent back if needed. |
| Data stops for one source | Integration or input failed on the new version | Check the agent's integration status in Fleet. |

**Do**

- Upgrade in batches and watch each one.
- Keep the number of agents upgraded at once small until you trust the process.

**Don't**

- Don't upgrade all agents at the same moment.
- Don't upgrade agents before the Fleet Server.

## 9. Phase 6: Post-upgrade verification

The upgrade is not finished until every item below passes. Record the results.

**9.1 Elasticsearch**

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/nodes?v&h=name,version,master"
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/health?pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_cluster/settings?flat_settings=true&pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_migration/deprecations?pretty"
curl -k -u elastic "https://<ES_HOST>:9200/_migration/system_features?pretty"
```

| Check | Expected |
| --- | --- |
| All node versions | Target version on every node |
| Cluster status | `green`, 0 unassigned shards |
| Allocation override | Not present in settings |
| Deprecations | No `critical` entries |
| System features | `NO_MIGRATION_NEEDED` |

**9.2 Kibana and detection**

- Kibana reports the target version and is available.
- Detection rules are enabled and executing, and a recent rule execution succeeded.
- Alerts, cases and dashboards load.
- There is no gap in alert generation across the upgrade window. If Kibana was down at any time, note the gap and review whether any detection coverage needs a back-fill search.

**9.3 Data ingestion**

- The latest document timestamp for each critical data source is recent.
- No agent is stuck unhealthy or offline in Fleet.
- Index and data stream write rates look normal.

**9.4 Housekeeping**

```
sudo find /etc/elasticsearch /etc/kibana \( -name '*.rpmnew' -o -name '*.rpmsave' \)
sudo yum history list elasticsearch kibana
```

Review any leftover config files and record the yum transaction numbers. Update your asset or configuration records with the new versions.

**Do**

- Run the verification even if everything seemed fine during the upgrade.
- Tell the team when the window is closed.

**Don't**

- Don't close the change until the allocation override is confirmed gone and the cluster is green.

## 10. Rollback and recovery

What you can undo depends on how far the upgrade got. Decide quickly, because a stuck Kibana means no detection.

| Situation | Can you roll back? | Approach |
| --- | --- | --- |
| Kibana upgraded too early, never connected to the cluster | Yes | Downgrade the Kibana package. No data was migrated. |
| Kibana upgraded and migrated saved objects | Only from a snapshot | Restore the Kibana system indices from the snapshot, then install the old Kibana. |
| An Elasticsearch node upgraded and rejoined | No (no in-place downgrade) | Go forward. Upgrade the remaining nodes, or restore the whole cluster from a snapshot. |
| An Elasticsearch node failed to start after upgrade | Investigate first | Read its log. Fix config or permissions. Do not re-enable allocation until it rejoins. |

**10.1 Rolling back Kibana that never migrated**

First confirm nothing migrated: the Kibana system indices must still carry only the old version in their names.

```
curl -k -u elastic "https://<ES_HOST>:9200/_cat/indices/.kibana*?v&h=index,health,status,docs.count"
```

Then undo the package change using the yum transaction number you recorded:

```
sudo systemctl stop kibana
sudo yum history list kibana
sudo yum history info <TRANSACTION_ID>
sudo yum history undo <TRANSACTION_ID> --enablerepo=elastic-9.x
rpm -q kibana
sudo systemctl start kibana
```

Expect yum to show a "Downgrading" line for Kibana and, on completion, a `.rpmnew` warning for `kibana.yml`. That warning means your live config was preserved.

**10.2 If a node will not rejoin**

```
sudo systemctl status elasticsearch
sudo journalctl -u elasticsearch -n 200 --no-pager
```

Common causes are a config file overwritten by the package, wrong file ownership, or a certificate path problem. Keep allocation restricted to primaries until the node is back, then re-enable it.

**10.3 Full restore**

If the cluster is damaged, restore from the snapshot you verified in Phase 2. This is a last resort and means data written after the snapshot is lost, so confirm what you can recover first.

## 11. Troubleshooting: errors encountered

These are the problems that occurred during the original upgrade, in the order they appeared.

**11.1 Elasticsearch**

| Symptom or message | Cause | Fix |
| --- | --- | --- |
| Cluster `yellow`, dozens of unassigned replicas, reason `INDEX_CREATED` | `cluster.routing.allocation.enable` left at `primaries` from earlier maintenance. Allocation explain says replica allocations are forbidden by that setting. | Set it to `null` for both `persistent` and `transient`. Replicas then initialize on their own. |
| Unassigned shard count jumped from tens to over a thousand | Nodes were restarted while the setting was still `primaries`, so their replicas dropped out and could not return. | Same fix. Expect a long recovery. Do not restart anything until green. |
| The setting reappeared after being cleared | Someone or something re-applied it, for example a colleague or script starting an upgrade. | Find out who, and agree one person owns the maintenance window. |
| Allocation explain: `same_shard` decider says a copy is already on this node | Normal output. It only means that node already holds the primary. | Read the `enable` decider line. That is the real blocker. |
| `relocating_shards` stays above 0 | Normal rebalancing. | Wait until it reaches 0 before restarting a node. |
| `#! this request accesses system indices` warning in Dev Tools | A query touched `.kibana*`, `.security*` or `.fleet*` directly. | Informational. Use the dedicated APIs where possible. |
| Deprecation about `xpack.monitoring.collection.enabled` | Legacy monitoring still enabled, and legacy monitoring indices still written. | Not a blocker. Plan to move to Elastic Agent based monitoring, and remove the setting only when nothing depends on the old indices. |

**11.2 Packages and repositories**

| Symptom or message | Cause | Fix |
| --- | --- | --- |
| `No match for argument: elasticsearch-<version>` | Repository missing, disabled, or stale metadata on that host. | `yum clean all`, `yum makecache`, then `yum --showduplicates list available`. Compare the repo file with a host where it works. |
| Package visible on one host, not another | Different repo files per host. | Make repo files identical on all hosts. |
| Only "Installed Packages" listed, no "Available Packages" | The Elastic repository is not enabled on that host. | Check `yum repolist all`. Use `--enablerepo=elastic-9.x`. |
| Several repo ids all serving the same Elastic packages | Duplicate repo files. Harmless but redundant. | Keep one and pin versions explicitly. |
| `Unable to read consumer identity` / system not registered with an entitlement server | RHEL subscription-manager plugin message. | Ignore. Unrelated to Elastic. |
| `Package ... is already installed. Nothing to do.` but the node reports the old version | The package was installed but the running process was never restarted. | Restart the service, following the safe sequence in section 6. |
| A running node's process start time is older than the package install time | Same cause as above. | Compare `ps -o lstart` with the package install time. |

**11.3 Kibana**

| Symptom or message | Cause | Fix |
| --- | --- | --- |
| `This version of Kibana is incompatible with the following Elasticsearch nodes in your cluster` | Kibana was upgraded before all Elasticsearch nodes. | Either finish the Elasticsearch upgrade, or roll Kibana back (section 10.1). Detection is down in the meantime. |
| Warning that a `kibana.yml` setting was replaced by a new name | A setting was renamed in the newer version. | Rename it in `kibana.yml` when convenient. Not an error. |
| `kibana.yml created as kibana.yml.rpmnew` | Package default written next to your edited file. | Your live file is kept. Diff the two, do not overwrite. |
| Startup warning about the browser sandbox on this OS | Chromium sandbox unsupported on that OS. Kibana disables it automatically. | Informational. |
| Kibana log prints hashes of encryption keys | Normal startup logging. These are hashes, not the keys. | None. Avoid sharing full logs anyway. |

**11.4 Shell and tooling**

| Symptom | Cause | Fix |
| --- | --- | --- |
| `curl` returns certificate error | Wrong CA path, or self-signed cert. | Pass `--cacert <path>`. For a quick check on a trusted host only, `-k` skips verification. |
| `curl` returns `401` | Wrong user or password. | Use an account with cluster monitor access. |
| `curl: (7) Failed to connect` | Node down, or bound to a different address. | Check `systemctl is-active elasticsearch` and `network.host`. |
| Credentials visible in the shell history | Password typed on the command line (`-u user:password`). | Use `-u user` only and type the password at the prompt. Clear the history entry and rotate the password. |

## 12. Dos and Don'ts summary

| Do | Don't |
| --- | --- |
| Confirm green, 0 unassigned and no allocation override before every node | Start or continue on a yellow or red cluster |
| Take and verify a snapshot, and back up config files | Assume an old snapshot is good enough |
| Check the package is available on every host before stopping anything | Stop a node before you know the package can be installed |
| Pin the exact version in the install command | Run a bare `yum update elasticsearch` |
| Upgrade one node at a time, master last | Restart two nodes together |
| Re-enable allocation immediately after each node rejoins | Leave allocation on `primaries` |
| Upgrade Elasticsearch, then Kibana, then Fleet Server, then agents | Upgrade Kibana before Elasticsearch |
| Keep `gpgcheck=1` and keep `kibana.yml` encryption keys unchanged | Disable signature checks to get past an error |
| Use `-u user` and type the password at the prompt | Put passwords in commands or paste them into chats and tickets |
| Agree one owner for the change window | Let several people change allocation settings independently |
| Verify ingestion and detection after the upgrade | Close the change without checking alerts and data flow |

## 13. Security hygiene and follow-up tasks

**Credential handling**

- Never put a password in a command (`-u user:password`). It lands in shell history, in screen shares and in anything you paste for help.
- If a password was exposed, reset it (`elasticsearch-reset-password -u <user>` on a node, or from Kibana), update every integration that uses it, and remove the history entry with `history -d <number>`.
- Redact tokens, keys, hostnames and addresses before sharing logs or command output outside the team.

**Follow-up tasks after the upgrade**

- [ ] Rotate any credential that was exposed during the change.
- [ ] Review `.rpmnew` and `.rpmsave` files on every host.
- [ ] Rename deprecated `kibana.yml` settings.
- [ ] Decide whether to move from legacy monitoring to agent-based monitoring.
- [ ] Reduce the shard count by consolidating many small daily indices, which speeds up restarts and recovery.
- [ ] Make Elastic repository files identical on all hosts.
- [ ] Remove duplicate or backup repo files that are no longer needed.
- [ ] Review the snapshot schedule and confirm the last successful run.
- [ ] Record the new versions in your asset or configuration records.
- [ ] Review any detection gap that occurred while Kibana was unavailable.

**Pre-flight checklist (print this)**

- [ ] Cluster green, 0 unassigned shards, no allocation override
- [ ] All nodes on the same starting version, elected master identified
- [ ] Target package visible on every host
- [ ] Snapshot verified `SUCCESS`, config backups taken
- [ ] Deprecations show no critical items
- [ ] One owner and one communication channel for the window
- [ ] Team told that detection will be affected if Kibana is down