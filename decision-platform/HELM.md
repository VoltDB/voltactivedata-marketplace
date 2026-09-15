# Deploy with Helm

The Volt Active Decision Platform is published from one build in two forms: as a Google Cloud
Marketplace Kubernetes application (the **Deploy** button in the Cloud Console) and as the same
Helm chart in a public OCI repository. This page describes the Helm path.

Billing is the same on both paths. Usage is reported through the reporting secret that Google Cloud
Marketplace creates for your entitlement, and the chart reads that secret however it was installed.

## Which path to use

| | Click to deploy | Helm |
|---|---|---|
| What you can configure | The deploy form: instance name, namespace, license, CPU, memory | Every value in the chart |
| Pipeline | A demonstration pipeline that prints a message once a day | Your pipeline |
| Pods | One VoltSP pod, no autoscaling | Replica count and autoscaling as you set them |
| Reconfiguration | Delete the instance and deploy again | `helm upgrade` |
| Helm release | None. The Marketplace deployer renders the chart and applies the result | A normal Helm release |

The deploy form is deliberately small. Use Helm for anything beyond a first look.

## Prerequisites

- A purchased entitlement for the product on Google Cloud Marketplace.
- A **VoltSP license file**. Request it from `sales@voltactivedata.com`, quoting the order or
  entitlement id for your purchase. VoltSP does not start without a license.
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
  the file as it is — no reformatting, no escaping. Without a license the install stops with
  "License has not been provided".
- `my-values.yaml` is optional on the first install. Without it you get the demonstration pipeline,
  which prints a message and processes one event per day. See **Configuration** below.

The release name (`voltdp` above) is yours to choose too, and it names the Helm release only: the
Kubernetes objects inside it have fixed names, `volt-streams` and `metering-agent`, because VoltSP
reaches the Metering Agent at a fixed address. That is why only one release fits in a namespace.

## 4. Check the deployment

```bash
kubectl -n voltdp get pods
```

```bash
kubectl -n voltdp logs deployment/metering-agent
```

```bash
kubectl -n voltdp logs deployment/volt-streams
```

The deployment also appears under **Kubernetes Engine → Applications**, where its side panel links
back to this documentation.

Both pods reach `Running` and `Ready`. VoltSP logs the license it loaded, including how long it
has left, and then the pipeline it started. VoltSP stays `NotReady` until it holds a valid license,
so a VoltSP pod that never becomes ready is usually a licensing problem, and its own log states the
cause — an expired license, or an XML that lost characters on the way in.

## Configuration

Everything in the [VoltSP Helm chart](https://docs.voltactivedata.com/ActiveSP/) is available under
the `volt-streams` key. The settings below are the ones most installations change. If you use an AI
coding agent, the [`voltsp` and `volt-kubernetes` skills](https://github.com/VoltDB/volt-skills) walk
it through pipeline and Helm configuration — see the [README](./README.md).

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
```

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

VoltSP needs exactly one of two things, and prefers the first:

- `volt-streams.streaming.licenseXMLFile` — the license Volt issued for your purchase. This is what
  both install paths use. Supply it with `--set-file`, as above.
- `volt-streams.streaming.licenseServer` — runtime licensing, where VoltSP asks the in-cluster
  Metering Agent, which asks Volt's licensing service for a short-lived license. Switched off.

Setting both is safe: VoltSP reads the file and only calls the server if the file is missing or
unreadable.

The license carries an expiry date, and nothing renews it. VoltSP stops when it passes. Request a
replacement from `sales@voltactivedata.com` and apply it without downtime:

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
| `volt-streams.podLabels`, `metering-agent.podLabels` | Carry the `goog-partner-solution` label that Google requires on every pod of the product. |

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

This removes the release. It does not remove the reporting secret, which Marketplace created and
which you need again if you reinstall into the same namespace. Usage reporting stops when the pods
stop; the entitlement itself is cancelled in the Marketplace console, not here.

## Rules and limits

- **One release per namespace.** The Kubernetes object names are fixed, so a second release in the
  same namespace collides with the first.
- **Do not install into a namespace that holds a click-to-deploy instance.** To move an existing
  instance to Helm, delete the Application object first, then install with Helm into that
  namespace and reuse the reporting secret that is already there.
- **The reporting secret must be in the release's namespace.** A Secret in another namespace is not
  visible to the pods.

## Support

- VoltSP documentation: https://docs.voltactivedata.com/ActiveSP/
- Volt Active Data: https://www.voltactivedata.com
