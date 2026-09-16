# DEV ROSA Cluster Upgrade -- Observations, Issues and Resolutions

## 1. Overview

  -----------------------------------------------------------------------
  Item                                Details
  ----------------------------------- -----------------------------------
  **Environment**                     DEV

  **Platform**                        Red Hat OpenShift Service on AWS
                                      (ROSA)

  **Upgrade**                         OpenShift `4.19.40` → `4.20.33`

  **Upgrade Type**                    Minor version upgrade

  **Primary Issue**                   Machine Config Pools became
                                      degraded because nodes could not
                                      complete drain operations

  **Primary Root Cause**              Stale Dynatrace OneAgent DaemonSet
                                      pods stuck in deletion/termination
                                      state

  **Additional Observations**         Authentication/OpenShift API
                                      degradation, kubelet version skew,
                                      OLM warning for OpenShift Logging

  **Status**                          Remediation in progress / upgrade
                                      progressing
  -----------------------------------------------------------------------

## 2. Executive Summary

During the DEV ROSA upgrade from OpenShift `4.19.40` to `4.20.33`, the
upgrade stalled while the Machine Config Operator (MCO) attempted to
update control-plane and worker nodes.

Both `master` and `worker` MachineConfigPools entered a `Degraded` state
because MCO could not drain specific nodes within the one-hour timeout.

Initial affected nodes:

``` text
Master: ip-10-64-34-146.ec2.internal
Worker: ip-10-64-34-142.ec2.internal
```

MCO logs identified stale Dynatrace OneAgent pods as the drain blocker.
The affected pods were `Succeeded/Completed`, but had remained in
deletion since:

``` text
2026-08-07T19:24:11Z
```

Examples:

``` text
dynakube-oneagent-dsw22
dynakube-oneagent-fpl2x
```

They were owned by `DaemonSet/dynakube-oneagent`, had no finalizers, and
used the Dynatrace CSI driver `csi.oneagent.dynatrace.com`.

Force-removing the confirmed stale, already-terminating pod objects
allowed the affected nodes to continue their MachineConfig update. Nodes
`142` and `146` subsequently advanced from kubelet `v1.32.13` to
`v1.33.13`.

The same stale Dynatrace condition was later identified on additional
nodes, indicating a broader Dynatrace OneAgent/CSI cleanup issue.

## 3. Initial Upgrade Symptoms

The console continued to show:

``` text
This cluster is updating from 4.19.40 to 4.20.33
```

  ------------------------------------------------------------------------------
  Cluster Operator               Status                  Observation
  ------------------------------ ----------------------- -----------------------
  `authentication`               Degraded                1 of 3 API/OAuth
                                                         instances unavailable

  `openshift-apiserver`          Degraded                1 of 3 requested
                                                         instances unavailable

  `kube-apiserver`               Cannot update           Nodes remained on older
                                                         kubelet minor version

  `machine-config`               Cannot update           One or more MCPs
                                                         degraded

  `operator-lifecycle-manager`   Cannot update           Logging 6.3.4 blocks a
                                                         future OCP 4.21+
                                                         upgrade

  Other upgraded operators       Available               Already reporting
                                                         4.20.33
  ------------------------------------------------------------------------------

The primary current upgrade blocker was MachineConfigPool degradation.

## 4. Issue 1 -- Master MCP Drain Failure

### Observation

``` text
Updated:    False
Updating:   True
Degraded:   True

Node ip-10-64-34-146.ec2.internal:
failed to drain node after 1 hour
```

### Investigation

``` bash
oc get pods -A --field-selector spec.nodeName=ip-10-64-34-146.ec2.internal -o wide
```

The control-plane components were running. A stale Dynatrace pod was
identified:

``` text
dynatrace/dynakube-oneagent-dsw22
STATUS: Completed
```

Pod metadata:

``` text
Phase:     Succeeded
Deleting:  2026-08-07T19:24:11Z
Owner:     DaemonSet/dynakube-oneagent
Finalizer: None
CSI:       csi.oneagent.dynatrace.com
```

MCO logs showed repeated eviction attempts followed by timeout waiting
for this pod to terminate.

### Resolution

A normal deletion did not clear the stale object. The
already-terminating pod was force removed:

``` bash
oc delete pod dynakube-oneagent-dsw22 -n dynatrace --grace-period=0 --force
```

Verification:

``` bash
oc get pod dynakube-oneagent-dsw22 -n dynatrace
```

returned `NotFound`.

### Result

The master MCP cleared its degraded condition during subsequent
processing, and node `ip-10-64-34-146.ec2.internal` advanced from
kubelet `v1.32.13` to `v1.33.13`.

