# GLPI on Azure Kubernetes Service (AKS)

Step-by-step runbook for deploying GLPI on the AKS cluster `aksprodwesteu01` with Helm.

## Current design

- AKS cluster: `aksprodwesteu01`
- Kubernetes namespace: `glpi`
- Helm release: `glpi`
- Helm chart: `eftechcombr/glpi`
- GLPI application version used during this deployment: `11.0.8`
- Database: MariaDB deployed by the chart
- Cache: Redis deployed by the chart
- Exposure: Azure `LoadBalancer`, HTTP only
- Public service: `nginx`
- HTTPS/Ingress: disabled
- Persistent storage: AKS default Azure Disk storage, `ReadWriteOnce`
- GLPI CronJob: suspended because the shared GLPI volumes are `ReadWriteOnce`

> **Security warning:** This setup exposes GLPI over unencrypted HTTP. Do not enter production credentials or sensitive ticket data until HTTPS, DNS, access restrictions, backups, and authentication have been configured.

---

## 1. Prerequisites

The administrator workstation needs:

- Azure CLI
- `kubectl`
- Helm
- OpenSSL
- Permission to access and deploy to the AKS cluster

On Arch Linux / Omarchy:

```bash
sudo pacman -S --needed azure-cli kubectl helm openssl
```

Verify the tools:

```bash
az version
kubectl version --client
helm version
openssl version
```

---

## 2. Connect to the correct AKS cluster

Authenticate to Azure:

```bash
az login
```

If the AKS credentials are not already configured, replace the placeholders and run:

```bash
export AKS_RESOURCE_GROUP="CHANGE-ME"
export AKS_CLUSTER_NAME="aksprodwesteu01"

az aks get-credentials \
  --resource-group "$AKS_RESOURCE_GROUP" \
  --name "$AKS_CLUSTER_NAME" \
  --overwrite-existing
```

Verify the active context and nodes:

```bash
kubectl config current-context
kubectl get nodes -o wide
```

Expected context:

```text
aksprodwesteu01
```

Do not continue if another Kubernetes context is active.

---

## 3. Verify Helm access

```bash
helm version
helm list --all-namespaces
```

If both commands succeed, Helm can communicate with AKS.

> On the Helm version used during the original installation, `helm list --all` was not supported. Use `helm list -n glpi`, `helm list -n glpi --failed`, or `helm list -n glpi --pending` instead.

---

## 4. Add the GLPI Helm repository

```bash
helm repo add eftechcombr https://eftechcombr.github.io/glpi/
helm repo update
helm search repo eftechcombr/glpi --versions | head
```

Create a working directory:

```bash
mkdir -p ~/glpi-aks
cd ~/glpi-aks
```

Save the chart defaults for troubleshooting:

```bash
helm show values eftechcombr/glpi > default-values.yaml
```

---

## 5. Check AKS storage classes

```bash
kubectl get storageclass
```

This deployment used the AKS default storage class. The resulting GLPI PVCs used Azure Disk with `ReadWriteOnce` access.

Important consequence:

- nginx and PHP-FPM mount the same GLPI volumes.
- They must run on the same Kubernetes node when Azure Disk RWO storage is used.
- The GLPI installation job also needs those volumes during installation.
- The GLPI CronJob cannot run concurrently on another node with those volumes.

For a later production redesign, use shared `ReadWriteMany` storage for the shared GLPI directories.

---

## 6. Remove an old test installation, if present

Check whether an old release or namespace exists:

```bash
helm list -n glpi
kubectl get namespace glpi
```

For a clean test reinstallation only, remove the entire namespace:

```bash
kubectl delete namespace glpi
kubectl wait \
  --for=delete namespace/glpi \
  --timeout=180s
```

Confirm removal:

```bash
kubectl get namespace glpi 2>/dev/null \
  || echo "OK: Namespace glpi was removed"
```

> **Destructive action:** Deleting the namespace deletes the GLPI workloads and PVC claims in that namespace. Never do this after GLPI contains required configuration or ticket data unless a verified restore path exists.

Create the namespace:

```bash
kubectl create namespace glpi
kubectl get namespace glpi
```

---

## 7. Generate database passwords

Always generate new passwords. Do not reuse passwords shown in terminals, tickets, chat messages, or screenshots.

