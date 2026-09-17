# Volt Active Decision Platform

Volt's stream processing engine and its database in one deployment: **VoltSP** runs your pipeline,
**VoltDB** stores and serves what the pipeline decides, and the **Volt Management Center (VMC)**
shows you the database as it runs. One license covers all of them; one subscription bills them.

## What gets deployed

| Component | What it does |
|---|---|
| **VoltSP** | Runs your pipeline. One pod by default, sized from the deploy form. |
| **VoltDB** | The database, as a one-node cluster by default, sized from the deploy form. Its operator runs next to it and manages it. |
| **Volt Management Center** | The web console for the database: tables, procedures, live statistics, a query window. Reached by port-forwarding, see below. |
| **Metering Agent** | Reports the declared CPU of the VoltSP, VoltDB and VMC pods to Google Cloud so the subscription is billed. |
| **Usage-based billing agent** | Sidecar that delivers those reports to Google's Service Control API. |

A click-to-deploy install runs a demonstration pipeline that writes one generated event per second
into the `events` table of a one-table demonstration schema, so that a new instance comes up
licensed, healthy and visibly working without any configuration. Your own pipeline and schema are
deployed with Helm — see
[HELM.md](https://github.com/VoltDB/voltactivedata-marketplace/blob/main/decision-platform/HELM.md).
The instance is a Helm release named after it, so `helm upgrade` reconfigures it in place.

## First steps after deploying

1. Point `kubectl` at the cluster you deployed into. Take the cluster name and location from the
   instance's page in the Cloud Console:
   ```bash
   gcloud container clusters get-credentials CLUSTER \
     --location LOCATION \
     --project PROJECT
   ```
2. Check that the five pods are running: `kubectl -n NAMESPACE get pods`. The database takes one
   to two minutes on a first start; VoltSP waits for it.
3. Read the VoltSP log to confirm the license loaded and the pipeline started:
   `kubectl -n NAMESPACE logs deployment/volt-streams`.
4. Open the Volt Management Center. It is a service inside your cluster, not exposed to the
   internet; forward its port and open http://localhost:8080 in a browser:
   ```bash
   kubectl -n NAMESPACE port-forward svc/voltdb-vmc 8080:8080
   ```
   The `events` table fills at one row per second while the demonstration pipeline runs. Stop the
   port-forward with Ctrl-C.
5. Replace the demonstration pipeline and schema with your own, using
   [HELM.md](https://github.com/VoltDB/voltactivedata-marketplace/blob/main/decision-platform/HELM.md).

`NAMESPACE` is the namespace you chose on the deploy form.

## Licensing

The product needs a license issued by Volt for your purchase. Request it from
`sales@voltactivedata.com`, quoting the order or entitlement id, and paste it into the deploy form's
license field. One license covers VoltSP and VoltDB. Licenses carry an expiry date; request a
replacement before it passes.

## Billing

Usage is metered in your cluster and reported to Google Cloud: the CPU each VoltSP, VoltDB and
VMC pod declares, for as long as it runs, so the bill follows the sizes you chose on the deploy
form. The charges appear on your Google Cloud bill under this product's subscription. Deleting the
deployment stops usage reporting; it does not cancel the subscription, which is canceled in the
Cloud Marketplace console. The database's volume survives deletion, with your data, until you
remove it; [HELM.md](HELM.md#uninstall) explains why and gives the command.

## Documentation and support

- Command-line deployment: [README.md](https://github.com/VoltDB/voltactivedata-marketplace/blob/main/decision-platform/README.md)
- Deploying with Helm, configuring pipelines and schemas: [HELM.md](https://github.com/VoltDB/voltactivedata-marketplace/blob/main/decision-platform/HELM.md)
- VoltSP product documentation: https://docs.voltactivedata.com/ActiveSP/
- VoltDB on Kubernetes: https://docs.voltactivedata.com/KubernetesAdmin/
- Volt Active Data: https://www.voltactivedata.com
