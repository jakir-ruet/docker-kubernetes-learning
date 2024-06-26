[Storage](https://kubernetes.io/docs/concepts/storage/) 

Container File System
- Container File System is ephemeral.
- Files in container File System Exists only as long as the Container Exists.
- Data in container File System is lost as soon as Container Deleted or recreated.

***Volumes***
- Many Application needs a persistent Data.
- Volumes allows to store Data Outside the Container, while allow container to Access Data at RunTime.

***Persistent Volumes***
- Volumes offer a way to provide external storage to container within the Pod/Container Specification.
- Persistent Volumes are a bit more advanced than Volumes.
- Persistent Volumes allow user to treat Storage as an Abstract Resource and consume it using Pods.

***Volume Types***
- Volume & Persistent Volumes each have a Volume Type.
- Volume Type determines how storage will be handled.
- Various Volume Types supports in K8s:
  - NFS - Network file System
  - Cloud Storage - AWS, GCP, Azure
  - ConfigMaps & Secrets
  - File System on K8s Node

***Volumes & Volume Mount***
- Volume : In Pod Spec, user can define the storage volume available in for the Pod.
- Volume Specify the VolumeType and where the data is actually store.
- VolumeMount : VolumeMount in container spec, refer the Volume in Pod Spec and provide a MountPath.
```bash
apiVersion: v1
kind: Pod
metadata: 
  name: volume-mount-pod
spec:
  volumes:
    - name: my-sample-vol
      hostPath:
        path: /data
    containers:
      - name: volume-mount-pod
        image: nginx:latest
        command: ["/bin/sh", "-c", "echo Kubernetes DevOps"]
        volumeMounts:
          - name: volume-mount-pod
            mountPath: /output
```

***emptyDir Volume***
- emptyDir : emptyDir created when Pod is assigned to Node and Persist as long as Pod running on the Node.
- Multiple containers can refer the same emptyDir Volume.
- Multiple containers in the Pod can read and write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each container.
```bash
apiVersion: v1
kind: Pod
metadata: 
  name: volume-mount-pod
spec:
  volumes:
    - name: my-cache-vol
      emptyDir: {}
    containers:
      - name: volume-mount-pod
        image: nginx:latest
        command: ["/bin/sh", "-c", "echo Kubernetes DevOps"]
        volumeMounts:
          - name: my-cache-vol
            mountPath: /cache
```
***Share Volume***
- User can use the same volumeMounts to share the same Volume to multiple container within the Same Pod.
- This is very powerful feature which can be used to data transformation of Data Processing.
- hostPath & emtpyDir volumeType support share volumes.
```bash
apiVersion: v1
kind: Pod
metadata: 
  name: volume-mount-pod
spec:
  volumes:
    - name: my-cache-vol
      emptyDir: {}
    containers:
      - name: volume-mount-pod
        image: nginx:latest
        command: ["/bin/sh", "-c", "echo Kubernetes DevOps"]
        volumeMounts:
          - name: my-cache-vol
            mountPath: /cache
    containers:
      - name: volume-mount-pod-1
        image: nginx:latest
        command: ["/bin/sh", "-c", "echo Kubernetes DevOps"]
        volumeMounts:
          - name: my-cache-vol
            mountPath: /cache/tmp
```
Host Path Volume
```bash
ls /var/tmp
kubectl apply -f hostpath-volume-pod.yaml
kubectl get pod
ls /var/tmp # see output.txt
```
NB: Delete/Remove of the pod, the file output.txt remain unchanged.

emptyDir
```bash
kubectl apply -f empty-dir-vol.yaml
kubectl get pod
kubectl get pod -o wide # see mount point
kubectl describe pod/redis-emptydir
kubectl exec -it redis-emptydir -- /bin/bash
ls # see redis directory
ls redis # see redis directory
cd /data/redis
echo "Hello, Kubernetes How are you" >> index.txt
cat index.txt
exit
kubectl delete pod redis-emptydir
kubectl apply -f empty-dir-vol.yaml
kubectl exec -it redis-emptydir -- /bin/bash
cd /data/redis # not index.txt available
touch testfile.txt
ps -aux # if it not work
apt-get update
apt-get install procps
ps -aux # delete the pod then only delete the emptyDir. And Delete/Destroy/Remove the container the emptyDir will remain unchanged.
kubectl get pod redis-emptydir --watch # run it another tab of terminal
kill 1 # kill the redis process, it means kil the container & run again
ls redis # see testfile.txt is exist
```

Shared/Common Volume
```bash
kubectl apply -f shared-volume.yaml
kubectl get pod
kubectl describe pod/shared-multi-container # check mount point
kubectl get pod -o wide
kubectl exec -it shared-multi-container -- /bin/bash
cd /usr/share/nginx/html
cat index.html
echo "welcome to nginx server"
```

***Persistent Volumes***
- PersistentVolumes are k8s Object that allow user to treat Storage as an Abstract Resource.
- PV is resource in the cluster just like a node is a cluster resource.
- PV uses a set of Attribute to describe the underlying storage resources (Disk or Cloud Storage), which will be used to store data.
Sample manifest
```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  - name: static-persistent-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /var/tmp
  storageClassName: local-storage  
```

***Storage Classes***
- StorageClass allows K8s Administrator to Specify all type of Storage Service they offer on their Platform.
- Admin cloud create a StorageClass called Slow to describe inexpensive storage for general Development use.
- Admin cloud create a StorageClass called Fast for High I/O Operation Applications.
Sample manifest
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
```bash
kubectl apply -f localhost-storage-class.yaml
kubectl describe storageclass.storage.k8s.io/local-storage
```
Slow Storage Class
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io1
  iopsPerGB: "10"
  fsType: ext4
```
Fast Storage Class
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ext4
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - us-east-1a
```

***allowVolume Expansion***
- allowVolumeExpansion - This field can accept boolean value only.
- This is the property of StorageClass and define whether StorageClass supports the ability to resize after they are created.
- All Cloud Disk Supports this property.
Sample manifest
```bash
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true  
```

***Reclaim Policy***
- persistentVolumeReclaimPolicy - This define, how the storage will be reused, when the PVs associated PVCs are deleted.
- Retain - Keep all the data. This require manual data cleanup and prepare for reuse.
- Delete - Delete underlying storage resources automatically (Support for Cloud Resource Only).
- Recycle - Automatically delete all data in underlying storage. Allow PVs to be reuse.
Sample manifest
```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: static-persistent-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /var/tmp
  storageClassName: local-storage
  persistentVolumeReclaimPolicy: Recycle
```

***PersistentVolumeClaim***
- PersistentVolumeClaim (PVC) is a request for storage by a user.
- PVCs define a set of attribute Similar to those of PVs.
- PVCs look for a PVs that is able to meet the criteria. If it found one, will automatically be bound to that PV.
Sample manifest
```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: static-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
```
```bash
kubectl apply -f persistent-volume.yaml
kubectl describe persistentvolume/persistent-volume
kubectl get pv -o wide
```
```bash
kubectl apply -f persistent-volume-claim.yaml
kubectl describe persistentvolumeclaim/static-pvc
kubectl get pv -o wide
kubectl get pvc -o wide # status pending due to first consumer
```
Checking activities of PVC in details
```bash
kubectl apply -f persistent-volume-pod.yaml
kubectl describe pod/pvc-pod
kubectl get pv -o wide # see persistent volume created, status bound & claim is default/static-pvc
kubectl get pvc -o wide # see status bound (earlier was pending) & cname is static-pvc
kubectl edit pvc static-pvc # here static-pvc is pvc name. change the storage
kubectl delete -f persistent-volume-pod.yaml
kubectl get pvc -o wide # here static-pvc is pvc name > still unchanged
kubectl delete -f persistent-volume-claim.yaml
kubectl get pv -o wide # status change bound to available
```