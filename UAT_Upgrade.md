# UAT ROSA/OpenShift Upgrade --- 4.19.45 to 4.20.38

**Environment:** UAT\
**Platform:** Red Hat OpenShift Service on AWS (ROSA)\
**Upgrade:** OpenShift 4.19.45 → 4.20.38\
**Upgrade Status:** **Completed successfully**\
**Outstanding Issue:** Intermittent Console / `*.apps` route
connectivity\
**Purpose:** Document upgrade blockers, observations, actions,
resolutions, and post-upgrade follow-up items.

------------------------------------------------------------------------

## 1. Executive Summary

The UAT ROSA cluster was upgraded from **OpenShift 4.19.45 to 4.20.38**.

Before the upgrade, two significant issues were identified:

1.  The **Monitoring operator** initially blocked the upgrade because
    User Workload Monitoring used the reserved Prometheus external label
    `cluster`.
2.  The **Console operator** intermittently reported route-health
    failures caused by timeouts reaching the Console `*.apps` route.

The Monitoring issue was corrected before the upgrade. The 4.20.38
upgrade was then accidentally scheduled/initiated while the intermittent
Console route issue was still being investigated.

Despite transient Console and Ingress health failures during the
upgrade, the control plane and Machine Config Pools successfully
completed the 4.20.38 update. The nodes completed the Kubernetes upgrade
from **v1.32.13 to v1.33.13**.

The OpenShift upgrade is complete. However, the Console operator
continues to intermittently become `Available=False / Degraded=True`
because the Console route times out while awaiting HTTP response
headers. Earlier AWS investigation also identified NLB targets reporting
`Target.FailedHealthChecks`.

**Overall upgrade status: SUCCESSFUL**\
**Current OpenShift version: 4.20.38**\
**Primary outstanding issue: Intermittent Console / ingress route
connectivity**

------------------------------------------------------------------------

## 2. Initial Cluster State

Initial OpenShift version:

``` text
4.19.45
```

Initial Machine Config Pools were healthy:

``` text
master:
UPDATED=True
UPDATING=False
DEGRADED=False

worker:
UPDATED=True
UPDATING=False
DEGRADED=False
```

Most ClusterOperators were:

``` text
AVAILABLE=True
PROGRESSING=False
DEGRADED=False
```

Two areas required investigation before upgrading:

-   Monitoring `Upgradeable=False`
-   Intermittent Console route-health failures

------------------------------------------------------------------------

## 3. Issue 1 --- Monitoring Operator Blocking 4.20 Upgrade

### Problem

The Monitoring ClusterOperator was healthy on 4.19.45 but reported:

``` text
Upgradeable=False
```

The User Workload Monitoring configuration contained:

``` yaml
prometheus:
  externalLabels:
    cluster: pathnc-pp
    env: uat
```

OpenShift identified `cluster` as a reserved external label. Reserved
labels included:

``` text
prometheus
prometheus_replica
cluster
```

### Root Cause

The custom Prometheus external label:

``` yaml
cluster: pathnc-pp
```

conflicted with an OpenShift-reserved label and prevented the Monitoring
operator from reporting `Upgradeable=True`.

### Resolution

The User Workload Monitoring ConfigMap was updated to:

``` yaml
prometheus:
  externalLabels:
    cluster_name: pathnc-pp
    env: uat
```

After reconciliation:

``` text
monitoring
Available=True
Progressing=False
Degraded=False
Upgradeable=True
```

### Follow-up

Review external Prometheus, Thanos, and Grafana queries that previously
used:

``` promql
cluster="pathnc-pp"
```

and update them where applicable to:

``` promql
cluster_name="pathnc-pp"
```

**Status:** RESOLVED

------------------------------------------------------------------------

## 4. Issue 2 --- Intermittent Console Route Health Failure

### Problem

Before the 4.20 upgrade, the Console operator intermittently became
unavailable.

Example:

``` text
console
Available=False
Progressing=False

RouteHealthAvailable:
failed to GET route
https://console-openshift-console.apps.pathnc-pp.7ap4.p1.openshiftapps.com

context deadline exceeded
(Client.Timeout exceeded while awaiting headers)
```

