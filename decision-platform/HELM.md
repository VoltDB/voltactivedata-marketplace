# Deploy with Helm

The Volt Active Decision Platform is published from one build in two forms: as a Google Cloud
Marketplace Kubernetes application (the **Deploy** button in the Cloud Console) and as the same
Helm chart in a public OCI repository. This page describes the Helm path.

The product is three things in one chart: 
**VoltSP**, the stream processing engine that runs your pipeline; 
**VoltDB**, the database, run by its operator; and the 
**Volt Management Center (VMC)**, the web console for the database. One license covers all of them.

Billing is the same on both paths. Usage is reported through the reporting secret that Google Cloud
Marketplace creates for your entitlement, and the chart reads that secret however it was installed.

## Which path to use

| | Click to deploy | Helm |
|---|---|---|
| What you can configure | The deploy form: instance name, namespace, license, CPU and memory of the VoltSP pod and of the VoltDB node | Every value in the chart |
| Pipeline and schema | A demonstration pipeline that writes one event per second into a one-table demonstration schema | Your pipeline, your schema |
| Pods | One VoltSP pod, a one-node VoltDB cluster, the VMC; no autoscaling | Replica counts, node count and autoscaling as you set them |
| Reconfiguration | `helm upgrade` from this chart, see **Upgrade** | `helm upgrade` |
| Helm release | Yes. The Marketplace deployer installs the chart as a Helm release named after the instance | A normal Helm release |

The deploy form is deliberately small. Use Helm for anything beyond a first look.

## Prerequisites

- A purchased entitlement for the product on Google Cloud Marketplace.
- A **Volt license file**. Request it from `sales@voltactivedata.com`, quoting the order or
  entitlement id for your purchase. One file licenses VoltSP and VoltDB; neither starts without it.
- A **GKE cluster in a project linked to the billing account you bought the product with.** That
  link is what gives the cluster access to the product's images: they live in Google's Marketplace
  registry at `gcr.io/cloud-marketplace/voltactivedata-public/volt-active-decision-platform`, and
  the chart points there, so your cluster pulls them directly — you do not copy images or create a
  pull secret. A cluster in a project billed to a different account cannot pull them, and its pods
  stay in `ImagePullBackOff`. If your nodes reach the internet through a proxy, allow
  `marketplace.gcr.io`.

  Point `kubectl` at that cluster:
  ```bash
  gcloud container clusters get-credentials CLUSTER \
      --location LOCATION \
      --project PROJECT
  ```
- Helm 3.8 or later. Earlier versions cannot pull from an OCI repository.
- The **Application CRD** on the cluster, installed once per cluster:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml
  ```
  The chart creates an `app.k8s.io` Application resource, which is what lists your instance under
  **Kubernetes Engine → Applications** in the Cloud Console with links to this documentation. Without
  the CRD the install fails before anything is created, reporting that there is no match for kind
  Application in `app.k8s.io/v1beta1`. Installing a CRD is a cluster-wide action; if you cannot do it,
  add `--set application.create=false` to the install command instead. Everything except the
  Applications entry works the same way.

The chart repository itself is public. Pulling the chart needs no authentication.

## 1. Create the namespace

Everything below goes into one namespace: the reporting secret, the release, and the pods.

```bash
kubectl create namespace voltdp
```

`voltdp` is an example — use any name you like. This page uses it throughout, so substitute yours
in the commands that follow.

## 2. Get the reporting secret

The reporting secret is the only per-customer input. Google Cloud Marketplace creates it; we never
issue one. It must live in the namespace you just created, because a Secret is only visible to pods
in its own namespace.

1. Open the product in the Cloud Console and choose **Deploy via command line** on the purchased
   entitlement.
2. Run the command the console shows (`kubectl apply -f service-account-key.yaml`), targeting the
   namespace from step 1 — add `-n voltdp` if the manifest does not name one. It creates a Secret
   with three keys: `consumer-id`, `entitlement-id`, and `reporting-key`.
3. Note the Secret's name. You pass it to Helm in the next step.

If you already deployed this product from the console, its namespace already holds the Secret. Use
that namespace instead and skip step 1.

Check what you have:

```bash
kubectl -n voltdp get secret REPORTING_SECRET_NAME -o jsonpath='{.data}' | tr ',' '\n'
```

All three keys must be present. The Metering Agent uses `entitlement-id` and `consumer-id` to
request the license and `reporting-key` both to authenticate that request and to report usage.

## 3. Install

```bash
helm install voltdp \
  oci://us-docker.pkg.dev/voltactivedata-public/charts/vsp-marketplace \
  --version PRODUCT_VERSION \
  --namespace voltdp \
  --set metering-agent.reporting.secretName=REPORTING_SECRET_NAME \
  --set-file volt-streams.streaming.licenseXMLFile=license.xml \
  --values my-values.yaml
