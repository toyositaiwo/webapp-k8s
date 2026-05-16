# webapp — Kubernetes Deployment (Kustomize + Sealed Secrets)

A GitOps-friendly Kubernetes deployment structure for webapp (nginx) across staging and production environments using Kustomize overlays and Sealed Secrets for secret management.

## Repository Structure
k8s/
+-- base/
¦   +-- deployment.yaml
¦   +-- service.yaml
¦   +-- kustomization.yaml
+-- overlays/
+-- staging/
¦   +-- namespace.yaml
¦   +-- deployment-patch.yaml
¦   +-- sealed-secret.yaml
¦   +-- kustomization.yaml
+-- production/
+-- namespace.yaml
+-- deployment-patch.yaml
+-- sealed-secret.yaml
+-- kustomization.yaml
## Environment Differences

| Setting   | Staging    | Production  |
|-----------|------------|-------------|
| Namespace | staging    | production  |
| Replicas  | 1          | 3           |
| Image tag | nginx:1.25 | nginx:1.27  |

## How to Deploy

Prerequisite: kubectl configured against your cluster and kustomize v5+ installed.

### Deploy to Staging
kustomize build k8s/overlays/staging | kubectl apply -f -

### Deploy to Production
kustomize build k8s/overlays/production | kubectl apply -f -

### Validate Without a Cluster
kustomize build k8s/overlays/staging
kustomize build k8s/overlays/production

## Secret Management

### Approach: Bitnami Sealed Secrets

I chose Sealed Secrets because it is the most straightforward GitOps-native solution:

- Secrets are asymmetrically encrypted using the cluster public key via the kubeseal CLI
- The SealedSecret manifest is safe to commit to a public repository
- Only the in-cluster controller can decrypt it
- No external secret store required

### In-Cluster Setup (one-time)
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

### Generating Real Sealed Secrets
kubeseal --fetch-cert > pub-cert.pem

kubectl create secret generic webapp-secret \
  --from-literal=DB_PASSWORD='your-db-password' \
  --from-literal=API_KEY='your-api-key' \
  --namespace=staging \
  --dry-run=client -o yaml > /tmp/secret.yaml

kubeseal --cert pub-cert.pem --format yaml < /tmp/secret.yaml \
  > k8s/overlays/staging/sealed-secret.yaml

Note: The sealed-secret.yaml files in this repo contain placeholder stub values.
Replace them with real sealed values generated against your cluster public key before deploying.

### Why Not Plain Secrets?
Plain Kubernetes Secrets store values as base64 which is encoding, not encryption.
Committing them to a public repository would expose credentials in plain text.

## Assumptions and Trade-offs

| Decision | Reasoning |
|----------|-----------|
| Sealed Secrets over Vault | Simpler setup, no external dependency |
| ClusterIP Service | Safe default, no ingress requirement specified |
| No resource limits in base | Not specified in task, should be added in production |
| Stub sealed secrets | No real cluster available, structure is correct |

## What I Would Improve Given More Time

1. Add resource requests and limits to the Deployment
2. Add a Horizontal Pod Autoscaler for production
3. Add liveness and readiness probes
4. Add an Ingress manifest per environment
5. Add a CI pipeline to run kustomize build on every PR
6. Automate secret rotation per environment