The condition periodically recovered and then returned.

### Observation

The Console application itself appeared healthy. Console pods were
running and Console service endpoints were available.

The failing path was:

``` text
Console Operator
       |
       v
Console *.apps DNS
       |
       v
Load Balancer / network
       |
       v
router-default
       |
       v
console Service
       |
       v
console Pods
```

This indicated that pod health alone did not explain the route-health
failure.

**Status:** UNDER INVESTIGATION

------------------------------------------------------------------------

## 5. Ingress Investigation

The default IngressController initially reported healthy conditions:

``` text
Admitted=True
DeploymentAvailable=True
LoadBalancerReady=True
DNSReady=True
Available=True
Progressing=False
Degraded=False
Upgradeable=True
```

Two `router-default` pods were running on infrastructure nodes:

``` text
ip-10-64-46-132.ec2.internal
ip-10-64-46-149.ec2.internal
```

Both infra/router nodes were observed in:

``` text
us-east-1b
```

### Architecture Observation

Only two infra nodes were identified for the default ingress routers,
and both were in the same Availability Zone.

This was recorded as an architectural/resiliency observation. No
topology changes were made during the upgrade or incident investigation.

------------------------------------------------------------------------

## 6. AWS NLB Configuration Observations

The `router-default` Service used an internal AWS Network Load Balancer.

Relevant configuration:

``` text
Service Type: LoadBalancer
externalTrafficPolicy: Local

HTTPS NodePort: 31200
HTTP NodePort: 31450
HealthCheckNodePort: 30796
Health Check Path: /healthz
```

The AWS NLB was active and the target groups used:

``` text
TCP/31200 - HTTPS traffic
TCP/31450 - HTTP traffic
```

with health checks against:

``` text
HTTP :30796/healthz
```

------------------------------------------------------------------------

## 7. AWS Target Health Failures

During investigation, AWS reported multiple NLB targets as:

``` text
unhealthy

Reason:
Target.FailedHealthChecks

Description:
Health checks failed
```

At least one HTTPS target was simultaneously reported as healthy.

Because the Service uses:

``` text
externalTrafficPolicy: Local
```

an unhealthy registered node was not treated by itself as proof of a
faulty router. Nodes without a local router endpoint can behave
differently with Local traffic policy.

No targets were manually deregistered.

**Status:** FOLLOW-UP REQUIRED

------------------------------------------------------------------------

## 8. DNS Observations

During troubleshooting, the Console route resolved to:

``` text
10.64.46.206
10.64.46.249
```

The `router-default` NLB hostname resolved during testing to:

``` text
10.64.46.152
```

The difference was recorded for further OpenShift DNS / Route53
investigation.

No DNS configuration was changed during the upgrade.

The recurring failure was:

``` text
Client.Timeout exceeded while awaiting headers
```

rather than an explicit DNS lookup failure.

------------------------------------------------------------------------

## 9. Accidental Initiation of the 4.20.38 Upgrade

While the Console route-health issue was still under investigation, the
4.20.38 upgrade was accidentally scheduled/initiated.

CVO reported:

``` text
ReleaseAccepted=True
Reason=PayloadLoaded

Payload loaded version="4.20.38"
```

Initial observed progress:

``` text
Working towards 4.20.38:
142 of 962 done
14% complete
```

At that stage:

``` text
Available=True
Failing=False
Progressing=True
```

Because the upgrade had already started and was progressing without a
CVO failure, it was allowed to continue.

### Actions deliberately avoided

-   No forced rollback
-   No manual kubelet updates
-   No arbitrary operator restarts
-   No router pod deletion
-   No Console pod deletion
-   No MCP pause during the active update
-   No manual NLB changes
-   No NLB target deregistration

------------------------------------------------------------------------

## 10. Control Plane Upgrade

Early in the upgrade, several operators reached 4.20.38:

``` text
config-operator     4.20.38
etcd                4.20.38
kube-apiserver      4.20.38
```

The following temporarily reported `Progressing=True`:

``` text
kube-controller-manager
kube-scheduler
```

Example status:

