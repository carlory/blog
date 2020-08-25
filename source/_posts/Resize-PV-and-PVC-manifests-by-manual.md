---
title: Resize PV and PVC manifests by Manual
date: 2020-08-25 15:42:07
tags:
- Kubernetes
- Storage
---

## 环境准备

```shell
$ kubectl create -f sc.yaml -f pvc.yaml -f pv.yaml

$ kubectl get pvc test-pvc

NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-pvc   Bound    test-pv   200Gi      RWX            manual         18m

$ kubectl get pv test-pv

NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
test-pv   200Gi      RWX            Retain           Bound    carlory/test-pvc   manual                    18m
```

编排清单：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: manual
provisioner: fake
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-pv
spec:
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 200Gi
  nfs:
    path: /if/kubernetes/volumes/test-pv
    server: 192.168.101.106
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  volumeName: test-pv
  storageClassName: manual
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
```

## 扩容示例

**Step 1**: 修改PVC的请求容量到400Gi

```shell
$ kubectl get pvc test-pvc -ojson | jq 'setpath(["spec", "resources", "requests", "storage"]; "400Gi")' | kubectl apply -f - 

persistentvolumeclaim/test-pvc configured

$ kubectl get pvc test-pvc

 NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
 test-pvc   Bound    test-pv   200Gi      RWX            manual         20m

$ kubectl get pvc test-pvc -oyaml

 apiVersion: v1
 kind: PersistentVolumeClaim
 metadata:
   annotations:
     pv.kubernetes.io/bind-completed: "yes"
   creationTimestamp: "2020-03-31T15:07:20Z"
   finalizers:
   - kubernetes.io/pvc-protection
   name: test-pvc
   namespace: carlory
   resourceVersion: "60619371"
   selfLink: /api/v1/namespaces/carlory/persistentvolumeclaims/test-pvc
   uid: 51b2b9bb-7361-11ea-af09-005056b4d66c
 spec:
   accessModes:
   - ReadWriteMany
   resources:
     requests:
       storage: 400Gi
   storageClassName: manual
   volumeName: test-pv
 status:
   accessModes:
   - ReadWriteMany
   capacity:
     storage: 200Gi
   conditions:
   - lastProbeTime: null
     lastTransitionTime: "2020-03-31T15:31:07Z"
     status: "True"
     type: Resizing
   phase: Bound
```

**Step 2**: 修改PV的容量到400Gi

```shell
$ kubectl get pv test-pv -ojson | jq 'setpath(["spec", "capacity", "storage"]; "400Gi")' | kubectl apply -f -

persistentvolume/test-pv configured

$ kubectl get pv test-pv
 NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
 test-pv   400Gi      RWX            Retain           Bound    carlory/test-pvc    manual                    30m
```

**Step 3**: 删除PVC的Resizing的状态

```shell
# shell 1

$ kubectl proxy

# shell 2
$ data=$(oc get pvc test-pvc -ojson | jq 'setpath(["status", "capacity", "storage"]; "400Gi")' | jq 'setpath(["status", "conditions"]; [])')
$ curl -XPUT -H "Content-Type: application/json" -d $data  127.0.0.1:8001/api/v1/namespaces/carlory/persistentvolumeclaims/test-pvc/status

$ kubectl get pvc

 NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
 test-pvc   Bound    test-pv   400Gi      RWX            manual         37m

 $ kubectl get pvc test-pvc -oyaml

 apiVersion: v1
 kind: PersistentVolumeClaim
 metadata:
   annotations:
     pv.kubernetes.io/bind-completed: "yes"
   creationTimestamp: "2020-03-31T15:07:20Z"
   finalizers:
   - kubernetes.io/pvc-protection
   name: test-pvc
   namespace: carlory
   resourceVersion: "60622486"
   selfLink: /api/v1/namespaces/carlory/persistentvolumeclaims/test-pvc
   uid: 51b2b9bb-7361-11ea-af09-005056b4d66c
 spec:
   accessModes:
   - ReadWriteMany
   resources:
     requests:
       storage: 400Gi
   storageClassName: manual
   volumeName: test-pv
 status:
   accessModes:
   - ReadWriteMany
   capacity:
     storage: 400Gi
   phase: Bound
```
