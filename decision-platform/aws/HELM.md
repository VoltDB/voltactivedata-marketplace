# Volt Active Decision Platform on AWS Marketplace

The Volt Active Decision Platform is a Helm chart that you install into your own Amazon EKS
cluster. It contains three products:

- **VoltSP**, the stream processing engine that runs your pipeline;
- **VoltDB**, the database, run by its Kubernetes operator;
- the **Volt Management Center (VMC)**, the web console for the database.

One Volt license covers all of them. A metering agent in the same release reports usage to AWS
Marketplace, and the charges appear on your AWS bill.

This page describes the whole installation. After you subscribe, the product's launch page in AWS
Marketplace shows the same commands with your product version filled in.

## Prerequisites

- **A subscription** to the product in AWS Marketplace, in the AWS account that owns the cluster.
  The subscription gives that account access to the product's chart and images.
- **A Volt license file.** Request it from `cloudsales@voltactivedata.com` and quote your AWS
  Marketplace subscription. One file licenses VoltSP and VoltDB; neither starts without it.
- **An Amazon EKS cluster** with:
  - an IAM OIDC provider, which the metering agent's IAM role needs (step 2);
  - a default StorageClass that provisions EBS volumes, for example through the Amazon EBS CSI
    driver add-on. VoltDB keeps its data on a persistent volume. `kubectl get storageclass` must
    show one class marked `(default)`.
- **Tools:** the AWS CLI, `kubectl` pointed at the cluster, `eksctl`, and Helm 3.8 or later. Earlier
  Helm versions cannot pull from an OCI registry.

Point `kubectl` at the cluster:

```bash
aws eks update-kubeconfig \
  --region REGION \
  --name CLUSTER
```

`REGION` and `CLUSTER` are your cluster's region and name. They are used in the commands below.

## 1. Create the namespace

The release, the metering agent's service account and all pods go into one namespace. This page
uses `volt`, the namespace the launch page uses.

```bash
kubectl create namespace volt
```

## 2. Create the metering agent's service account

The metering agent reports usage with an IAM role for a Kubernetes service account (IRSA). AWS
Marketplace accepts usage on EKS only from such a role. The chart does not create the account; you
create it once per cluster and namespace.

If the cluster has no IAM OIDC provider yet:

```bash
eksctl utils associate-iam-oidc-provider \
  --region REGION \
  --cluster CLUSTER \
  --approve
```

Then create the service account `metering-agent` with the AWS managed policy
`AWSMarketplaceMeteringRegisterUsage`:

```bash
eksctl create iamserviceaccount \
  --region REGION \
  --cluster CLUSTER \
  --namespace volt \
  --name metering-agent \
  --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage \
  --approve
```

The name must be `metering-agent`, and the account must be in the release's namespace.

## 3. Log in to the product's registry

The chart and the images are in the AWS Marketplace registry in `us-east-1`. Log Helm in with the
credentials of the subscribed AWS account:

```bash
aws ecr get-login-password \
  --region us-east-1 \
  | helm registry login \
  --username AWS \
  --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com
```

The cluster pulls the images itself; you do not copy images or create a pull secret. A cluster in
another region pulls them from `us-east-1` too.

## 4. Install

```bash
helm install volt \
  oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/volt-active-data-v4sd-test/volt-active-decision-platform \
  --version PRODUCT_VERSION \
  --namespace volt \
  --set-file volt-streams.streaming.licenseXMLFile=volt-license.xml \
  --values my-values.yaml
```

- `PRODUCT_VERSION` is the version shown on the launch page, for example `1.1.42`.
- `volt-license.xml` is the license file Volt issued for your subscription. Use `--set-file`, which
  reads the file as it is. The same file licenses VoltDB; the chart copies it into the Secret the
  database reads.
- `my-values.yaml` is optional on the first install. Without it you get a demonstration pipeline,
  which writes one generated event per second into the `events` table of a demonstration schema,
  and a one-node VoltDB cluster. See **Configuration**.

Before Helm creates anything, a short check runs: a Job (`metering-agent-install-gate`) sends one
usage record of 0 to AWS Marketplace under the `metering-agent` service account. It bills nothing.
If the subscription or the service account is missing, the Job fails and Helm stops the install
with an error about a failed pre-install hook. Read the cause in the Job's log:

```bash
kubectl -n volt logs job/metering-agent-install-gate
```

Fix the cause (step 2, or the subscription). Then remove the failed release and run the same
`helm install` again:

```bash
helm uninstall volt --namespace volt
```

Without `--set-file` the install succeeds, but VoltSP and VoltDB do not start: the chart's default
license is a placeholder. Helm's notes after the install say so. Add the license with
`helm upgrade`, see **Licensing**.

The release name (`volt` above) names the Helm release only. The Kubernetes objects inside it have
fixed names (`volt-streams`, `metering-agent`, `voltdb-cluster`, `voltdb-vmc`, `voltdb-operator`),
because VoltSP reaches the database at a fixed address. So one namespace holds one release.