``` text
NodeInstallerProgressing

kube-controller-manager:
3 nodes at revision 57
0 nodes achieved revision 59

kube-scheduler:
3 nodes at revision 47
0 nodes achieved revision 48
```

Neither operator was Degraded.

### Resolution

No manual intervention was performed. Both operators subsequently
completed their 4.20.38 rollout.

**Status:** RESOLVED

------------------------------------------------------------------------

## 11. Transient Ingress Degradation During Upgrade

During the upgrade, the Ingress ClusterOperator temporarily became:

``` text
Degraded=True
```

The condition reported:

``` text
CanaryChecksSucceeding=False
CanaryChecksRepetitiveFailures
```

with:

``` text
error sending canary HTTP Request:

Timeout:
Get "https://canary-openshift-ingress-canary.apps.pathnc-pp.7ap4.p1.openshiftapps.com":

context deadline exceeded
(Client.Timeout exceeded while awaiting headers)
```

This was significant because the same symptom had already been observed
against the Console route.

Affected routes therefore included:

``` text
console-openshift-console.apps...
canary-openshift-ingress-canary.apps...
```

This strengthened the hypothesis that the issue involved the common
`*.apps` ingress / load-balancer / network path rather than the Console
application alone.

### Resolution

No manual remediation was performed during the upgrade.

Ingress subsequently recovered:

``` text
ingress
VERSION=4.20.38
AVAILABLE=True
PROGRESSING=False
DEGRADED=False
```

**Status:** TRANSIENTLY RECOVERED --- ROOT CAUSE FOLLOW-UP REQUIRED

------------------------------------------------------------------------

## 12. Machine Config / Node Upgrade

Later, CVO reported:

``` text
waiting on machine-config
```

Machine Config Operator:

``` text
machine-config
Available=True
Progressing=True
Degraded=False
Working towards 4.20.38
```

The Machine Config Pools entered the expected updating state:

``` text
master:
UPDATED=False
UPDATING=True
DEGRADED=False

worker:
UPDATED=False
UPDATING=True
DEGRADED=False
```

Nodes progressively moved from:

``` text
Kubernetes v1.32.13
```

to:

``` text
Kubernetes v1.33.13
```

No MCP degradation was observed during this upgrade.

------------------------------------------------------------------------

## 13. Final Machine Config Status

At completion:

``` text
master:
CONFIG=rendered-master-e14bc1ab688a3d7458c7f7ac071314c6
UPDATED=True
UPDATING=False
DEGRADED=False
MACHINECOUNT=3
READYMACHINECOUNT=3
UPDATEDMACHINECOUNT=3
DEGRADEDMACHINECOUNT=0
```

Worker:

``` text
CONFIG=rendered-worker-bcda7b9d19298eff539b4432c1e2677e
UPDATED=True
UPDATING=False
DEGRADED=False
MACHINECOUNT=7
READYMACHINECOUNT=7
UPDATEDMACHINECOUNT=7
DEGRADEDMACHINECOUNT=0
```

All nodes shown in the final validation were running:

``` text
v1.33.13
```

**Status:** RESOLVED

------------------------------------------------------------------------

## 14. Worker Count Change

Before/during the upgrade, the worker MCP showed:

``` text
MACHINECOUNT=8
```

Post-upgrade it showed:

``` text
MACHINECOUNT=7
```

An earlier node inventory contained 11 nodes while the final inventory
contained 10.

The reason was not conclusively established during troubleshooting.

### Follow-up Commands

``` bash
oc get machines -n openshift-machine-api -o wide
oc get machinesets -n openshift-machine-api
oc get nodes -o wide
```

Review Machine API events if the reduction was unexpected.

**Status:** FOLLOW-UP REQUIRED

------------------------------------------------------------------------

## 15. Upgrade Completion

The upgrade ultimately reported:

``` text
Cluster version is 4.20.38
```

The Machine Config Pools completed successfully and ClusterOperators
reached 4.20.38.

**Upgrade result:** SUCCESSFUL

------------------------------------------------------------------------

## 16. OLM `Upgradeable=False` After Upgrade

After reaching 4.20.38:

``` text
Upgradeable=False
Reason: IncompatibleOperatorsInstalled
```

