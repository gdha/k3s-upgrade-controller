# Upgrade k3s kubernetes cluster with a system upgrade controller

It is advisable to track the security vulnarabilities published by Rancher Labs around k3s. For example, in November 2020 a critical bug was detected in k3s (see [1]). Therefore, it is quite important to be able to update k3s without interupting the k3s cluster, hence this procedure from Rancher Labs.

```bash
$ kubectl get nodes
NAME   STATUS   ROLES    AGE    VERSION
n3     Ready    <none>   117d   v1.19.2+k3s1
n2     Ready    <none>   117d   v1.19.2+k3s1
n4     Ready    <none>   117d   v1.19.2+k3s1
n1     Ready    master   117d   v1.19.2+k3s1
n5     Ready    <none>   117d   v1.19.2+k3s1
```
## CRD installation

We will follow the procedure described in [2] and are required first to install a kubernetes Custom Resource Definition (CRD) [3] followed by creating a Plan.

To find the latest version of the system upgrade controller use the following command:

```bash
curl -s "https://api.github.com/repos/rancher/system-upgrade-controller/releases/latest" | awk -F '"' '/tag_name/{print $4}'
v0.6.2
```

To download the CRD locally execute:

```bash
$ wget https://raw.githubusercontent.com/rancher/system-upgrade-controller/v0.6.2/manifests/system-upgrade-controller.yaml
```

Now get it applied by:

```bash
$ kubectl apply -f ./system-upgrade-controller.yaml 
namespace/system-upgrade created
serviceaccount/system-upgrade created
clusterrolebinding.rbac.authorization.k8s.io/system-upgrade created
configmap/default-controller-env created
deployment.apps/system-upgrade-controller created

$ kubectl get all -n system-upgrade
NAME                                             READY   STATUS    RESTARTS   AGE
pod/system-upgrade-controller-556df575dd-2qfrs   1/1     Running   0          17s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/system-upgrade-controller   1/1     1            1           17s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/system-upgrade-controller-556df575dd   1         1         1       18s
```
## Making a k3s upgrade plan