## 5. Check the deployment

```bash
kubectl -n volt get pods
```

Five pods reach `Running` and `Ready`: `volt-streams-…`, `metering-agent-…`, `voltdb-operator-…`,
`voltdb-cluster-0` and `voltdb-vmc-…`. The database takes one to two minutes on a first start.

VoltSP and the database start at the same time. VoltSP retries its first connection to the database
for up to 10 minutes, so the pipeline connects as soon as the database listens. Until then its log
shows "No connections to cluster".

```bash
kubectl -n volt logs deployment/volt-streams
```

VoltSP logs the license it loaded, with its remaining time, and then the pipeline it started.
VoltSP stays `NotReady` until it has a valid license, so a VoltSP pod that never becomes ready
usually has a license problem, and its log states the cause.

```bash
kubectl -n volt logs deployment/metering-agent
```

The metering agent logs one usage record per hour, with the record ID that AWS returned, and logs
every error it gets from AWS.

The demonstration pipeline writes into the database. Count its rows:

```bash
kubectl -n volt exec voltdb-cluster-0 -- sqlcmd --query='SELECT COUNT(*) FROM events'
```

### Open the Volt Management Center

The VMC is the web console for the database: tables, procedures, live statistics and a query
window. It runs as a Service inside the cluster and is not exposed outside it. Forward its port and
open http://localhost:8080 in a browser:

```bash
kubectl -n volt port-forward svc/voltdb-vmc 8080:8080
```

With VoltDB security off, which is the chart's default, the console needs no login. Stop the
port-forward with Ctrl-C.

## Billing

The product has one price dimension, **vCPU seconds**. The metering agent measures the CPU that each
VoltSP, VoltDB and VMC pod declares (its CPU limit, or its request when no limit is set), for every
second the pod runs, and reports the total to AWS Marketplace once per hour. The VoltDB operator
and the metering agent are not billed. The charges appear on your AWS bill under this product.

So the bill follows the sizes you set: 4 vCPU for VoltSP, 2 vCPU per VoltDB node and half a vCPU for
the VMC by default. A product with 0 pods is not billed, see **Stopping a product**.

Usage is billed while the pods run. Cancelling the subscription does not stop the pods; uninstall
the release to stop them, see **Uninstall**.

## Configuration

