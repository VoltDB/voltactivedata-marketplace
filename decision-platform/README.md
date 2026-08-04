# Volt Active Decision Platform — Google Cloud Marketplace

VoltSP (real-time stream processing) plus the Volt Metering Agent, packaged as a Google Cloud
Marketplace **Kubernetes application** for deployment on GKE.

This directory documents how to deploy the app from the **command line** (in addition to the Cloud
Console one-click flow).

## Prerequisites

- A **GKE cluster** and `kubectl` configured to point at it:
  ```bash
  gcloud container clusters get-credentials CLUSTER --zone ZONE --project PROJECT
  ```
- **`gcloud`** authenticated (`gcloud auth login`) with access to the project.
- **`mpdev`** — the Marketplace CLI from
  [marketplace-k8s-app-tools](https://github.com/GoogleCloudPlatform/marketplace-k8s-app-tools):
  ```bash
  docker run gcr.io/cloud-marketplace-tools/k8s/dev cat /scripts/dev > ~/bin/mpdev
  chmod +x ~/bin/mpdev
  ```
- The **Application CRD** installed on the cluster (once per cluster):
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/marketplace-k8s-app-tools/master/crd/app-crd.yaml
  ```
- A **reporting secret** for usage-based billing — Google Cloud Marketplace provisions this in your
  cluster when you deploy a purchased entitlement (keys `consumer-id`, `entitlement-id`,
  `reporting-key`). Its name is passed as the `reporting secret` parameter below.

## Deploy (command line)

```bash
mpdev install \
  --deployer=gcr.io/voltactivedata-public/decision-platform/deployer:1.0 \
  --parameters='{
    "name": "decision-platform-1",
    "namespace": "decision-platform",
    "metering-agent.reporting.secretName": "REPORTING_SECRET_NAME",
    "volt-streams.streaming.pipeline.definition": "version: \"1\"\nname: my-pipeline\nsource:\n  kafka:\n    ...\nsink:\n  ..."
  }'
```

Create the namespace first if it doesn't exist (`kubectl create namespace decision-platform`).

> The `--deployer` path above is illustrative; use the deployer image published with the offer
> version you are deploying (shown on the product's Marketplace listing).

## Parameters

| Parameter | Required | Description |
|---|---|---|
| `name` | yes | Application instance name. |
| `namespace` | yes | Target namespace. |
| `metering-agent.reporting.secretName` | yes | Name of the Marketplace-provisioned usage-reporting Secret. |
| `volt-streams.streaming.pipeline.definition` | yes | The VoltSP pipeline to run, in YAML. See the [VoltSP docs](https://docs.voltactivedata.com/ActiveSP/). |
| `volt-streams.autoscaling.enabled` | no (default `true`) | Horizontally autoscale VoltSP. |
| `volt-streams.autoscaling.maxReplicas` | no (default `5`) | Upper bound for autoscaling. |

Licensing is handled automatically at runtime by the Metering Agent — no license file is required.

## Verify the deployment

```bash
kubectl -n decision-platform get pods
kubectl -n decision-platform wait --for=condition=Available deployment --all --timeout=600s
```

VoltSP runs the configured pipeline; the Metering Agent serves its license and reports usage.

## Uninstall

```bash
kubectl delete application decision-platform-1 -n decision-platform
# or: mpdev delete, or delete the namespace
```

## Support

- VoltSP documentation: https://docs.voltactivedata.com/ActiveSP/
- Volt Active Data: https://www.voltactivedata.com
