# Dynamic PVC and PV for StatefulSet – MongoDB Setup

## 1. Overview

To enable **dynamic PV allocation** for a MongoDB StatefulSet, we install the **Local Path Provisioner CSI** driver in the Kubernetes cluster.

Official GitHub link:
[https://github.com/rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)

Install using:

```sh
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.32/deploy/local-path-storage.yaml
```

OR use the manual YAML (shown below).

---

## 2. local-path-storage.yaml

Below is the complete YAML definition used for local dynamic provisioning.

> **Note:**
> In this configuration, the **ConfigMap path is changed to `/data`** instead of the official default `/opt/local-path-provisioner`.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: local-path-storage
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: local-path-provisioner-service-account
  namespace: local-path-storage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: local-path-provisioner-role
  namespace: local-path-storage
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: local-path-provisioner-role
rules:
  - apiGroups: [""]
    resources: ["nodes", "persistentvolumeclaims", "configmaps", "pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: local-path-provisioner-bind
  namespace: local-path-storage
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: local-path-provisioner-role
subjects:
  - kind: ServiceAccount
    name: local-path-provisioner-service-account
    namespace: local-path-storage
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-path-provisioner-bind
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: local-path-provisioner-role
subjects:
  - kind: ServiceAccount
    name: local-path-provisioner-service-account
    namespace: local-path-storage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-path-provisioner
  namespace: local-path-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-path-provisioner
  template:
    metadata:
      labels:
        app: local-path-provisioner
    spec:
      serviceAccountName: local-path-provisioner-service-account
      containers:
        - name: local-path-provisioner
          image: rancher/local-path-provisioner:v0.0.32
          imagePullPolicy: IfNotPresent
          command:
            - local-path-provisioner
            - --debug
            - start
            - --config
            - /etc/config/config.json
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config/
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: CONFIG_MOUNT_PATH
              value: /etc/config/
      volumes:
        - name: config-volume
          configMap:
            name: local-path-config
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
    {
      "nodePathMap":[
        {
          "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
          "paths":["/data"]
        }
      ]
    }
  setup: |-
    #!/bin/sh
    set -eu
    mkdir -m 0777 -p "$VOL_DIR"
  teardown: |-
    #!/bin/sh
    set -eu
    rm -rf "$VOL_DIR"
  helperPod.yaml: |-
    apiVersion: v1
    kind: Pod
    metadata:
      name: helper-pod
    spec:
      priorityClassName: system-node-critical
      tolerations:
        - key: node.kubernetes.io/disk-pressure
          operator: Exists
          effect: NoSchedule
      containers:
      - name: helper-pod
        image: busybox
        imagePullPolicy: IfNotPresent
```

---

## 3. Headless Service (mongo-service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: autox-dev
  name: mongo-db-autox
  labels:
    app: mongo-db-autox
spec:
  ports:
    - port: 27017
  clusterIP: None
  selector:
    app: mongo-db-autox
```

---

## 4. MongoDB StatefulSet (statefulset.yaml)

Creates **3 MongoDB pods**, each with its own dynamic PV/PVC.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo-db-autox
  namespace: autox-dev
spec:
  serviceName: mongo-db-autox
  replicas: 3
  selector:
    matchLabels:
      app: mongo-db-autox
  template:
    metadata:
      labels:
        app: mongo-db-autox
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
             - matchExpressions:
               - key: node-type
                 operator: In
                 values:
                  - "k8-autox-worker1"
      containers:
        - name: mongo-db
          image: emindsguardians/twinarc:mongodb
          ports:
            - containerPort: 27017
              name: mongodb
          volumeMounts:
            - name: mongodb-storage
              mountPath: /data/db
          command:
            - /bin/sh
            - -c
            - |
              echo "Starting MongoDB..."
              mongod --replSet rs0 --bind_ip_all --dbpath /data/db &

              sleep 15

              if [ "$(hostname)" = "mongo-db-autox-0" ]; then
                echo "Initializing MongoDB Replica Set..."
                mongo --eval '
                  rs.initiate({
                    _id: "rs0",
                    members: [
                      { _id: 0, host: "mongo-db-autox-0.mongo-db-autox.autox-dev.svc.cluster.local:27017" },
                      { _id: 1, host: "mongo-db-autox-1.mongo-db-autox.autox-dev.svc.cluster.local:27017" },
                      { _id: 2, host: "mongo-db-autox-2.mongo-db-autox.autox-dev.svc.cluster.local:27017" }
                    ]
                  })
                '
              else
                echo "Replica member pod detected — skipping rs.initiate."
              fi
              wait
  volumeClaimTemplates:
    - metadata:
        name: mongodb-storage
      spec:
        storageClassName: local-path
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Gi
```

---

## 5. Deployment Steps

```sh
kubectl apply -f local-path-storage.yaml
kubectl apply -f mongo-service.yaml
kubectl apply -f statefulset.yaml
```

---

## 6. PVC/PV Allocation

Each MongoDB StatefulSet pod gets its own dynamically created PV:

Example generated paths:

```
/data/pvc-ec3545db-a557-429a-98c4-97acfc042a48_autox-dev_mongodb-storage-mongo-db-autox-0
/data/pvc-ec3545db-a557-429a-98c4-97acfc042a48_autox-dev_mongodb-storage-mongo-db-autox-1
/data/pvc-ec3545db-a557-429a-98c4-97acfc042a48_autox-dev_mongodb-storage-mongo-db-autox-2
```

Watch the CSI pod logs:

```sh
kubectl get pods -n local-path-storage -w
```

---

## 7. Replica Set Behavior

* One pod becomes **primary**
* Other two become **secondary**
* If the primary is deleted, a secondary is auto-promoted
* Each pod uses **its own PV/PVC**

Internal cluster connection string:

```
mongodb://mongo-db-autox:27017/workflow
```

---

## 8. MongoDB Data Migration Scripts (If needed)

Copy all `.json` files into directory:

```
mongo_scripts/
```

Example files:

```
workflow.activities.json
workflow.connectors.json
workflow.departments.json
workflow.processes.json
...
```

Copy the folder to the **mongo-db-autox-0** pod.

Run:

```sh
./mongodb_load.sh
```

If errors occur, run commands one by one.

---

## 9. mongodb_load.sh

```sh
mongoimport --db workflow --collection departments --file workflow.departments.json --jsonArray
mongoimport --db workflow --collection activities --file workflow.activities.json --jsonArray
mongoimport --db workflow --collection activityversions --file workflow.activityversions.json --jsonArray
mongoimport --db workflow --collection connectors --file workflow.connectors.json --jsonArray
mongoimport --db workflow --collection emailtemplates --file workflow.emailtemplates.json --jsonArray
mongoimport --db workflow --collection functions --file workflow.functions.json --jsonArray
mongoimport --db workflow --collection packages --file workflow.packages.json --jsonArray
mongoimport --db workflow --collection processes --file workflow.processes.json --jsonArray
mongoimport --db workflow --collection processversions --file workflow.processversions.json --jsonArray
mongoimport --db workflow --collection triggers --file workflow.triggers.json --jsonArray
mongoimport --db workflow --collection workjams --file workflow.workjams.json --jsonArray
```

---

## 10. Checking Primary and Secondary Pods

To verify which MongoDB pod is **PRIMARY** or **SECONDARY**:

1. Exec into any MongoDB pod:

```sh
kubectl exec -it -n autox-dev mongo-db-autox-0 -- bash
```

2. Start the Mongo shell:

```sh
mongo
```

3. MongoDB will display whether the pod is:

* `PRIMARY`
* `SECONDARY`

Example prompt:

```
rs0:PRIMARY>
rs0:SECONDARY>
```

---

