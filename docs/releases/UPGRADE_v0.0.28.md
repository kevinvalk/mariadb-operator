# Upgrade guide v0.0.28

> [!NOTE]  
> APIs are currently in `v1alpha1`, which implies that non backward compatible changes might happen. See [Kubernetes API versioning](https://kubernetes.io/docs/reference/using-api/#api-versioning) for more detail.

This guide illustrates, step by step, how to migrate to `v0.0.28` from previous versions.

> [!NOTE]  
> Do not attempt to skip intermediate version upgrades. Upgrade progressively through each version.

For example, if upgrading from `0.0.24` to `0.0.28`:
An attempt to upgrade from `0.0.24` directly to `0.0.28` will result in unpredictable behavior.
An attempt to upgrade from `0.0.24` to `0.0.26` and then `0.0.28` will result in success.

Some breaking changes have been introduced in this release:

- __Metadata__: https://github.com/mariadb-operator/mariadb-operator/pull/537
- __Affinity__: https://github.com/mariadb-operator/mariadb-operator/pull/566 and https://github.com/mariadb-operator/mariadb-operator/pull/568
- __Opt-in password generation__: https://github.com/mariadb-operator/mariadb-operator/pull/598

Follow these steps for upgrading:

- Install `v0.0.28` CRDs:
> [!NOTE]  
> Helm does not handle CRD upgrades. See [helm docs](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#some-caveats-and-explanations).

```bash
kubectl apply --server-side=true --force-conflicts -f https://github.com/mariadb-operator/mariadb-operator/releases/download/helm-chart-0.28.0/crds.yaml
```


- Uninstall you current `mariadb-operator` for preventing conflicts:
```bash
helm uninstall mariadb-operator
```
Alternatively, you may only downscale and delete the webhook configurations:
```bash
kubectl scale deployment mariadb-operator --replicas=0
kubectl scale deployment mariadb-operator-webhook --replicas=0
kubectl delete validatingwebhookconfiguration mariadb-operator-webhook
kubectl delete mutatingwebhookconfiguration mariadb-operator-webhook
```

- In case you are manually applying manifests, get a copy of your resources, as the CRD upgrade will wipe out fields that are no longer supported:
```bash
kubectl get mariadb mariadb -o yaml > mariadb.yaml
kubectl get maxscale maxscale -o yaml > maxscale.yaml
kubectl get backup backup -o yaml > backup.yaml
kubectl get restore restore -o yaml > restore.yaml
kubectl get restore sqljob -o yaml > sqljob.yaml
```

- Upgrade CRDs to `v0.0.28`:
> [!IMPORTANT]  
> Helm does not handle CRD upgrades. See [helm docs](https://helm.sh/docs/chart_best_practices/custom_resource_definitions/#some-caveats-and-explanations).

> [!WARNING]  
> This step will delete fields that are no longer supported in your resources.
```bash
kubectl replace -f https://github.com/mariadb-operator/mariadb-operator/releases/download/helm-chart-0.28.0/crds.yaml
```

- Perform `metadata` related migrations in `MariaDB`, `MaxScale` and `Connection` resouces:
```diff
  storage:
    volumeClaimTemplate:
-     labels:
-       k8s.mariadb.io/storage: "fast"
-     annotations:
-       k8s.mariadb.io/storage: "fast"
+     metadata:
+       labels:
+         k8s.mariadb.io/storage: "fast"
+       annotations:
+         k8s.mariadb.io/storage: "fast"
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: standard

  service:
    type: LoadBalancer
-   labels:
-     k8s.mariadb.io/service: "mysvc"
-   annotations:
-     metallb.universe.tf/loadBalancerIPs: 172.18.0.150
+   metadata:
+     labels:
+       k8s.mariadb.io/service: "mysvc"
+     annotations:
+       metallb.universe.tf/loadBalancerIPs: 172.18.0.150

  connection:
    secretName: mariadb-galera-conn
    secretTemplate:
-     labels:
-       k8s.mariadb.io/secret: "supersecret"
-     annotations:
-       k8s.mariadb.io/secret: "supersecret"
+     metadata:
+       labels:
+         k8s.mariadb.io/secret: "supersecret"
+       annotations:
+         k8s.mariadb.io/secret: "supersecret"
      key: dsn

  galera:
    initJob:
-     labels:
-       sidecar.istio.io/inject: "false"
-     annotations
-       sidecar.istio.io/inject: "false"
+     metadata:
+       labels:
+         sidecar.istio.io/inject: "false"
+       annotations
+         sidecar.istio.io/inject: "false"

  bootstrapFrom:
    restoreJob:
-     labels:
-       sidecar.istio.io/inject: "false"
-     annotations
-       sidecar.istio.io/inject: "false"
+     metadata:
+       labels:
+         sidecar.istio.io/inject: "false"
+       annotations
+         sidecar.istio.io/inject: "false"
```

- Perform `affinity` related migrations in `MariaDB`, `MaxScale`, `Backup`, `Restore` and `SqlJob` resources:
```diff
  affinity:
-   enableAntiAffinity: true
+   antiAffinityEnabled: true

  galera:
    initJob:
      affinity:
-       enableAntiAffinity: true
+       antiAffinityEnabled: true

  bootstrapFrom:
    restoreJob:
      affinity:
-       enableAntiAffinity: true
+       antiAffinityEnabled: true
```

- Perform password generation migrations in `MariaDB` and `MaxScale` resources. This only applies if you are currently relying on `Secrets` generated by the operator:
```diff
-  rootPasswordSecretKeyRef:
-    name: mariadb
-    key: root-password
+  rootPasswordSecretKeyRef:
+    name: mariadb
+    key: root-password
+    generate: true

-  passwordSecretKeyRef:
-    name: mariadb
-    key: root-password
+  passwordSecretKeyRef:
+    name: mariadb
+    key: root-password
+    generate: true
```
 
-  Upgrade `mariadb-operator` to `v0.0.28`:
```bash 
helm repo update mariadb-operator
helm upgrade --install mariadb-operator mariadb-operator/mariadb-operator --version 0.28.0 
```

- If you previously decided to downscale the operator, make sure you upscale it back:
```bash
kubectl scale deployment mariadb-operator -n default --replicas=1
kubectl scale deployment mariadb-operator-webhook -n default --replicas=1
```
