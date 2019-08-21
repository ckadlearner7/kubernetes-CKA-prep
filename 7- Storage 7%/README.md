## Managing Data in the Kubernetes Cluster

### Persistent Volumes

In Kubernetes, pods are ephemeral. This creates a unique challenge with attaching storage directly to the filesystem of a container. Persistent Volumes are used to create an abstraction layer between the application and the underlying storage, making it easier for the storage to follow the pods as they are deleted, moved, and created within your Kubernetes cluster. In the Google Cloud Engine, find the region your cluster is in:

    gcloud container clusters list

Using Google Cloud, create a persistent disk in the same region as your cluster:

    gcloud compute disks create --size=1GiB --zone=us-central1-a mongodb

The YAML for a pod that will use persistent disk:

    apiVersion: v1
    kind: Pod
    metadata:
      name: mongodb 
    spec:
      volumes:
      - name: mongodb-data
        gcePersistentDisk:
          pdName: mongodb
          fsType: ext4
      containers:
      - image: mongo
        name: mongodb
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        ports:
        - containerPort: 27017
          protocol: TCP

Create the pod with disk attached and mounted:

    kubectl apply -f mongodb-pod.yaml

See which node the pod landed on:

    kubectl get pods -o wide

Connect to the mongodb shell:

    kubectl exec -it mongodb mongo

Switch to the mystore database in the mongodb shell:

    use mystore

Create a JSON document to insert into the database:

    db.foo.insert({name:'foo'})

View the document you just created:

    db.foo.find()

Exit from the mongodb shell:

    exit

Delete the pod:

    kubectl delete pod mongodb

Create a new pod with the same attached disk:

    kubectl apply -f mongodb-pod.yaml

Check to see which node the pod landed on:

    kubectl get pods -o wide

Drain the node (if the pod is on the same node as before):

    kubectl drain [node_name] --ignore-daemonsets

Once the pod is on a different node, access the mongodb shell again:

    kubectl exec -it mongodb mongo

Access the mystore database again:

    use mystore

Find the document you created from before:

    db.foo.find()

The YAML for a PersistentVolume object in Kubernetes:

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: mongodb-pv
    spec:
      capacity: 
        storage: 1Gi
      accessModes:
        - ReadWriteOnce
        - ReadOnlyMany
      persistentVolumeReclaimPolicy: Retain
      gcePersistentDisk:
        pdName: mongodb
        fsType: ext4

Create the Persistent Volume resource:

    kubectl apply -f mongodb-persistentvolume.yaml

View our Persistent Volumes:

    kubectl get persistentvolumes
    
### Volume Access Modes
A PersistentVolume can be mounted on a host in any way supported by the resource provider. As shown in the table below, providers will have different capabilities and each PV’s access modes are set to the specific modes supported by that particular volume. For example, NFS can support multiple read/write clients, but a specific NFS PV might be exported on the server as read-only. Each PV gets its own set of access modes describing that specific PV’s capabilities.

The access modes are:

* ReadWriteOnce – the volume can be mounted as read-write by a single node
* ReadOnlyMany – the volume can be mounted read-only by many nodes
* ReadWriteMany – the volume can be mounted as read-write by many nodes

In the CLI, the access modes are abbreviated to:

* RWO - ReadWriteOnce
* ROX - ReadOnlyMany
* RWX - ReadWriteMany

Important! A volume can only be mounted using one access mode at a time, even if it supports many. For example, a GCEPersistentDisk can be mounted as ReadWriteOnce by a single node or ReadOnlyMany by many nodes, but not at the same time.

### Persistent Volume Claims

Persistent Volume Claims (PVCs) are a way for an application developer to request storage for the application without having to know where the underlying storage is. The claim is then bound to the Persistent Volume (PV), and it will not be released until the PVC is deleted. In this lesson, we will go through creating a PVC and accessing storage within our persistent disk.

The YAML for a PVC:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mongodb-pvc 
    spec:
      resources:
        requests:
          storage: 1Gi
      accessModes:
      - ReadWriteOnce
      storageClassName: ""

Create a PVC:

    kubectl apply -f mongodb-pvc.yaml

View the PVC in the cluster:

    kubectl get pvc

View the PV to ensure it’s bound:

    kubectl get pv

The YAML for a pod that uses a PVC:

    apiVersion: v1
    kind: Pod
    metadata:
      name: mongodb 
    spec:
      containers:
      - image: mongo
        name: mongodb
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        ports:
        - containerPort: 27017
          protocol: TCP
      volumes:
      - name: mongodb-data
        persistentVolumeClaim:
          claimName: mongodb-pvc

Create the pod with the attached storage:

    kubectl apply -f mongo-pvc-pod.yaml

