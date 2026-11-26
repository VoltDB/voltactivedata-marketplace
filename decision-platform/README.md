# Volt Active Decision Platform — Google Cloud Marketplace

VoltSP (real-time stream processing), VoltDB (the database, with its operator and the Volt
Management Center) and the Volt Metering Agent, packaged as a Google Cloud Marketplace
**Kubernetes application** for deployment on GKE.

| Document | What it covers |
|---|---|
| [PRODUCT.md](./PRODUCT.md) | What the product is, what gets deployed, first steps, licensing and billing. |
| [HELM.md](./HELM.md) | Deploying with Helm: your own pipeline, sizing, scaling, upgrades. |

## Two ways to deploy

- **Deploy on GKE from the Cloud Console.** The deploy form: instance name, namespace, your license,
  and the CPU and memory of the VoltSP pod and of the VoltDB node. It installs one VoltSP pod
  running a built-in demonstration pipeline, a one-node VoltDB cluster and the Volt Management
  Center. The instance is a Helm release named after it, so you can reconfigure it later with
  `helm upgrade`, see [HELM.md](./HELM.md).
- **[Helm](./HELM.md).** The "Deploy via command line" path uses the same chart, from a public
  repository, with every VoltSP and VoltDB setting available. Use this to run your own pipeline and
  schema, to size or scale the deployment, or to reconfigure it later with `helm upgrade`.

Both paths install the same chart and the same images, and both are billed the same way.

## What you need before deploying

- A **reporting secret** — the Kubernetes Secret that carries your entitlement and the credentials
  the product reports usage with. Google Cloud Marketplace creates it; we never issue one. It holds
  three keys:
  ```yaml
  data:
    entitlement-id: eID     # identifies your purchase
    consumer-id: cID        # identifies who is billed
    reporting-key: rKEY     # the service account key usage is reported with
  ```
  The service account belongs to your project and is linked to your billing plan on the product's
  purchase page. You get the Secret one of two ways: **Deploy on GKE** creates it in your cluster
  automatically, or the **Deploy via command line** tab gives you the manifest to apply yourself.
  Its name is the `metering-agent.reporting.secretName` value.
- A **Volt license**. Request it from `sales@voltactivedata.com`, quoting the order or entitlement
  id for your purchase. One file licenses VoltSP and VoltDB; the product does not start without it.
  Paste the XML on **one line**: the deploy form is a single-line field, and a license with its line
  breaks removed still validates. On the Helm path use `--set-file`, which reads the file as it is.

  The license carries an expiry date, and nothing renews it. VoltSP and VoltDB stop when it passes,
  so request a replacement before then and apply it with `helm upgrade`, or by deploying again.

## What the deploy form asks for

| Field | Required | Description |
|---|---|---|
| Existing Kubernetes cluster | yes | The GKE cluster to install into. |
| Namespace | yes | An existing namespace or a new one. One instance per namespace. |
| App instance name | yes | Names the deployed instance and its Helm release. |
| Volt license (XML) | yes | The license Volt issued for your purchase, on a single line. |
| CPU cores reserved / allowed per VoltSP pod | no (default `2`) | Whole cores, 1 to 16. Keep the two equal. |
| Memory reserved / allowed per VoltSP pod | no (default `2Gi`) | Keep the two equal. |
| CPU cores reserved / allowed per VoltDB node | no (default `4`) | Whole cores, 1 to 16. Keep the two equal. |
| Memory reserved / allowed per VoltDB node | no (default `4Gi`) | Keep the two equal. |
| Reporting service account | yes | The service account usage is reported with. The form creates a key for it and puts the reporting secret above into the namespace. |

That is the whole list. There is no pipeline, replica or autoscaling field: a form takes one line of
text per field, which cannot hold a pipeline, and those settings belong to the
[Helm path](./HELM.md).

Before raising CPU or memory, check that a node can host the larger pod. The cluster check at
deploy time uses fixed sizes, not the values you enter, so a pod no node can hold stays `Pending`.

## Check the deployment

Point `kubectl` at the cluster, then look at the pods:

```bash
gcloud container clusters get-credentials CLUSTER \
    --location LOCATION \
    --project PROJECT
```

```bash
kubectl -n NAMESPACE get pods
```

```bash
kubectl -n NAMESPACE wait --for=condition=Available deployment --all --timeout=600s
```

VoltSP runs the demonstration pipeline; the Metering Agent reports usage. To run your own pipeline,
use the [Helm path](./HELM.md).

## Uninstall

Delete the instance from **Kubernetes Engine → Applications** in the Cloud Console, or:

```bash
kubectl delete application INSTANCE_NAME -n NAMESPACE
```

A Helm-installed release is removed with `helm uninstall`, see [HELM.md](./HELM.md#uninstall). After
`helm uninstall` the reporting secret stays; deleting an instance in the Cloud Console also deletes
the reporting secret the console created. Either way the VoltDB volume stays, and the subscription
itself is cancelled in the Cloud Marketplace console.

## Help with pipelines and Helm values

If you work with an AI coding agent, Volt publishes **agent skills** — instruction files that teach
the agent how to build and configure Volt applications, following the
[Agent Skills](https://agentskills.io) open standard. Two of them cover what this product needs:

| Skill | Use it for |
|---|---|
| [`voltsp`](https://github.com/VoltDB/volt-skills/tree/main/skills/voltsp) | Writing, testing and troubleshooting pipelines in the YAML or Java API, runtime configuration and secrets, and deploying them on Kubernetes. |
| [`volt-kubernetes`](https://github.com/VoltDB/volt-skills/tree/main/skills/volt-kubernetes) | Writing the Helm and Kubernetes configuration itself, as code. |

Install one by copying it into your project's skills directory:

```bash
git clone https://github.com/VoltDB/volt-skills.git
```

```bash
cp -r volt-skills/skills/voltsp your-project/.claude/skills/voltsp
```

See [volt-skills](https://github.com/VoltDB/volt-skills) for the full list and for other agents.
The skills are optional: everything in [HELM.md](./HELM.md) works without them.

## Support

- VoltSP documentation: https://docs.voltactivedata.com/ActiveSP/
- VoltDB on Kubernetes: https://docs.voltactivedata.com/KubernetesAdmin/
- Volt Active Data: https://www.voltactivedata.com
