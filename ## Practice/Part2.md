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


