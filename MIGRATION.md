# Migrating from `default` to the `portfolio` namespace

A `PersistentVolume` (PV) isn't namespaced — only the `PersistentVolumeClaim`
(PVC) that binds to it is. So moving namespaces doesn't require copying any
data: just release the old PVC, keep its PV around, and bind a new PVC (in
the new namespace) to that same PV. Postgres and Directus uploads land in
the new namespace still pointing at their original storage.

Run these from a machine with `kubectl` pointed at the cluster.

## 1. Create the namespace and copy the secrets

```sh
kubectl apply -f namespace.yaml

kubectl get secret portfolio-secrets -n default -o yaml \
  | sed 's/namespace: default/namespace: portfolio/' \
  | kubectl apply -n portfolio -f -

kubectl get secret ghcr-secret -n default -o yaml \
  | sed 's/namespace: default/namespace: portfolio/' \
  | kubectl apply -n portfolio -f -
```

## 2. Scale down the old workloads

Needed so nothing has the volumes mounted when we detach the PVCs:

```sh
kubectl scale deployment/portfolio portfolio-cms portfolio-db -n default --replicas=0
```

## 3. Detach the PVCs, keep the PVs

```sh
DB_PV=$(kubectl get pvc portfolio-db-data -n default -o jsonpath='{.spec.volumeName}')
UP_PV=$(kubectl get pvc portfolio-directus-uploads -n default -o jsonpath='{.spec.volumeName}')

kubectl patch pv "$DB_PV" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl patch pv "$UP_PV" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

kubectl delete pvc portfolio-db-data portfolio-directus-uploads -n default

kubectl patch pv "$DB_PV" --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
kubectl patch pv "$UP_PV" --type=json -p='[{"op":"remove","path":"/spec/claimRef"}]'
```

Retain stops deletion of the underlying storage; deleting the PVC just
drops the *binding*, and the claimRef removal makes the PV `Available`
again for a new claim to pick up.

## 4. Re-bind them in the new namespace

```sh
cat <<EOF | kubectl apply -n portfolio -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: portfolio-db-data
  namespace: portfolio
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 2Gi}}
  volumeName: $DB_PV
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: portfolio-directus-uploads
  namespace: portfolio
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 5Gi}}
  volumeName: $UP_PV
EOF
```

Setting `volumeName` explicitly forces the bind to that exact PV instead
of provisioning a new one.

## 5. Apply the rest of the manifests

```sh
kubectl apply -n portfolio -f portfolio-db.yaml
kubectl apply -n portfolio -f portfolio-cms.yaml
kubectl apply -n portfolio -f portfolio-deployment.yaml
```

These files also define the two PVCs (without `volumeName`) — applying
them just reconciles the same objects you already bound in step 4;
`volumeName` is immutable and won't be touched.

## 6. Verify

```sh
kubectl get pods -n portfolio -w
```

- `http://192.168.0.175:30081` (Directus) — your existing content and
  uploaded files should already be there, no restore step needed.
- The portfolio frontend loads and pulls from Directus (`DIRECTUS_URL:
  http://portfolio-cms-svc` still resolves fine — same short-name DNS,
  just now within the `portfolio` namespace).
- If the frontend is reached via the Traefik `IngressRoute`, confirm
  Traefik is watching the `portfolio` namespace (default installs watch
  all namespaces; if yours is scoped via
  `--providers.kubernetescrd.namespaces`, add `portfolio` to that list).

## 7. Clean up `default`

```sh
kubectl delete -f portfolio-deployment.yaml -n default
kubectl delete -f portfolio-cms.yaml -n default
kubectl delete -f portfolio-db.yaml -n default
kubectl delete secret portfolio-secrets ghcr-secret -n default
```

(The PVCs there are already gone from step 3 — this just clears the
Deployments/Services/Secrets.)

## Notes

- This relies on your storage provisioner tolerating a manual PV rebind
  (true for local-path-provisioner and most CSI drivers — the PV itself
  doesn't change, only which PVC claims it). If your cluster instead uses
  something where PVs are strictly one-shot/non-reusable, fall back to a
  `pg_dump`/`tar` copy instead — ask and I'll write that version.
- `CORS_ORIGIN` / `PUBLIC_URL` are node IP:NodePort values, not
  namespace-relative — no change needed.
- Don't run step 7 until step 6 is confirmed working.
