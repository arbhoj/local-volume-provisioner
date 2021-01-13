# [Creating Local-Volume-Provisioner Storage Classes]() 

Reference: https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner

For scenarios where local-volume-provisioner is being used to serve Persistent Volumes (PV) and the size of these volumes varies significantly (e.g. some PVCs require 10 Gi while some requie 2Ti), it is benificial to create storage classes to represent these different size categories.  The Persistent Volume Claim (PVC) resources or volumeClaimTemplates in a StatefulSet can then reference these classes using the storageClass property to ensure that a PVC with a 10Gi request does not get bound to a 2Ti PV.

This involves configuration in the following places:
- The worker nodes to create a `discovery directory` for each class and mount volumes to these
- The konvoy configuration in cluster.yaml to define the storage class objects and reference the `discovery directory` for each

![](https://via.placeholder.com/10/F90E09/000000?text=+) Note: These two steps are just one time. Once configured, any volume mounted to the `discovery directory` will automatically be processed to form an PV in the given class. Follow the same steps given below to add more storage classes as needed.

[ **Worker Node Configuration**]()

Do the following on the worker nodes that are going to serve local-volume-provisioner PVs 

1. Create a `discovery directory` for each storage class

    ```
    e.g. 
    sudo mkdir -p /mnt/disks #This is for default 

    sudo mkdir -p /mnt/xsmall

    sudo mkdir -p /mnt/small

    sudo mkdir -p /mnt/medium

    sudo mkdir -p /mnt/large

    sudo mkdir -p /mnt/xlarge
    ```

1. Format and mount the volumes to the appropriate `discovery directory`

```
e.g.
sudo mkfs.ext4 /dev/sdh 
DISK_UUID=$(sudo blkid -s UUID -o value blkid /dev/sdh)

sudo mkdir /mnt/small/$DISK_UUID

sudo mount -t ext4 /dev/sdh /mnt/small/$DISK_UUID

echo UUID=`sudo blkid -s UUID -o value /dev/sdh` /mnt/small/$DISK_UUID ext4 defaults 0 2 | sudo tee -a /etc/fstab
```

Mount more volumes as required to the appropriate `discovery directories`


[ **Konvoy cluster.yaml Configuration**]()

Update the `localvolumeprovisioner` section in cluster.yaml to create the storage classes and tie them with the discovery directories inside the `/mnt` directory. 

    - name: localvolumeprovisioner
      enabled: true
      values: |
        # Multiple storage classes can be defined here. This allows to, e.g.,
        # distinguish between different disk types.
        # For each entry a storage class '$name' and
        # a host folder '/mnt/$dirName' will be created. Volumes mounted to this
        # folder are made available in the storage class.
        storageclasses:
          - name: localvolumeprovisioner
            dirName: disks
            isDefault: true
            reclaimPolicy: Delete
            volumeBindingMode: WaitForFirstConsumer
          - name: small
            dirName: small
            isDefault: false
            reclaimPolicy: Delete
            volumeBindingMode: WaitForFirstConsumer
          - name: medium
            dirName: medium
            isDefault: false
            reclaimPolicy: Delete
            volumeBindingMode: WaitForFirstConsumer
          - name: large
            dirName: large
            isDefault: false
            reclaimPolicy: Delete
            volumeBindingMode: WaitForFirstConsumer


Apply the changes to the cluster. This will create the storage classes defined in cluster.yaml

```
konvoy deploy addons
```
Once the storage classes have been created, they will automatically create PVs for the given classes using the appropriate volumes that were mounted their respective `discovery directory`. 

```
kubectl get pv

NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-1715b44e   4911Mi     RWO            Delete           Available           medium                  166m
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-3af920aa   9951Mi     RWO            Delete           Available           large                   3h55m
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-88d834a6   975Mi      RWO            Delete           Available           small                   3m35s
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-a96874e1   1014Mi     RWO            Delete           Available           small                   3h39m
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-b3ae36be   4911Mi     RWO            Delete           Available           medium                  166m
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-dc4bbb2b   975Mi      RWO            Delete           Available           small                   3m35s
```

The auto-generated PVs automatically have the nodeAffinity and volume path defined

```
e.g. 
kubectl get pv local-pv-1715b44e -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: local-volume-provisioner-ip-10-0-132-70.us-west-2.compute.internal-dba161b4-005a-426d-85c1-8b1cc5122122
  creationTimestamp: "2020-05-14T20:32:24Z"
  finalizers:
  - kubernetes.io/pv-protection
  name: local-pv-1715b44e
  resourceVersion: "95013"
  selfLink: /api/v1/persistentvolumes/local-pv-1715b44e
  uid: d0240e5d-486a-4029-aa31-634c3ae2638e
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 4911Mi
  local:
    fsType: ext4
    path: /mnt/medium/35e5d833-cd99-4a98-9dcf-5e335a772bc2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - ip-10-0-132-70.us-west-2.compute.internal
  persistentVolumeReclaimPolicy: Delete
  storageClassName: medium
  volumeMode: Filesystem
status:
  phase: Available

```


[ **Testing**]()

Run the following to test that the classes are working as expected. 
Note: These use the small, medium and large example. Update as required to fit your usecase.

```
kubectl apply -f  statefulsmall.yaml
kubectl apply -f  statefulmedium.yaml
kubectl apply -f  statefullarge.yaml

```