Access the mogodb shell:

    kubectl exec -it mongodb mongo

Find the JSON document created in previous lessons:

    db.foo.find()

Delete the mongodb pod:

    kubectl delete pod mogodb

Delete the mongodb-pvc PVC:

    kubectl delete pvc mongodb-pvc

Check the status of the PV:

    kubectl get pv

The YAML for the PV to show its reclaim policy:

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: mongodb-pv
    spec:
      capacity: 
        storage: 1Gi
      accessModes:
        - ReadWriteOnce
        - ReadOnlyMany
      persistentVolumeReclaimPolicy: Retain
      gcePersistentDisk:
        pdName: mongodb
        fsType: ext4

### Storage Objects
There’s an even easier way to provision storage in Kubernetes with StorageClass objects. Also, your storage is safe from data loss with the “Storage Object in Use Protection” feature, which ensures any pods using a Persistent Volume will not lose the data on the volume as long as it is actively mounted. We’ve been using Google Storage for this section, but there are many different volume types you can use in Kubernetes. In this lesson, we will talk about the hostPath volume and the empty directory volume type.

See the PV protection on your volume:

    kubectl describe pv mongodb-pv

See the PVC protection for your claim:

    kubectl describe pvc mongodb-pvc

Delete the PVC:

    kubectl delete pvc mongodb-pvc

See that the PVC is terminated, but the volume is still attached to pod:

    kubectl get pvc

Try to access the data, even though we just deleted the PVC:

    kubectl exec -it mongodb mongo
    use mystore
    db.foo.find()

Delete the pod, which finally deletes the PVC:

    kubectl delete pods mongodb

Show that the PVC is deleted:

    kubectl get pvc

YAML for a StorageClass object:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: fast
    provisioner: kubernetes.io/gce-pd
    parameters:
      type: pd-ssd

Create the StorageClass type "fast":

    kubectl apply -f sc-fast.yaml

Change the PVC to include the new StorageClass object:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mongodb-pvc 
    spec:
      storageClassName: fast
      resources:
        requests:
          storage: 100Mi
      accessModes:
        - ReadWriteOnce

Create the PVC with automatically provisioned storage:

    kubectl apply -f mongodb-pvc.yaml

View the PVC with new StorageClass:

    kubectl get pvc

View the newly provisioned storage:

    kubectl get pv

The YAML for a hostPath PV:

    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv-hostpath
    spec:
      storageClassName: local-storage
      capacity:
        storage: 1Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: "/mnt/data"

The YAML for a pod with an empty directory volume:

    apiVersion: v1
    kind: Pod
    metadata:
      name: emptydir-pod
    spec:
      containers:
      - image: busybox
        name: busybox
        command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
        volumeMounts:
        - mountPath: /tmp/storage
          name: vol
      volumes:
      - name: vol
        emptyDir: {}
 
 ### Applications with Persistent Storage
In this lesson, we’ll wrap everything up in a nice little bow and create a deployment that will allow us to use our storage with our pods. This is to demonstrate how a real-world application would be deployed and used for storing data.

The YAML for our StorageClass object:

    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: fast
    provisioner: kubernetes.io/gce-pd
    parameters:
      type: pd-ssd

The YAML for our PVC:

    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: kubeserve-pvc 
    spec:
      storageClassName: fast
      resources:
        requests:
          storage: 100Mi
      accessModes:
        - ReadWriteOnce

Create our StorageClass object:

    kubectl apply -f storageclass-fast.yaml

View the StorageClass objects in your cluster:

    kubectl get sc

Create our PVC:

    kubectl apply -f kubeserve-pvc.yaml

View the PVC created in our cluster:

    kubectl get pvc

View our automatically provisioned PV:

    kubectl get pv

The YAML for our deployment:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kubeserve
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: kubeserve
      template:
        metadata:
          name: kubeserve
          labels:
            app: kubeserve
        spec:
          containers:
          - env:
            - name: app
              value: "1"
            image: linuxacademycontent/kubeserve:v1
            name: app
            volumeMounts:
            - mountPath: /data
              name: volume-data
          volumes:
          - name: volume-data
            persistentVolumeClaim:
              claimName: kubeserve-pvc

Create our deployment and attach the storage to the pods:

    kubectl apply -f kubeserve-deployment.yaml

Check the status of the rollout:

    kubectl rollout status deployments kubeserve

Check the pods have been created:

    kubectl get pods

Connect to our pod and create a file on the PV:

    kubectl exec -it [pod-name] -- touch /data/file1.txt

Connect to our pod and list the contents of the /data directory:

    kubectl exec -it [pod-name] -- ls /data