```

- `PRODUCT_VERSION` is the product version shown on the Marketplace listing, for example `1.0.0`.
  The chart version and the product version are the same number, and the chart's image defaults
  point at that version of the images.
- `REPORTING_SECRET_NAME` is the Secret from step 2, in the same namespace as the release.
- `license.xml` is the license file Volt issued for your purchase. Use `--set-file`, which reads
  the file as it is — no reformatting, no escaping. The same file licenses VoltDB; the chart copies
  it into the Secret the database reads. Without a license the install stops with
  "License has not been provided".
- `my-values.yaml` is optional on the first install. Without it you get the demonstration pipeline,
  which writes one generated event per second into the `events` table of the demonstration schema,
  and a one-node VoltDB cluster. See **Configuration** below.

The release name (`voltdp` above) is yours to choose too, and it names the Helm release only: the
Kubernetes objects inside it have fixed names (`volt-streams`, `metering-agent`, `voltdb-cluster`,
`voltdb-vmc`, `voltdb-operator`), because VoltSP reaches the Metering Agent and the database at
fixed addresses. That is why only one release fits in a namespace.

## 4. Check the deployment

```bash
kubectl -n voltdp get pods
```

Five pods reach `Running` and `Ready`: `volt-streams-…`, `metering-agent-…`, `voltdb-operator-…`,
`voltdb-cluster-0` and `voltdb-vmc-…`. The database takes one to two minutes on a first start.

**Replace the VoltSP pod once after a fresh install.** VoltSP 1.8.5 connects to the database when
its pipeline starts and does not retry a first connection that failed, so on a fresh install, where
both start at the same time, VoltSP keeps logging "No connections to cluster" even after the
database is up. When `voltdb-cluster-0` is `Ready`, run:

```bash
kubectl -n voltdp delete pod \
  --selector=app.kubernetes.io/name=volt-streams
