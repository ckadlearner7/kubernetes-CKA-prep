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