```bash
cd ~/glpi-aks

export GLPI_DB_PASSWORD="$(openssl rand -hex 32)"
export GLPI_DB_ROOT_PASSWORD="$(openssl rand -hex 32)"

umask 077

cat > glpi-passwords.txt <<EOF
GLPI_DB_NAME=glpi
GLPI_DB_USER=glpi
GLPI_DB_PASSWORD=${GLPI_DB_PASSWORD}
GLPI_DB_ROOT_PASSWORD=${GLPI_DB_ROOT_PASSWORD}
EOF

chmod 600 glpi-passwords.txt
```

Verify permissions without printing the secrets:

```bash
ls -l glpi-passwords.txt
```

Expected permissions:

```text
-rw-------
```

Move these secrets to the organization's approved secret store before production use.

---

## 8. Create the Helm values file

The nginx service configuration belongs under `glpi.nginx.service`. A top-level `service.type` value is ignored by this chart.

Create the values file with `<<EOF`:

```bash
cat > values-glpi.yaml <<EOF
glpi:
  nginx:
    replicaCount: 1
    service:
      type: LoadBalancer
      port: 80
      targetPort: 8080

ingress:
  enabled: false

mariadb:
  enabled: true
  auth:
    rootPassword: "${GLPI_DB_ROOT_PASSWORD}"
    database: glpi
    username: glpi
    password: "${GLPI_DB_PASSWORD}"
  primary:
    persistence:
      enabled: true
      storageClass: managed-csi
      accessModes:
        - ReadWriteOnce
      size: 30Gi

redis:
  enabled: true
EOF
```

> The chart created some PVCs with the cluster's `default` storage class despite the value above. Always inspect the rendered chart and actual PVCs rather than assuming every supplied key was consumed.

---

## 9. Render and validate before installation

Render the chart locally:

```bash
helm template glpi eftechcombr/glpi \
  --namespace glpi \
  --values values-glpi.yaml \
  > rendered-glpi.yaml
```

Confirm a LoadBalancer service is present:

```bash
if grep -q 'type: LoadBalancer' rendered-glpi.yaml; then
  echo "OK: LoadBalancer service will be created"
else
  echo "ERROR: No LoadBalancer found in rendered chart"
fi
```

Confirm that Ingress is disabled:

```bash
if grep -q '^kind: Ingress$' rendered-glpi.yaml; then
  echo "ERROR: An Ingress resource was rendered"
else
  echo "OK: No Ingress will be created"
fi
```

Validate against the AKS API server:

```bash
kubectl apply \
  --dry-run=server \
  -f rendered-glpi.yaml
```

Do not continue if validation reports an error.

---

## 10. Install GLPI

Install without `--atomic`. During the original deployment, `--atomic` removed the failed Helm release when the initialization job could not mount its RWO volumes, leaving Kubernetes resources without a manageable deployed release.

```bash
helm install glpi eftechcombr/glpi \
  --namespace glpi \
  --values values-glpi.yaml
```

Check the release:

```bash
helm list -n glpi
helm status glpi -n glpi
```

Check the created resources:

```bash
kubectl get pods,pvc,svc,cronjob -n glpi -o wide
```

The nginx service should appear as `LoadBalancer` and receive an Azure external IP.

---

## 11. Suspend the GLPI CronJob immediately

The chart creates `glpi-cronjob`. With Azure Disk RWO volumes, the CronJob can fail with `Multi-Attach` while nginx/PHP-FPM are using the same volumes.

Suspend it:

```bash
kubectl patch cronjob glpi-cronjob \
  -n glpi \
  --type=merge \
  -p '{"spec":{"suspend":true}}'
```

Delete any CronJob job that already started:

```bash
kubectl get jobs -n glpi -o name \
  | grep '^job.batch/glpi-cronjob-' \
  | xargs -r kubectl delete -n glpi
```

Verify:

```bash
kubectl get cronjob glpi-cronjob -n glpi
```

Expected:

```text
SUSPEND   True
ACTIVE    0
```

Do not reactivate this CronJob until shared storage or another safe scheduling/storage solution has been implemented.

---

## 12. Allow the database installation job to mount the GLPI volumes

The job `glpi-db-install` initializes the GLPI database and writes configuration to the shared GLPI volumes.

If it remains at `Init:0/1` and events show `Multi-Attach`, temporarily stop nginx and PHP-FPM:

```bash
kubectl scale deployment nginx php-fpm \
  -n glpi \
  --replicas=0
```

Wait for both web pods to be deleted:

```bash
kubectl wait \
  --for=delete pod \
  -l app.kubernetes.io/component=nginx \
  -n glpi \
  --timeout=180s

kubectl wait \
  --for=delete pod \
  -l app.kubernetes.io/component=php-fpm \
  -n glpi \
  --timeout=180s
```