```

The Deployment creates a new pod at once, and it connects immediately; the client reconnects on its
own from then on. Delete the pod rather than `kubectl rollout restart`: a rolling restart starts the
new pod beside the old one and needs a second VoltSP worth of CPU meanwhile, so on a cluster without
that spare capacity the new pod stays `Pending`. The Marketplace deployer performs this step for
you; a `helm upgrade` that only changes VoltSP does not need it, because the database is already
running.

```bash
kubectl -n voltdp logs deployment/volt-streams
```

VoltSP logs the license it loaded, including how long it has left, and then the pipeline it
started. VoltSP stays `NotReady` until it holds a valid license, so a VoltSP pod that never becomes
ready is usually a licensing problem, and its own log states the cause — an expired license, or an
XML that lost characters on the way in.

```bash
kubectl -n voltdp logs deployment/metering-agent
```

The demonstration pipeline writes into the database. Count its rows:

```bash
kubectl -n voltdp exec voltdb-cluster-0 -- sqlcmd --query='SELECT COUNT(*) FROM events'
```

The deployment also appears under **Kubernetes Engine → Applications**, where its side panel links
back to this documentation.

### Open the Volt Management Center

The VMC is the web console for the database cluster: tables, procedures, live statistics, the
`sqlcmd`-style query window. It runs as a Service inside the cluster and is not exposed outside it.
Forward its port and open http://localhost:8080 in a browser:

```bash
kubectl -n voltdp port-forward svc/voltdb-vmc 8080:8080
```

With VoltDB security off, the chart's default, the console needs no login. Stop the port-forward
with Ctrl-C; nothing else changes.

## Configuration

Everything in the [VoltSP Helm chart](https://docs.voltactivedata.com/ActiveSP/) is available under
the `volt-streams` key, and everything in the
[VoltDB Helm chart](https://docs.voltactivedata.com/KubernetesAdmin/) under the `voltdb` key. The
settings below are the ones most installations change. If you use an AI coding agent, the
[`voltsp` and `volt-kubernetes` skills](https://github.com/VoltDB/volt-skills) walk it through
pipeline, schema and Helm configuration — see the [README](./README.md).

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

  # Pod size. Keep the request equal to the limit unless you want the pod to burst.
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
sites per host, 2 CPU and 4Gi reserved and limited, a 10Gi volume, command log and automatic
snapshots off. That is a demonstration database, not a durable one. For a real cluster raise the
node count and `kfactor` together, size the nodes, and turn on the durability features you need in
`voltdb.cluster.config.deployment`. The volume size cannot change after the first install.

Your pipeline reaches the database at `voltdb-cluster-client:21212` inside the namespace, the
address the demonstration pipeline uses; see the `voltdb-client` resource in the chart's default
pipeline for the shape.

The VMC is on by default and is billed like the other pods, at half a core; size it with
`voltdb.vmc.resources`. It is only reachable by port-forwarding, see **Open the Volt Management
Center** above.

### Stopping a product

The node count is the switch. `voltdb.cluster.clusterSpec.replicas: 0` stops the database and the
bill for it while keeping its configuration and its volume; `volt-streams.replicaCount: 0` does the
same for VoltSP. With both at 0, the only pods left are the VoltDB operator, the Metering Agent and
the VMC. Set the count back and `helm upgrade` to start again.

### Pod size

The chart reserves 4 CPU cores and 5Gi, and sets the same values as limits. Equal requests and
limits give the pod Guaranteed quality of service: VoltSP is not throttled below what it reserved,
and it is evicted only after pods that reserved less. Change both together. A request above its
limit is rejected when the pod is created, and a limit well above the request lets VoltSP burst
into memory the node has not reserved for it, which ends in an eviction under node pressure.

### Replicas, workers and autoscaling

`replicaCount` is 1 and autoscaling is off by default. Enabling autoscaling creates a
HorizontalPodAutoscaler that then controls the replica count.

`volt-streams.streaming.parallelism` is the number of worker threads inside one pod, 1 by default.
Raise it to use more of the pod's CPU before adding pods.

### Application artifacts

A pipeline written in Java needs its jar on the classpath. Use `volt-streams.streaming.voltapps`
for a single jar, or `volt-streams.streaming.appVolumes` to mount a volume (for example a Cloud
Storage bucket through the GCS FUSE CSI driver) under `/volt-apps`.

### Licensing

One Volt license covers VoltSP and VoltDB, and one value carries it. VoltSP needs exactly one of
two things, and prefers the first:

- `volt-streams.streaming.licenseXMLFile` — the license Volt issued for your purchase. This is what
  both install paths use, and what the database reads too: the chart copies it into the
  `voltdb-license` Secret. Supply it with `--set-file`, as above.
- `volt-streams.streaming.licenseServer` — runtime licensing, where VoltSP asks the in-cluster
  Metering Agent, which asks Volt's licensing service for a short-lived license. Switched off.
  VoltDB has no such path: a database with nodes needs the file, so runtime licensing is for a
  VoltSP-only install (`voltdb.cluster.clusterSpec.replicas: 0`).

Setting both is safe: VoltSP reads the file and only calls the server if the file is missing or
unreadable.

The license carries an expiry date, and nothing renews it. VoltSP and VoltDB stop when it passes.
Request a replacement from `sales@voltactivedata.com` and apply it without downtime:

```bash
helm upgrade voltdp \
  oci://us-docker.pkg.dev/voltactivedata-public/charts/vsp-marketplace \
  --version PRODUCT_VERSION \
  --namespace voltdp \
  --reuse-values \
  --set-file volt-streams.streaming.licenseXMLFile=new-license.xml
```

To use runtime licensing instead, ask Volt whether your purchase is set up for it, then:

```yaml
volt-streams:
  streaming:
    licenseXMLFile: ""
    licenseServer: metering-agent:8443
