# Objectives

You have been given access to a two-node cluster. Within that cluster, a PersistentVolume has already been created. You must identify the size of the volume in order to make a PersistentVolumeClaim and mount the volume to your pod. Once you have created the PVC and mounted it to your running pod, you must copy the contents of /etc/passwd to the volume. Finally, you will delete the pod and create a new pod with the volume mounted in order to demonstrate the persistence of data. You must perform the following tasks in order to complete this hands-on lab:

* All objects should be in the web namespace.
* The PersistentVolumeClaim name should be data-pvc.
* The PVC request should be 256 MiB.
* The access mode for the PVC should be ReadWriteOnce.
* The storage class name should be local-storage.
* The pod name should be data-pod.
* The pod image should be busybox and the tag should be 1.28.
* The pod should request the PersistentVolumeClaim named data-pvc, and the volume name should be temp-data.
* The pod should mount the volume named temp-data to the /tmp/data directory.
* The name of the second pod should be data-pod2.

# Answers: Learning Objectives
## Create the `PersistentVolumeClaim`.
1. Use the following command to check if there is already a web namespace:

      kubectl get ns

2. Use the following command to create a PersistentVolumeClaim:

    vi data-pvc.yaml #copy and paste from docs

3. Change the following in the data-pvc.yaml file:

    name: data-pvc
    namespace: web
    storageClassName: local-storage
    storage: 256Mi

4. Use the following command to create the PVC:

    kubectl apply -f data-pvc.yaml
## Create a pod mounting the volume and write data to the volume.

1. Use the following command to create the YAML for a busybox pod:

    kubectl run data-pod --image=busybox --restart=Never -o yaml --dry-run -- /bin/sh -c 'sleep 3600' > data-pod.yaml

2. Use the following command to edit the data-pod.yaml file:

    vi data-pod.yaml

3. Add the following to the data-pod.yaml file:

    namespace: web
      volumeMounts: 
      - name: temp-data
        mountPath: /tmp/data
    volumes: 
    - name: temp-data
    persistentVolumeClaim:
    claimName: data-pvc

4. Use the following command to create the pod:

    kubectl apply -f data-pod.yaml

5. Use the following command to connect to the pod:

kubectl exec -it data-pod -n web -- sh

6. Use the following command to copy the contents of the etc/passwd file to /tmp/data:

cp /etc/passwd /tmp/data/passwd

7. Use the following command to list the contents of /tmp/data:

ls /tmp/data/

## Delete the pod and create a new pod and view volume data.

1. Use the following command to delete the pod:

     kubectl delete po data-pod -n web

2. Use the following command to modify the pod YAML:

     vi data-pod.yaml

3. Change the following line in the data-pod.yaml file:

     name: data-pod2

4. Use the following command to create a new pod:

     kubectl apply -f data-pod.yaml

5. Use the following command to connect to the pod and view the contents of the volume:

     kubectl exec -it data-pod2 -n web -- sh
     # ls /tmp/data
