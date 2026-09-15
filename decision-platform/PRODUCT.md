# Volt Active Decision Platform

VoltSP, Volt's real-time stream processing engine driven by configured pipeline - See helm installation guide.


## What gets deployed

| Component | What it does                                                                            |
|---|-----------------------------------------------------------------------------------------|
| **VoltSP** | Runs your pipeline. One pod by default, sized from the deploy form.                     |
| **Metering Agent** | Is a GCP specific agent, reporting usage to Google Cloud so the subscription is billed. |
| **Usage-based billing agent** | Sidecar that delivers those reports to Google's Service Control API.                    |

A click-to-deploy install runs a demonstration pipeline that prints a message and processes one
event per day, so that a new instance comes up licensed and healthy without any configuration. Your
own pipeline is deployed with Helm — see [HELM.md](./HELM.md).

## First steps after deploying

1. Point `kubectl` at the cluster you deployed into. Take the cluster name and location from the
   instance's page in the Cloud Console:
   ```bash
   gcloud container clusters get-credentials CLUSTER \
     --location LOCATION \
     --project PROJECT
   ```
2. Check that both pods are running: `kubectl -n NAMESPACE get pods`.
3. Read the VoltSP log to confirm the license loaded and the pipeline started:
   `kubectl -n NAMESPACE logs deployment/volt-streams`.
4. Replace the demonstration pipeline with your own, using [HELM.md](./HELM.md).

`NAMESPACE` is the namespace you chose on the deploy form.

## Licensing

The product needs a license issued by Volt for your purchase. Request it from
`sales@voltactivedata.com`, quoting the order or entitlement id, and paste it into the deploy form's
license field. Licenses carry an expiry date; request a replacement before it passes.

## Billing

Usage is metered in your cluster and reported to Google Cloud. The charges appear on your Google
Cloud bill under this product's subscription. Deleting the deployment stops usage reporting; it does
not cancel the subscription, which is canceled in the Cloud Marketplace console.

## Documentation and support

- Command-line deployment: [README.md](./README.md)
- Deploying with Helm and configuring pipelines: [HELM.md](./HELM.md)
- VoltSP product documentation: https://docs.voltactivedata.com/ActiveSP/
- Volt Active Data: https://www.voltactivedata.com