The following CSVs were identified:

``` text
openshift-logging/cluster-logging.v6.2.11
Maximum supported OCP version: 4.20

rtf-uat/runtime-fabric-operator.v3.0.1
Maximum supported OCP version: 4.20
```

The message explicitly stated:

``` text
ClusterServiceVersions blocking minor version upgrades
to 4.21.0 or higher
```

### Interpretation

This did **not** indicate failure of the 4.20.38 upgrade.

It currently prevents a future:

``` text
4.20 -> 4.21
```

minor-version upgrade.

### Required Action Before 4.21

Review and upgrade, where supported:

-   `cluster-logging.v6.2.11`
-   `runtime-fabric-operator.v3.0.1`

to versions compatible with OCP 4.21 before scheduling a future minor
upgrade.

**Status:** NO ACTION REQUIRED FOR 4.20.38; ACTION REQUIRED BEFORE 4.21

------------------------------------------------------------------------

## 17. Post-Upgrade Console Failure

Although the platform reached 4.20.38, the Console operator again became
unhealthy afterward.

ClusterVersion:

``` text
VERSION=4.20.38
AVAILABLE=True
PROGRESSING=False

STATUS:
Error while reconciling 4.20.38:
the cluster operator console is not available
```

Console:

``` text
VERSION=4.20.38
AVAILABLE=False
PROGRESSING=False
DEGRADED=True
```

Message:

``` text
RouteHealthAvailable:

failed to GET route
https://console-openshift-console.apps.pathnc-pp.7ap4.p1.openshiftapps.com

context deadline exceeded
(Client.Timeout exceeded while awaiting headers)
```

At the same time:

``` text
authentication   Available=True  Degraded=False
ingress          Available=True  Degraded=False
```

Because the same Console route-health issue existed before the upgrade,
it is being treated as a separate pre-existing/intermittent connectivity
issue rather than evidence that the 4.20.38 upgrade failed.

**Status:** OPEN / UNDER INVESTIGATION

------------------------------------------------------------------------

## 18. Current Working Hypothesis

Evidence collected so far points toward an intermittent problem
somewhere in the common ingress route path:

``` text
Console Operator
       |
       v
*.apps DNS
       |
       v
AWS internal NLB / network path
       |
       v
router-default
       |
       v
OpenShift Service
       |
       v
Application Pod
```

Supporting observations:

1.  Console pods and service appeared healthy.
2.  Console route intermittently timed out before the upgrade.
3.  AWS NLB reported `Target.FailedHealthChecks`.
4.  During the 4.20 rollout, Ingress Canary checks also timed out.
5.  Ingress subsequently recovered without manual remediation.
6.  After successful 4.20 completion, Console route timeouts returned.
7.  Ingress and Authentication could be healthy while the Console
    route-health check failed.

This makes an isolated Console pod/application failure less likely.

------------------------------------------------------------------------

## 19. Current Status Summary

  -----------------------------------------------------------------------
  Component               Status                  Notes
  ----------------------- ----------------------- -----------------------
  OpenShift               **4.20.38**             Upgrade completed

  Control Plane           **Healthy**             Operators reached
                                                  4.20.38

  Master MCP              **Healthy**             Updated=True,
                                                  Degraded=False

  Worker MCP              **Healthy**             Updated=True,
                                                  Degraded=False

  Nodes                   **v1.33.13**            Node upgrade completed

  Monitoring              **Resolved**            Reserved external label
                                                  corrected

  Ingress                 **Healthy currently**   Transient canary
                                                  failure observed

  Authentication          **Healthy**             Available=True

  Console                 **Intermittently        Route timeout
                          degraded**              

  AWS NLB                 **Needs investigation** FailedHealthChecks
                                                  previously observed

  OLM                     **4.21 compatibility    Does not invalidate
                          warning**               4.20

  Worker count            **Follow-up**           8 → 7 workers observed
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 20. Recommended Console / Ingress Follow-up

### Validate Console resources

``` bash
oc get pods -n openshift-console -o wide
oc get svc console -n openshift-console
oc get endpoints console -n openshift-console
oc get route console -n openshift-console -o wide
oc get pods -n openshift-console-operator -o wide
```