Before making a plan we need to decide to which k3s version we need to upgrade, therefore, check out the [GitHub release page of k3s](https://github.com/rancher/k3s/releases). The latest release of this writing was v1.19.4+k3s1 (30 November 2020).

The upgrade plan will upgrade the k3s server node (called k3s-server in the plan) and the k3s worker nodes (called k3s-agent in the plan). For that reason we must first label our master node (in our case *n1*) if that was not yet done:

```bash
$ kubectl get node --selector='node-role.kubernetes.io/master'
NAME   STATUS   ROLES    AGE    VERSION
n1     Ready    master   117d   v1.19.2+k3s1
```

Here we see that node *n1* was laready labelled 'master', however, if that was not yet the case we could realize this by:

```bash
kubectl label node n1 node-role.kubernetes.io/master=true
```

Apply the plan:

```bash
$ kubectl apply -f ./k3s-upgrade-plan.yaml 
plan.upgrade.cattle.io/k3s-server created
plan.upgrade.cattle.io/k3s-agent created
```

Check out if the Plans were added correctly:

```bash
$ kubectl describe plans.upgrade.cattle.io  -n system-upgrade
Name:         k3s-server
Namespace:    system-upgrade
Labels:       k3s-upgrade=server
Annotations:  <none>
API Version:  upgrade.cattle.io/v1
Kind:         Plan
Metadata:
  Creation Timestamp:  2020-11-30T10:53:24Z
  Generation:          1
  Managed Fields:
    API Version:  upgrade.cattle.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
        f:labels:
          .:
          f:k3s-upgrade:
      f:spec:
        .:
        f:concurrency:
        f:cordon:
        f:nodeSelector:
          .:
          f:matchExpressions:
        f:serviceAccountName:
        f:upgrade:
          .:
          f:image:
        f:version:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2020-11-30T10:53:24Z
    API Version:  upgrade.cattle.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
        f:latestHash:
        f:latestVersion:
    Manager:         system-upgrade-controller
    Operation:       Update
    Time:            2020-11-30T10:53:24Z
  Resource Version:  190476
  Self Link:         /apis/upgrade.cattle.io/v1/namespaces/system-upgrade/plans/k3s-server
  UID:               d8bb36e7-7602-4e8f-bda7-b75947752eb1
Spec:
  Concurrency:  1
  Cordon:       true
  Node Selector:
    Match Expressions:
      Key:       k3s-upgrade
      Operator:  Exists
      Key:       k3s-upgrade
      Operator:  NotIn
      Values:
        disabled
        false
      Key:       k3s.io/hostname
      Operator:  Exists
      Key:       k3os.io/mode
      Operator:  DoesNotExist
      Key:       node-role.kubernetes.io/master
      Operator:  In
      Values:
        true
  Service Account Name:  system-upgrade
  Upgrade:
    Image:  rancher/k3s-upgrade
  Version:  v1.19.4+k3s1
Status:
  Conditions:
    Last Update Time:  2020-11-30T10:53:24Z
    Reason:            Version
    Status:            True
    Type:              LatestResolved
  Latest Hash:         e50d232791db24fa7ce5039d6f9cf61238b420aacf2ecd32db7cfce3
  Latest Version:      v1.19.4-k3s1
Events:                <none>


Name:         k3s-agent
Namespace:    system-upgrade
Labels:       k3s-upgrade=agent
Annotations:  <none>
API Version:  upgrade.cattle.io/v1
Kind:         Plan
Metadata:
  Creation Timestamp:  2020-11-30T10:53:24Z
  Generation:          1
  Managed Fields:
    API Version:  upgrade.cattle.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
        f:labels:
          .:
          f:k3s-upgrade:
      f:spec:
        .:
        f:concurrency:
        f:drain:
          .:
          f:force:
        f:nodeSelector:
          .:
          f:matchExpressions:
        f:prepare:
          .:
          f:args:
          f:image:
        f:serviceAccountName:
        f:upgrade:
          .:
          f:image:
        f:version:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2020-11-30T10:53:24Z
    API Version:  upgrade.cattle.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
        f:latestHash:
        f:latestVersion:
    Manager:         system-upgrade-controller
    Operation:       Update
    Time:            2020-11-30T10:53:24Z
  Resource Version:  190477
  Self Link:         /apis/upgrade.cattle.io/v1/namespaces/system-upgrade/plans/k3s-agent
  UID:               db4567a7-f80c-44eb-b5dd-0505159ace87
Spec:
  Concurrency:  2
  Drain:
    Force:  true
  Node Selector:
    Match Expressions:
      Key:       k3s-upgrade
      Operator:  Exists
      Key:       k3s-upgrade
      Operator:  NotIn
      Values:
        disabled
        false
      Key:       k3s.io/hostname
      Operator:  Exists
      Key:       k3os.io/mode
      Operator:  DoesNotExist
      Key:       node-role.kubernetes.io/master
      Operator:  NotIn
      Values:
        true
  Prepare:
    Args:
      prepare
      k3s-server
    Image:               rancher/k3s-upgrade
  Service Account Name:  system-upgrade
  Upgrade:
    Image:  rancher/k3s-upgrade
  Version:  v1.19.4+k3s1
Status:
  Conditions:
    Last Update Time:  2020-11-30T10:53:24Z
    Reason:            Version
    Status:            True
    Type:              LatestResolved
  Latest Hash:         e50d232791db24fa7ce5039d6f9cf61238b420aacf2ecd32db7cfce3
  Latest Version:      v1.19.4-k3s1
Events:                <none>
```

And, for the magic to happen we just have to enable to k3s upgrade with command:

```bash
$ kubectl label node --all k3s-upgrade=enabled
node/n2 labeled
node/n4 labeled
node/n3 labeled
node/n1 labeled
node/n5 labeled
```

To check if the k3s upgrades are kicking in watch with:

```bash
$ kubectl get pods -n system-upgrade -w
NAME                                                              READY   STATUS     RESTARTS   AGE
system-upgrade-controller-556df575dd-2qfrs                        1/1     Running    0          109m
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     Init:0/2   0          73s
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     Init:0/2   0          73s
apply-k3s-server-on-n1-with-e50d232791db24fa7ce5039d6f9cf-2j42m   1/1     Running    0          73s
apply-k3s-server-on-n1-with-e50d232791db24fa7ce5039d6f9cf-2j42m   0/1     Completed   0          75s
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     Init:1/2    0          89s
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     Init:1/2    0          92s
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     Init:1/2    0          104s
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     Init:1/2    0          107s
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     PodInitializing   0          108s
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   1/1     Running           0          110s
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     PodInitializing   0          110s
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   1/1     Running           0          112s
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     Completed         0          2m19s
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     Completed         0          2m21s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Pending           0          0s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Pending           0          0s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Pending           0          0s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Pending           0          0s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Init:0/2          0          0s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Init:0/2          0          0s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Init:0/2          0          18s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Init:0/2          0          30s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Init:1/2          0          33s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Init:1/2          0          42s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Init:1/2          0          47s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Init:1/2          0          61s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     PodInitializing   0          70s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     PodInitializing   0          77s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   1/1     Running           0          79s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   1/1     Running           0          79s
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Completed         0          107s
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Completed         0          109s
apply-k3s-server-on-n1-with-e50d232791db24fa7ce5039d6f9cf-2j42m   0/1     Terminating       0          16m
apply-k3s-server-on-n1-with-e50d232791db24fa7ce5039d6f9cf-2j42m   0/1     Terminating       0          16m
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     Terminating       0          17m
apply-k3s-agent-on-n2-with-e50d232791db24fa7ce5039d6f9cf6-2zb2j   0/1     Terminating       0          17m
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     Terminating       0          17m
apply-k3s-agent-on-n4-with-e50d232791db24fa7ce5039d6f9cf6-sspgt   0/1     Terminating       0          17m
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Terminating       0          16m
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Terminating       0          16m
apply-k3s-agent-on-n5-with-e50d232791db24fa7ce5039d6f9cf6-rtjr7   0/1     Terminating       0          16m
apply-k3s-agent-on-n3-with-e50d232791db24fa7ce5039d6f9cf6-567zz   0/1     Terminating       0          16m
```

As you can see from the sequence first the k3s-server gets updated and thereafter, 2 worker nodes at a time as we requested in the plan description.

After a couple of minutes we see:

```bash
$ kubectl get nodes
NAME   STATUS   ROLES    AGE    VERSION
n1     Ready    master   117d   v1.19.4+k3s1
n2     Ready    <none>   117d   v1.19.4+k3s1
n4     Ready    <none>   117d   v1.19.4+k3s1
n3     Ready    <none>   117d   v1.19.4+k3s1
n5     Ready    <none>   117d   v1.19.4+k3s1
```

We believe it is better to disable the k3s upgrades once it is done with the command:

```bash
$ kubectl label node --all --overwrite k3s-upgrade=disabled
node/n5 labeled
node/n1 labeled
node/n2 labeled
node/n4 labeled
node/n3 labeled
```

If we want to upgrade again just edit the plan [4] again with the correct version of k3s and overwrite the label k3s-upgrade again with keyword *enabled*.

## References

[1] [Rancher Operational Advisory: Attention All Rancher K3s Customers, DB bug requires upgrade](https://support.rancher.com/hc/en-us/articles/360052567151-Rancher-Operational-Advisory-Attention-All-Rancher-K3s-Customers-DB-bug-requires-upgrade#are-you-running-k3s-clusters-if-so-please-read-further-0-0)

[2] [Automate K3s Upgrades with System Upgrade Controller](https://rancher.com/blog/2020/upgrade-k3s-kubernetes-cluster-system-upgrade-controller?hss_channel=tw-309890367)

[3] [CRD system-upgrade-controller](./system-upgrade-controller.yaml)

[4] [k3s Upgrade Plan](./k3s-upgrade-plan.yaml)
