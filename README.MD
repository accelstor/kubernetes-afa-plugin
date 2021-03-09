# 如何在kubernetes安裝accelcsi

## Setup

### Step 1

先clone [external-snapshotter](https://github.com/kubernetes-csi/external-snapshotter.git)後，執行安裝，請注意，本命令每個cluster只需一次

``` shell
    git clone https://github.com/kubernetes-csi/external-snapshotter.git
    kubectl create -f external-snapshotter/client/config/crd
    kubectl create -f external-snapshotter/deploy/kubernetes/snapshot-controller
```

### Step 2

安裝我們的[csi](https://github.com/accelstor/kubernetes-afa-plugin)

``` shell
    kubectl apply -f https://raw.githubusercontent.com/accelstor/kubernetes-afa-plugin/main/rbac-external-provisioner.yaml
    kubectl apply -f https://raw.githubusercontent.com/accelstor/kubernetes-afa-plugin/main/rbac-csi-snapshotter.yaml
    kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-attacher/master/deploy/kubernetes/rbac.yaml
```

下載並指向AFA正確IP，填入正確AFATOKEN

``` shell
    curl https://raw.githubusercontent.com/accelstor/kubernetes-afa-plugin/main/accelcsi.yaml
    vim accelcsi.yaml
```

從[env](https://github.com/accelstor/kubernetes-afa-plugin/blob/main/accelcsi.yaml#L117)位置處開始編輯

``` yaml
          env:
            - name: AFA_IP
              value: "10.146.6.205"
            - name: AFA_KEY
              value: "fa3fed9fceda15dd7960c3431c406f58848fd22f"
            - name: AFA_HTTPS
              value: "false"
```

apply剛剛編輯好的檔案，csi plugin就會正確地把volume掛載

``` shell
kubectl apply -f accelcsi.yaml
```

這樣csi就算安裝完成了

## Usage

在實際將volume分配給pod前，我們須先使用csi plugin來create pvc，以下範例分別分配出1G、10G、40G大小的空間

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sc.csi.accelstor.com
provisioner: csi.accelstor.com
reclaimPolicy: Delete
volumeBindingMode: Immediate

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1.csi.accelstor.com
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: sc.csi.accelstor.com # defined in csi-setup.yaml
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2.csi.accelstor.com
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: sc.csi.accelstor.com # defined in csi-setup.yaml

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc3.csi.accelstor.com
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 40Gi
  storageClassName: sc.csi.accelstor.com # defined in csi-setup.yaml
```

create一個pod，並把pvc配置給該pod

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: pvc1.csi.accelstor.com # defined in csi-pvs.yaml
```

這樣pod就會自動將該volume配置給pod，以下是pod狀態

``` shell
Name:         shell-demo
Namespace:    default
Priority:     0
Node:         accelstor-hs-218/10.144.1.218
Start Time:   Tue, 12 Jan 2021 14:40:31 +0800
Labels:       <none>
Annotations:  <none>
Status:       Pending
IP:           
IPs:          <none>
Containers:
  nginx:
    Container ID:   
    Image:          nginx
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from shared-data (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-74smr (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  shared-data:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  pvc1.csi.accelstor.com
    ReadOnly:   false
  default-token-74smr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-74smr
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
```

![機器狀況](images/1.PNG)

## 備註

- Fibre channel的機器目前不支援kubernetes的plugin支援
- 你可能需要修改accelcsi.yaml的機器位置和token以適應自己的機器

## rbac-csi-snapshotter.yaml
https://github.com/kubernetes-csi/external-snapshotter/blob/master/deploy/kubernetes/csi-snapshotter/rbac-csi-snapshotter.yaml

## rbac-external-provisioner.yaml
https://github.com/kubernetes-csi/external-provisioner/blob/master/deploy/kubernetes/rbac.yaml