## 5. Issue 2 -- Worker MCP Drain Failure

### Observation

``` text
Node ip-10-64-34-142.ec2.internal:
failed to drain node after 1 hour
```

### Investigation

A matching stale Dynatrace pod was found:

``` text
dynakube-oneagent-fpl2x
Phase:     Succeeded
Deleting:  2026-08-07T19:24:11Z
Finalizer: None
Owner:     DaemonSet/dynakube-oneagent
```

MCO logs confirmed that the drain controller was attempting to evict
this pod.

### Resolution

``` bash
oc delete pod dynakube-oneagent-fpl2x -n dynatrace --grace-period=0 --force
```

The pod subsequently returned `NotFound`.

### Result

Worker `ip-10-64-34-142.ec2.internal` advanced to kubelet `v1.33.13`.

## 6. Issue 3 -- Cluster-Wide Stale Dynatrace OneAgent Pods

Additional stale pods were found:

``` text
dynakube-oneagent-4d5v9  -> ip-10-64-34-133
dynakube-oneagent-b55s8  -> ip-10-64-34-132
dynakube-oneagent-g42fj  -> ip-10-64-34-152
dynakube-oneagent-gnfpt  -> ip-10-64-34-153
dynakube-oneagent-mtk9w  -> ip-10-64-34-145
dynakube-oneagent-qbkrw  -> ip-10-64-34-150
dynakube-oneagent-v6rxq  -> ip-10-64-34-134
```

All were observed as `Succeeded` with deletion timestamp
`2026-08-07T19:24:11Z`.

MCO later attempted to evict `dynatrace/dynakube-oneagent-qbkrw`,
demonstrating that the same stale-pod condition could repeatedly block
subsequent node drains.

### Resolution Approach

Only pods confirmed to be `Succeeded`, already deleting,
DaemonSet-owned, and without finalizers should be treated as the same
stale-object condition. Healthy Dynatrace workloads should not be
mass-deleted.

## 7. Issue 4 -- Kubelet Version Skew

Initially:

``` text
9 nodes -> v1.32.13
2 nodes -> v1.33.13
```

After remediation on `142` and `146`, the console warning decreased from
9 nodes to 7.

Last observed:

``` text
v1.33.13:
ip-10-64-34-142
ip-10-64-34-144
ip-10-64-34-146
ip-10-64-34-151

v1.32.13:
ip-10-64-34-132
ip-10-64-34-133
ip-10-64-34-134
ip-10-64-34-145
ip-10-64-34-150
ip-10-64-34-152
ip-10-64-34-153
```

No manual kubelet upgrade was performed. MCO should update kubelet as
each node completes the OpenShift node rollout.

## 8. Issue 5 -- Authentication and OpenShift API Server Degraded

The console reported one of three requested instances unavailable for
authentication/OAuth and OpenShift API server components.

Several OpenShift-managed control-plane PDBs showed
`ALLOWED DISRUPTIONS=0`, including:

``` text
openshift-apiserver-pdb
etcd-guard-pdb
kube-apiserver-guard-pdb
kube-controller-manager-guard-pdb
openshift-kube-scheduler-guard-pdb
oauth-apiserver-pdb
```

### Decision

The OpenShift-managed PDBs were not modified. MCO logs identified
Dynatrace stale pods as the concrete drain blocker, so PDB protections
were left intact.

After MCP completion:

``` bash
oc get co authentication openshift-apiserver kube-apiserver
```

Expected final state:

``` text
AVAILABLE=True
PROGRESSING=False
DEGRADED=False
```

## 9. Issue 6 -- OpenShift Logging / OLM Warning

Observed:

``` text
ClusterServiceVersions blocking minor version upgrades to 4.21.0 or higher:
maximum supported OCP version for
openshift-logging/cluster-logging.v6.3.4 is 4.20
```

Installed CSV:

``` text
cluster-logging.v6.3.4
Phase: Succeeded
```

Observed subscription:

``` text
cluster-logging / redhat-operators / stable-6.1
```

### Assessment

This does not block the current `4.19 → 4.20` upgrade. It is a
compatibility warning for a future `4.20 → 4.21+` upgrade.

### Resolution

No Logging Operator change was made during the current upgrade. Review
the Logging subscription/operator compatibility after `4.20.33` is
stable and before attempting 4.21.

## 10. Root Cause Summary

Confirmed primary root cause:

``` text
MCO begins node drain
        ↓
MCO attempts Dynatrace pod eviction
        ↓
Stale Succeeded pod already marked for deletion does not disappear
        ↓
Drain reaches one-hour timeout
        ↓
NodeDegraded=True
        ↓
MCP Degraded=True
        ↓
Node remains on old kubelet
        ↓
Cluster upgrade remains incomplete
```