Observe the installation job:

```bash
kubectl get pods -n glpi -w
```

Follow its logs:

```bash
kubectl logs \
  -n glpi \
  job/glpi-db-install \
  --all-containers=true \
  --follow
```

Successful output ends with:

```text
Installation done.
```

Confirm job completion:

```bash
kubectl get job glpi-db-install -n glpi
```

Expected:

```text
STATUS        Complete
COMPLETIONS   1/1
```

Do not restart nginx/PHP-FPM until this job is complete.

---

## 13. Start PHP-FPM first

```bash
kubectl scale deployment php-fpm \
  -n glpi \
  --replicas=1

kubectl rollout status deployment/php-fpm \
  -n glpi \
  --timeout=180s
```

Get the node selected for PHP-FPM:

```bash
PHP_NODE="$(
  kubectl get pod \
    -n glpi \
    -l app.kubernetes.io/component=php-fpm \
    -o jsonpath='{.items[0].spec.nodeName}'
)"

echo "PHP-FPM node: ${PHP_NODE}"
```

---

## 14. Pin nginx and PHP-FPM to the same node

Because the GLPI volumes are RWO Azure Disks, nginx and PHP-FPM must mount them from the same node.

Pin PHP-FPM to its current node:

```bash
kubectl patch deployment php-fpm \
  -n glpi \
  --type=merge \
  -p "{
    \"spec\": {
      \"template\": {
        \"spec\": {
          \"nodeSelector\": {
            \"kubernetes.io/hostname\": \"${PHP_NODE}\"
          }
        }
      }
    }
  }"
```

Pin nginx to the same node:

```bash
kubectl patch deployment nginx \
  -n glpi \
  --type=merge \
  -p "{
    \"spec\": {
      \"template\": {
        \"spec\": {
          \"nodeSelector\": {
            \"kubernetes.io/hostname\": \"${PHP_NODE}\"
          }
        }
      }
    }
  }"
```

Start nginx:

```bash
kubectl scale deployment nginx \
  -n glpi \
  --replicas=1
```

Check both rollouts:

```bash
kubectl rollout status deployment/php-fpm \
  -n glpi \
  --timeout=180s

kubectl rollout status deployment/nginx \
  -n glpi \
  --timeout=180s
```

Verify that both pods run on the same node:

```bash
kubectl get pods \
  -n glpi \
  -l 'app.kubernetes.io/component in (nginx,php-fpm)' \
  -o wide
```

The `NODE` value must be identical for both pods.

> This direct `kubectl patch` is not represented in `values-glpi.yaml`. A later Helm upgrade may replace it. Before any chart upgrade, inspect the rendered manifests and preserve an equivalent node-affinity configuration or migrate the shared volumes to RWX storage.

---

## 15. Retrieve and test the GLPI HTTP address

Show the service:

```bash
kubectl get svc nginx -n glpi -o wide
```

Extract the external IP:

```bash
GLPI_IP="$(
  kubectl get svc nginx \
    -n glpi \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
)"

echo "GLPI URL: http://${GLPI_IP}"
```

Test it:

```bash
curl -I "http://${GLPI_IP}"
```

Open the displayed URL in a browser.

Initial credentials documented by the chart image:

```text
Username: glpi
Password: glpi
```

Change the password immediately after the first successful sign-in.

---

## 16. Final health check

```bash
echo "=== Kubernetes context ==="
kubectl config current-context

echo
echo "=== Helm release ==="
helm list -n glpi
helm status glpi -n glpi

echo
echo "=== Jobs and CronJob ==="
kubectl get jobs,cronjob -n glpi

echo
echo "=== Pods ==="
kubectl get pods -n glpi -o wide

echo
echo "=== PVCs ==="
kubectl get pvc -n glpi

echo
echo "=== Services ==="
kubectl get svc -n glpi -o wide

echo
echo "=== Recent events ==="
kubectl get events -n glpi \
  --sort-by='.lastTimestamp' \
  | tail -30
```

Expected application state:

- `glpi-db-install`: `Complete`, `1/1`
- `mariadb-0`: `Running`, `1/1`
- nginx: `Running`, `1/1`
- PHP-FPM: `Running`, `1/1`
- Redis: `Running`, `1/1`
- nginx and PHP-FPM are on the same node
- `glpi-cronjob`: `SUSPEND=True`, `ACTIVE=0`
- nginx service: `LoadBalancer` with an external IP

---

## 17. Troubleshooting

### Helm says `release: not found`