### Capture Console operator conditions

``` bash
oc get co console -o json | \
jq -r '.status.conditions[] |
"\(.type)=\(.status) reason=\(.reason) message=\(.message)"'
```

### Repeated Console route test

``` bash
CONSOLE=$(oc get route console -n openshift-console \
  -o jsonpath='{.spec.host}')

for i in $(seq 1 10); do
  date
  curl -kso /dev/null \
    --connect-timeout 5 \
    --max-time 10 \
    -w 'IP=%{remote_ip} CODE=%{http_code} CONNECT=%{time_connect}s TOTAL=%{time_total}s ERROR=%{errormsg}\n' \
    "https://${CONSOLE}/"
  sleep 2
done
```

### Test each DNS IP independently

``` bash
dig +short "$CONSOLE"

for IP in $(dig +short "$CONSOLE" | grep -E '^[0-9.]+$'); do
  echo "===== ${IP} ====="

  for i in $(seq 1 5); do
    curl -kso /dev/null \
      --connect-timeout 5 \
      --max-time 10 \
      --resolve "${CONSOLE}:443:${IP}" \
      -w 'IP=%{remote_ip} CODE=%{http_code} CONNECT=%{time_connect}s TOTAL=%{time_total}s ERROR=%{errormsg}\n' \
      "https://${CONSOLE}/"
    sleep 2
  done
done
```

------------------------------------------------------------------------

## 21. AWS Follow-up

Recheck current NLB target health and map each EC2 target back to its
OpenShift node.

For each target capture:

-   EC2 Instance ID
-   Private IP
-   OpenShift node
-   Availability Zone
-   Whether a local router pod is present
-   Target health
-   Target health reason

Pay particular attention to:

``` text
Target.FailedHealthChecks
```

and determine whether unhealthy targets are expected because of:

``` text
externalTrafficPolicy: Local
```

or represent an actual connectivity/health-check problem.

**Do not manually deregister NLB targets until the mapping and
health-check path are understood.**

------------------------------------------------------------------------

## 22. Preventive Actions for Future Upgrades

Before the next minor upgrade:

1.  Verify `oc adm upgrade` and all `Upgradeable` conditions.
2.  Resolve unexpected `Upgradeable=False` conditions before scheduling.
3.  Check all ClusterOperators for `Available`, `Progressing`, and
    `Degraded`.
4.  Validate User Workload Monitoring configuration against
    target-version restrictions.
5.  Validate Console, Authentication, and Ingress route health.
6.  Check AWS NLB target health.
7.  Verify all MCPs are healthy.
8.  Confirm all nodes are `Ready`.
9.  Review installed OLM Operators for target-version compatibility.
10. Do not schedule the upgrade until outstanding infrastructure/network
    issues have been reviewed.

For the next **4.20 → 4.21** upgrade, `cluster-logging` and
`runtime-fabric-operator` compatibility must be addressed first.

------------------------------------------------------------------------

## 23. Final Conclusion

The **UAT ROSA/OpenShift upgrade from 4.19.45 to 4.20.38 completed
successfully**. The control plane, Machine Config Pools, and nodes
completed their upgrade phases without MCO degradation.

The Monitoring upgrade blocker was resolved by replacing the reserved
Prometheus external label:

``` text
cluster
```

with:

``` text
cluster_name
```

The remaining significant issue is an **intermittent Console `*.apps`
route timeout**. Because this behavior was observed before, during, and
after the 4.20 upgrade---and because AWS NLB target-health failures and
transient Ingress Canary failures were also observed---the remaining
investigation should focus on the:

``` text
DNS -> AWS NLB -> router-default -> service/application
```

network path rather than treating it as an unsuccessful OpenShift
upgrade.

### Final Status

-   **Upgrade:** SUCCESSFUL
-   **Current OpenShift Version:** 4.20.38
-   **Node Kubernetes Version:** v1.33.13
-   **Outstanding Priority:** Console / ingress route connectivity
-   **Future 4.21 Blocker:** Operator compatibility
-   **Additional Follow-up:** Validate worker count change from 8 to 7
