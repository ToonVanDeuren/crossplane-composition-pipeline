# crossplane-composition-pipeline

A working example for the blog post [Inside the Composition Pipeline](https://toon.consulting/2026/05/23/inside-the-composition-pipeline/).

This repo demonstrates how a Crossplane composition pipeline executes end to end: how `function-extra-resources` pulls in shared infrastructure, how `function-go-templating` renders managed resources, how `composition-resource-name` and `crossplane.io/external-name` keep identity stable across reconciliation cycles, and how `function-auto-ready` ties the composite's `Ready` condition to the underlying cloud state.

The example provisions a PostgreSQL database on a **shared Azure Flexible Server** that the composition does not own. The server is assumed to exist already (in a real platform it would be provisioned by Terraform); the composition declares only the database, and surfaces the connection details the application needs.

## Layout

```
01-setup/                   one-time cluster bootstrap
  functions.yaml            function-go-templating, function-extra-resources, function-auto-ready
  providers.yaml            provider-azure-dbforpostgresql
  provider-configs.yaml     ClusterProviderConfig per environment
  rbac.yaml                 ClusterRole + binding granting every SA in
                            crossplane-system verbs on the dbforpostgresql
                            API groups

02-environment-configs/     describes shared infra the composition references
  postgres-development.yaml
  postgres-staging.yaml
  postgres-production.yaml

03-platform/                the platform team's API and implementation
  xrd.yaml                  PostgresDatabase composite type
  composition.yaml          four-step pipeline

04-examples/                what a developer would create
  payments-db.yaml
```

## What happens when a developer creates a `PostgresDatabase`

1. **`fetch-environment-config`** — `function-extra-resources` selects the `EnvironmentConfig` labelled `workload=postgres,environment=<spec.environment>` and places it in the pipeline context.
2. **`render-database`** — `function-go-templating` reads `spec.databaseName` from the composite, reads the server details and database defaults (charset, collation, admin user) from the `EnvironmentConfig`, and renders a `FlexibleServerDatabase` managed resource.
3. **`render-connection-secret`** — a second `function-go-templating` step emits a native Kubernetes `Secret` in the consumer's namespace with `DATABASE_HOST`, `DATABASE_NAME`, and `DATABASE_USER`. Crossplane v2 composes regular Kubernetes resources directly, so no `provider-kubernetes` wrapper is needed when the target cluster is the one Crossplane runs in.
4. **`ensure-ready`** — `function-auto-ready` promotes the composite to `READY: True` only when every managed resource the pipeline produced is itself `Ready`.

## Running it locally

```bash
# Install Crossplane (one-time)
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system --create-namespace

# Apply in order — wait for the provider to become healthy before continuing
kubectl apply -f 01-setup/
kubectl wait provider/provider-azure-dbforpostgresql \
  --for=condition=Healthy --timeout=120s

# Edit 02-environment-configs/*.yaml first — replace <subscription-id>,
# <resource-group>, <server>, and <admin-user> with the real values for
# your shared Flexible Server in each environment.
kubectl apply -f 02-environment-configs/

kubectl apply -f 03-platform/

# Create the Azure service principal Secret the ClusterProviderConfig references.
# Repeat per environment you want to exercise (development, staging, production).
kubectl create secret generic azure-creds-production \
  --namespace crossplane-system \
  --from-literal=credentials='{"clientId":"<client-id>","clientSecret":"<client-secret>","subscriptionId":"<subscription-id>","tenantId":"<tenant-id>"}'

# Create a database
kubectl create namespace team-payments
kubectl apply -f 04-examples/payments-db.yaml

# Watch the composite reconcile
kubectl get postgresdatabases.platform.example.com -A
kubectl describe postgresdatabase payments -n team-payments
```

The composition picks the right `ClusterProviderConfig` from `spec.environment`. Create one credential Secret per environment you want to exercise (the snippet above shows production).

## Render the composition without applying it

```bash
crossplane render \
  04-examples/payments-db.yaml \
  03-platform/composition.yaml \
  01-setup/functions.yaml \
  --extra-resources 02-environment-configs/postgres-production.yaml
```

This runs the pipeline locally and prints the resources it would produce. Useful for iterating on the template without round-tripping through Azure.

## What this example deliberately does not show

- **Custom function development** — covered in a separate post
- **KCL** — covered in a separate post
- **Domain/Solution/App platform architecture** — covered in a separate post
- **Backups, point-in-time-restore, server SKU tuning** — these are properties of the shared server, owned by the team that runs it (typically DBAs via Terraform)