Check:

```bash
helm list -n glpi
helm list -n glpi --failed
helm list -n glpi --pending
```

If an installation was run with rollback/atomic behavior and failed, Kubernetes resources may have been removed or left without a deployed release. For a brand-new test instance, remove the namespace and reinstall cleanly. Do not delete the namespace if it contains required data.

### `helm upgrade` says `has no deployed releases`

There is no successfully deployed Helm revision to upgrade. For a new empty installation, clean the namespace and use `helm install`, not `helm upgrade`.

### `helm list -n glpi --all` returns `unknown flag: --all`

Use:

```bash
helm list -n glpi
helm list -n glpi --failed
helm list -n glpi --pending
helm history glpi -n glpi
```

### Pod remains `ContainerCreating` or `Init:0/1`

Describe it:

```bash
kubectl describe pod POD_NAME -n glpi
```

Show recent events:

```bash
kubectl get events -n glpi \
  --sort-by='.lastTimestamp' \
  | tail -40
```

If events show `Multi-Attach`, ensure all pods mounting the GLPI RWO PVCs run on the same node, or stop the web pods temporarily for one-time installation jobs.

### nginx readiness probe returns 502

Check PHP-FPM:

```bash
kubectl get pods -n glpi -o wide
kubectl logs -n glpi deployment/php-fpm --tail=200
kubectl logs -n glpi deployment/nginx --tail=200
```

Ensure PHP-FPM is ready and nginx is scheduled on the same node.

### Browser shows `The GLPI database must be configured and installed`

Check the database installation job:

```bash
kubectl get job glpi-db-install -n glpi
kubectl logs -n glpi job/glpi-db-install --all-containers=true --tail=200
```

The job must be `Complete 1/1`. If it is blocked by Multi-Attach, stop nginx/PHP-FPM, allow the job to finish, then restart the web deployments on the same node.

### Service remains `ClusterIP`

The correct chart key is:

```yaml
glpi:
  nginx:
    service:
      type: LoadBalancer
      port: 80
      targetPort: 8080
```

A top-level `service.type` is ignored by this chart.

### Get logs

```bash
kubectl logs -n glpi deployment/nginx --tail=200
kubectl logs -n glpi deployment/php-fpm --tail=200
kubectl logs -n glpi statefulset/mariadb --tail=200
kubectl logs -n glpi deployment/redis --tail=200
```

### Port-forward for private testing

```bash
kubectl port-forward -n glpi svc/nginx 8080:80
```

Open `http://localhost:8080` locally.

`broken pipe` messages from `kubectl port-forward` can occur when a browser connection is closed or refreshed; confirm application health with HTTP responses and pod logs.

---

## 18. Operational commands

### Restart the web tier

```bash
kubectl rollout restart deployment/php-fpm deployment/nginx -n glpi
kubectl rollout status deployment/php-fpm -n glpi --timeout=180s
kubectl rollout status deployment/nginx -n glpi --timeout=180s
```

After a restart, verify nginx and PHP-FPM remain on the same node.

### Show current Helm values

```bash
helm get values glpi -n glpi
```

Be careful: this can print database passwords to the terminal. Do not paste the output into tickets or chat.

### Show release status

```bash
helm status glpi -n glpi
```

### Show recent events

```bash
kubectl get events -n glpi \
  --sort-by='.lastTimestamp' \
  | tail -50
```

---

## 19. Known limitations and required production follow-up

This runbook reproduces the working HTTP deployment. Before production use, complete these items:

1. Replace HTTP with HTTPS and a controlled DNS name.
2. Restrict public network access or use an internal LoadBalancer according to the access model.
3. Move shared GLPI volumes to an appropriate RWX storage design so nginx, PHP-FPM, jobs, and CronJobs do not depend on same-node scheduling.
4. Re-enable scheduled GLPI tasks only after the storage/scheduling design is safe.
5. Implement off-cluster, tested backups for MariaDB and GLPI files.
6. Store secrets in the approved enterprise secret store and rotate any exposed credentials.
7. Configure Entra ID/LDAP/SSO according to organizational policy.
8. Change or disable all default GLPI accounts.
9. Configure monitoring, alerts, resource limits, update procedures, and restore testing.
10. Validate Copilot/API access separately with least-privilege identities.

---

## 20. Destructive removal

Only for an empty test system:

```bash
helm uninstall glpi -n glpi
kubectl delete namespace glpi
```

This deletes the workloads and namespace-scoped claims. Do not use it for a production system without verified backups and an approved restore plan.