Everything in the [VoltSP Helm chart](https://docs.voltactivedata.com/ActiveSP/) is available under
the `volt-streams` key, and everything in the
[VoltDB Helm chart](https://docs.voltactivedata.com/KubernetesAdmin/) under the `voltdb` key. The
settings below are the ones most installations change. If you use an AI coding agent, the
[`voltsp` and `volt-kubernetes` skills](https://github.com/VoltDB/volt-skills) help it with
pipeline, schema and Helm configuration.

```yaml
# my-values.yaml
volt-streams:
  streaming:
    # Provide exactly one of definition or className.
    pipeline:
      definition:
        version: "1"
        name: my-pipeline
        source:
          kafka:
            topics: input-topic
            brokers: kafka:9092
        sink:
          stdout: {}

  # Pod size. Keep the request equal to the limit.
  resources:
    requests:
      cpu: 8
      memory: 16Gi
    limits:
      cpu: 8
      memory: 16Gi

  replicaCount: 3

  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 75

voltdb:
  cluster:
    config:
      deployment:
        cluster:
          # Raise kfactor and the node count together; kfactor must stay below the node count.
          kfactor: 1
          sitesperhost: 4
      # Your schema, as SQL. Procedures written in Java go under classes as jars.
      schemas:
        schema.sql: |
          CREATE TABLE orders (id BIGINT NOT NULL, amount DECIMAL, PRIMARY KEY (id));
          PARTITION TABLE orders ON COLUMN id;
    clusterSpec:
      replicas: 3
      resources:
        requests:
          cpu: 4
          memory: 8Gi
        limits:
          cpu: 4
          memory: 8Gi
```

### VoltDB

The chart installs a one-node cluster: `voltdb.cluster.clusterSpec.replicas: 1`, `kfactor: 0`, two
sites per host, 2 vCPU and 4Gi reserved and limited, a 10Gi volume, and command log and automatic
snapshots off. That is a demonstration database, not a durable one. For a real cluster, raise the
node count and `kfactor` together, size the nodes, and turn on the durability features you need in
`voltdb.cluster.config.deployment`. The volume size cannot change after the first install.

Your pipeline reaches the database at `voltdb-cluster-client:21212` inside the namespace. The
`voltdb-client` resource in the chart's default pipeline shows the format.

The VMC is on by default and is billed like the other pods, at half a vCPU; size it with
`voltdb.vmc.resources`.

### Stopping a product

The pod count stops a product. `voltdb.cluster.clusterSpec.replicas: 0` stops the database and its
billing, and keeps its configuration and its volume. `volt-streams.replicaCount: 0` does the same for
VoltSP. Set the count back and run `helm upgrade` to start again.

### Pod size

The chart reserves 4 vCPU and 5Gi for VoltSP and sets the same values as limits. Equal requests and
limits give the pod Guaranteed quality of service: VoltSP is not throttled below what it reserved,
and it is evicted only after pods that reserved less. Change both together. You are billed for the
limit.

### Replicas, workers and autoscaling

`replicaCount` is 1 and autoscaling is off by default. Enabling autoscaling creates a
HorizontalPodAutoscaler, which then controls the replica count.

`volt-streams.streaming.parallelism` is the number of worker threads inside one pod, 1 by default.
Raise it to use more of the pod's CPU before you add pods.

### Application artifacts

A pipeline written in Java needs its jar on the classpath. Use `volt-streams.streaming.voltapps`
for a single jar, or `volt-streams.streaming.appVolumes` to mount a volume under `/volt-apps`, for
example an Amazon S3 bucket through the Mountpoint for Amazon S3 CSI driver.

### Licensing

`volt-streams.streaming.licenseXMLFile` carries the license for VoltSP and VoltDB. The license has
an expiry date, and nothing renews it. VoltSP and VoltDB stop when it passes. Request a replacement
from `cloudsales@voltactivedata.com` and apply it:

```bash
helm upgrade volt \
  oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/volt-active-data-v4sd-test/volt-active-decision-platform \
  --version PRODUCT_VERSION \
  --namespace volt \
  --reuse-values \
  --set-file volt-streams.streaming.licenseXMLFile=new-volt-license.xml
```

`volt-streams.streaming.licenseServer` (a license served at run time) is not available yet. Leave
it empty.

### Values to leave alone

| Value | Why |
|---|---|
| `metering-agent.*` | The product code, the price dimension, the billed pods and the service account. Changing them stops or breaks usage reporting. |
| `volt-streams.podLabels`, `voltdb.commonLabels` | The product label. VoltSP copies its pod labels into its Deployment selector, which Kubernetes does not allow to change, so a change needs a reinstall. |
| `voltdb.fullnameOverride`, `volt-streams.fullnameOverride` | The fixed object names that the pipeline and the metering agent use. |
| `voltdb.cluster.clusterSpec.disableFinalizers`, `voltdb.cluster.clusterSpec.deletePVC` | With finalizers on, the operator must run when the `VoltDBCluster` is deleted. An uninstall removes both at once and can leave the cluster half-deleted. See **Uninstall**. |

- **Value**: the Helm value path.
- **Why**: what depends on the value.

## Upgrade

```bash
helm upgrade volt \
  oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/volt-active-data-v4sd-test/volt-active-decision-platform \
  --version NEW_PRODUCT_VERSION \
  --namespace volt \
  --reuse-values
```

A new chart version also moves the images to that version. Log in to the registry first (step 3);
the login expires after 12 hours. The install check from step 4 runs again before the upgrade, so
an upgrade fails while the subscription is cancelled.

## Uninstall

```bash
helm uninstall volt --namespace volt
```

This removes VoltSP, the metering agent, the VoltDB operator, the VMC and the `VoltDBCluster` object.
The database's StatefulSet, its pod and its Services are deleted with the `VoltDBCluster`, because
the operator marks them as owned by it. Usage reporting stops when the pods stop. The subscription
itself is cancelled in AWS Marketplace, not here.

Three things stay behind:

- **The VoltDB volume, with your data.** A StatefulSet does not delete the volume claims it
  created, and the chart turns the operator's finalizer off (see **Values to leave alone**). To
  delete the volume and its data, after the database pod is gone:
  ```bash
  kubectl -n volt delete pvc \
    --selector=voltdb-cluster-name=voltdb-cluster
  ```
  Whether the EBS volume is deleted with the claim depends on the StorageClass's reclaim policy.
  Take a snapshot first if you want to keep the data.
- **The metering agent's state**, the ConfigMap `metering-agent-state`. A reinstall continues from
  it. To delete it:
  ```bash
  kubectl -n volt delete configmap metering-agent-state
  ```
- **The service account and its IAM role** from step 2. A reinstall into the same namespace uses
  them again. To delete them:
  ```bash
  eksctl delete iamserviceaccount \
    --region REGION \
    --cluster CLUSTER \
    --namespace volt \
    --name metering-agent
  ```

Deleting the namespace deletes the volume claim and the ConfigMap too.

## Rules and limits

- **One release per namespace.** The Kubernetes object names are fixed, so a second release in the
  same namespace collides with the first.
- **The service account `metering-agent` must be in the release's namespace**, with the IAM role
  from step 2.
- **The metering agent must reach the AWS Marketplace Metering Service** in the cluster's region,
  directly or through a proxy. Without it no usage is reported, and the install check fails.

## Support

- Licenses and billing questions: `cloudsales@voltactivedata.com`
- VoltSP documentation: https://docs.voltactivedata.com/ActiveSP/
- VoltDB on Kubernetes: https://docs.voltactivedata.com/KubernetesAdmin/
- Volt Active Data: https://www.voltactivedata.com