Removing the confirmed stale pod objects allowed affected nodes to
resume updating.

## 11. Actions Taken

  -----------------------------------------------------------------------
  Action                              Result
  ----------------------------------- -----------------------------------
  Reviewed `oc describe mcp`          Identified master `146` and worker
                                      `142` drain failures

  Reviewed affected-node workloads    Identified stale Dynatrace OneAgent
                                      pods

  Reviewed PDBs                       OpenShift control-plane PDBs left
                                      unchanged

  Reviewed MCO logs                   Confirmed Dynatrace
                                      eviction/termination timeout

  Inspected pod metadata              Confirmed Succeeded, old deletion
                                      timestamp, DaemonSet owner, no
                                      finalizers

  Normal deletion attempted           Stale pod remained

  Force-deleted                       Master `146` resumed update
  `dynakube-oneagent-dsw22`           

  Force-deleted                       Worker `142` resumed update
  `dynakube-oneagent-fpl2x`           

  Verified kubelet versions           `142` and `146` reached `v1.33.13`

  Investigated remaining OneAgent     Same stale condition identified
  pods                                across additional nodes

  Logging Operator left unchanged     Warning applies to future 4.21
                                      upgrade
  -----------------------------------------------------------------------

## 12. Current / Last Observed DEV Status

``` text
master:
UPDATED=False
UPDATING=True
DEGRADED=True
Machine Count=3
Ready Machine Count=1
Updated Machine Count=1
Degraded Machine Count=1

worker:
UPDATED=False
UPDATING=True
DEGRADED=True
Machine Count=8
Ready Machine Count=3
Updated Machine Count=3
Degraded Machine Count=1
```

At the last observation, 4 of 11 nodes had reached `v1.33.13`, while 7
remained on `v1.32.13`.

## 13. Validation / Exit Criteria

### Cluster Version

``` bash
oc get clusterversion
```

Expected:

``` text
VERSION    AVAILABLE   PROGRESSING
4.20.33    True        False
```

### MachineConfigPools

``` bash
oc get mcp master worker
```

Expected:

``` text
NAME     UPDATED   UPDATING   DEGRADED
master   True      False      False
worker   True      False      False
```

Expected counts:

``` text
master: Ready=3, Updated=3, Degraded=0
worker: Ready=8, Updated=8, Degraded=0
```

### Kubelet Versions

``` bash
oc get nodes -o custom-columns='NAME:.metadata.name,KUBELET:.status.nodeInfo.kubeletVersion'
```

Expected: all nodes on `v1.33.13`.

### Cluster Operators

``` bash
oc get co
```

Confirm that operators related to the current upgrade are healthy and no
longer unexpectedly progressing or degraded.

## 14. Follow-Up / Preventive Actions

1.  Review Dynatrace Operator compatibility with the installed OpenShift
    release.
2.  Review Dynatrace CSI driver health and logs.
3.  Determine why `dynakube-oneagent` DaemonSet pods remained after
    receiving a deletion timestamp.
4.  Check QA and PROD for the same stale Dynatrace condition before
    their OpenShift upgrades.
5.  Review OpenShift Logging Operator/subscription compatibility before
    a future `4.20 → 4.21` upgrade.
6.  Add a pre-upgrade check for pods stuck in deletion/termination
    state.

Useful pre-upgrade check:

``` bash
oc get pods -A   -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,PHASE:.status.phase,NODE:.spec.nodeName,DELETING:.metadata.deletionTimestamp'   | grep -v '<none>'
```

## 15. Lessons Learned

When an MCP becomes degraded during an OpenShift upgrade, investigate
the MCO drain error before modifying PDBs, manually draining nodes, or
changing kubelet.

In this incident:

-   The control-plane PDBs were functioning as protection and were not
    the confirmed root cause.
-   MCO logs exposed the actual blocker: stale Dynatrace OneAgent pods.
-   The kubelet-version warning reflected nodes whose MCO update had not
    completed; kubelet did not require manual upgrading.
-   Removing only confirmed stale, already-terminating Dynatrace pod
    objects allowed MCO to continue.
-   No manual kubelet upgrade, PDB bypass, MCP patch, or manual master
    drain was required.

## 16. Incident Status

**Environment:** DEV\
**Upgrade:** `4.19.40 → 4.20.33`\
**Status:** Remediation in progress / upgrade progressing\
**Confirmed primary issue:** Stale Dynatrace OneAgent pods preventing
MCO node drains\
**Next objective:** Complete all node updates to `v1.33.13`, reach
healthy master/worker MCP states, and perform final ClusterVersion and
ClusterOperator validation.
