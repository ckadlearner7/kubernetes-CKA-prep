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