```

The certificate authority, timeout and retry values that path needs are already in the chart.

### Values to leave alone

| Value | Why |
|---|---|
| `metering-agent.controlPlaneUrl` | Volt's licensing service, used only when runtime licensing is on. |
| `metering-agent.ubbagent.*`, `metering-agent.usage.*` | The billing dimension and the metered service name for this product. |
| `metering-agent.enabled` | Turning it off stops usage reporting, which is how the product is billed. |
| `volt-streams.podLabels`, `metering-agent.podLabels`, `voltdb.commonLabels` | Carry the `goog-partner-solution` label that Google requires on every pod of the product. |
| `voltdb.fullnameOverride`, `volt-streams.fullnameOverride` | The fixed object names the pipeline and the Metering Agent rely on. |
| `voltdb.cluster.clusterSpec.disableFinalizers`, `voltdb.cluster.clusterSpec.deletePVC` | With finalizers on, the operator must be running when the `VoltDBCluster` is deleted; an uninstall removes both at once and can leave the cluster half-deleted for good. See **Uninstall**. |

## Upgrade

```bash
helm upgrade voltdp \
  oci://us-docker.pkg.dev/voltactivedata-public/charts/vsp-marketplace \
  --version NEW_PRODUCT_VERSION \
  --namespace voltdp \
  --set metering-agent.reporting.secretName=REPORTING_SECRET_NAME \
  --values my-values.yaml
```

Pass the same values file and the same secret name. A chart version upgrade also moves the images
to that version.

**An instance deployed from the Cloud Console is a Helm release too**, named after the instance and
living in the namespace you chose on the form. Upgrade or reconfigure it with the command above,
using the instance name as the release name; `helm get values RELEASE -n NAMESPACE` shows what the
form set. Deleting the instance in the Cloud Console removes the release as installed; revisions
your own `helm upgrade` created afterwards are removed by `helm uninstall`.

**Upgrading from a release installed before chart version 1.0.0** fails with an error about the
Deployment selector being immutable. That version added a required Google label to the VoltSP pods,
and the chart it is built on copies pod labels into the Deployment's selector, which Kubernetes does
not allow to change. Uninstall and install again:

```bash
helm uninstall voltdp --namespace voltdp
```

The reporting secret survives an uninstall, so reinstall with the same command as a first install.
Usage is not reported while nothing is running; nothing else is lost, because the release holds no
state of its own.

## Uninstall

```bash
helm uninstall voltdp --namespace voltdp
```

This removes the release: VoltSP, the Metering Agent, the VoltDB operator, the VMC and the
`VoltDBCluster` object. The database's StatefulSet, its pod and its Services go with the
`VoltDBCluster`, because the operator marks them as owned by it and Kubernetes deletes owned objects
together with their owner, whether or not the operator is still running. Usage reporting stops when
the pods stop; the entitlement itself is cancelled in the Marketplace console, not here.

Two things stay behind on purpose:

- **The reporting secret.** Marketplace created it, and you need it again to reinstall into the same
  namespace.
- **The VoltDB volume, with your data.** The persistent volume claim is not in the chain of owned
  objects, for two reasons. A StatefulSet does not own the volume claims it creates, so deleting it
  leaves them. And the operator can delete the claims itself (`deletePVC: true`), but only through a
  finalizer that runs while the operator is alive; an uninstall deletes the operator and the
  `VoltDBCluster` at the same moment, so that finalizer could leave the `VoltDBCluster` half-deleted
  for good. The chart therefore sets `voltdb.cluster.clusterSpec.disableFinalizers: true`, and the
  volume is yours to delete. A click-to-deploy instance deleted from the Cloud Console behaves the
  same way: everything above goes, the volume stays.

To delete the volume and its data, once the database pod is gone:

```bash
kubectl -n voltdp delete pvc \
  --selector=voltdb-cluster-name=voltdb-cluster
```

Deleting the namespace deletes it too. The disk behind the claim is deleted with it (GKE's default
storage class reclaims on delete), so take a snapshot first if you want to keep the data.

## Rules and limits

- **One release per namespace.** The Kubernetes object names are fixed, so a second release in the
  same namespace collides with the first.
- **An instance deployed from the Cloud Console already is the release of its namespace.** Do not
  install a second one there; reconfigure the existing one with `helm upgrade`, see **Upgrade**.
- **The reporting secret must be in the release's namespace.** A Secret in another namespace is not
  visible to the pods.

## Support

- VoltSP documentation: https://docs.voltactivedata.com/ActiveSP/
- VoltDB on Kubernetes: https://docs.voltactivedata.com/KubernetesAdmin/
- Volt Active Data: https://www.voltactivedata.com
