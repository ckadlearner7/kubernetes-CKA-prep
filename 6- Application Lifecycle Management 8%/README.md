## Application Lifecycle Management
### Deploying an Application, Rolling Updates, and Rollbacks

We already know Kubernetes will run pods and deployments, but what happens when you need to update or change the version of your application running inside of the Kubernetes cluster? That’s where rolling updates come in, allowing you to update the app image with zero downtime. In this lesson, we’ll go over a rolling update, how to roll back, and how to pause the update if things aren’t going well.

The YAML for a deployment:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kubeserve
    spec:
      replicas: 3
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
          - image: linuxacademycontent/kubeserve:v1
            name: app

1- Create a deployment with a record (for rollbacks):

    kubectl create -f kubeserve-deployment.yaml --record

2- Check the status of the rollout:

    kubectl rollout status deployments kubeserve

3- View the ReplicaSets in your cluster:

    kubectl get replicasets

4- Scale up your deployment by adding more replicas:

     kubectl scale deployment kubeserve --replicas=5

5- Expose the deployment and provide it a service:

    kubectl expose deployment kubeserve --port 80 --target-port 80 --type NodePort

6- Set the minReadySeconds attribute to your deployment:

    kubectl patch deployment kubeserve -p '{"spec": {"minReadySeconds": 10}}'

7- Use kubectl apply to update a deployment:

    kubectl apply -f kubeserve-deployment.yaml

8- Use kubectl replace to replace an existing deployment:

    kubectl replace -f kubeserve-deployment.yaml

9- Run this curl look while the update happens:

    while true; do curl http://10.105.31.119; done

10- Perform the rolling update:

    kubectl set image deployments/kubeserve app=linuxacademycontent/kubeserve:v2 --v 6

11- Describe a certain ReplicaSet:
 
    kubectl describe replicasets kubeserve-[hash]

12- Apply the rolling update to version 3 (buggy):

    kubectl set image deployment kubeserve app=linuxacademycontent/kubeserve:v3

13- Undo the rollout and roll back to the previous version:

    kubectl rollout undo deployments kubeserve

14- Look at the rollout history:

    kubectl rollout history deployment kubeserve

15- Roll back to a certain revision:

    kubectl rollout undo deployment kubeserve --to-revision=2

16- Pause the rollout in the middle of a rolling update (canary release):

    kubectl rollout pause deployment kubeserve

17- Resume the rollout after the rolling update looks good:

    kubectl rollout resume deployment kubeserve

### Configuring an Application for High Availability and Scale
Continuing from the last lesson, we will go through how Kubernetes will save you from EVER releasing code with bugs. Then, we will talk about ConfigMaps and secrets as a way to pass configuration data to your apps.

The YAML for a readiness probe:

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kubeserve
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: kubeserve
      minReadySeconds: 10
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 0
        type: RollingUpdate
      template:
        metadata:
          name: kubeserve
          labels:
            app: kubeserve
        spec:
          containers:
          - image: linuxacademycontent/kubeserve:v3
            name: app
            readinessProbe:
              periodSeconds: 1
              httpGet:
                path: /
                port: 80

1- Apply the readiness probe:

    kubectl apply -f kubeserve-deployment-readiness.yaml

2- View the rollout status:

    kubectl rollout status deployment kubeserve

3- Describe deployment:

    kubectl describe deployment

4- Create a ConfigMap with two keys:

    kubectl create configmap appconfig --from-literal=key1=value1 --from-literal=key2=value2

5- Get the YAML back out from the ConfigMap:

    kubectl get configmap appconfig -o yaml

The YAML for the ConfigMap pod:

    apiVersion: v1
    kind: Pod
    metadata:
      name: configmap-pod
    spec:
      containers:
      - name: app-container
        image: busybox:1.28
        command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
        env:
        - name: MY_VAR
          valueFrom:
            configMapKeyRef:
              name: appconfig
              key: key1

6- Create the pod that is passing the ConfigMap data:

    kubectl apply -f configmap-pod.yaml

7- Get the logs from the pod displaying the value:

    kubectl logs configmap-pod

The YAML for a pod that has a ConfigMap volume attached:

    apiVersion: v1
    kind: Pod
    metadata:
      name: configmap-volume-pod
    spec:
      containers:
      - name: app-container
        image: busybox
        command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
        volumeMounts:
          - name: configmapvolume
            mountPath: /etc/config
      volumes:
        - name: configmapvolume
          configMap:
            name: appconfig

8- Create the ConfigMap volume pod:

    kubectl apply -f configmap-volume-pod.yaml

9- Get the keys from the volume on the container:

    kubectl exec configmap-volume-pod -- ls /etc/config

10- Get the values from the volume on the pod:

    kubectl exec configmap-volume-pod -- cat /etc/config/key1

The YAML for a secret:

    apiVersion: v1
    kind: Secret
    metadata:
      name: appsecret
    stringData:
      cert: value
      key: value

11- Create the secret:

    kubectl apply -f appsecret.yaml

The YAML for a pod that will use the secret:

    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-pod
    spec:
      containers:
      - name: app-container
        image: busybox
        command: ['sh', '-c', "echo Hello, Kubernetes! && sleep 3600"]
        env:
        - name: MY_CERT
          valueFrom:
            secretKeyRef:
              name: appsecret
              key: cert

12- Create the pod that has attached secret data:

    kubectl apply -f secret-pod.yaml

13- Open a shell and echo the environment variable:

    kubectl exec -it secret-pod -- sh

    echo $MY_CERT

The YAML for a pod that will access the secret from a volume:

    apiVersion: v1
    kind: Pod
    metadata:
      name: secret-volume-pod
    spec:
      containers:
      - name: app-container
        image: busybox
        command: ['sh', '-c', "echo $(MY_VAR) && sleep 3600"]
        volumeMounts:
          - name: secretvolume
            mountPath: /etc/certs
      volumes:
        - name: secretvolume
          secret:
            secretName: appsecret

14- Create the pod with volume attached with secrets:

    kubectl apply -f secret-volume-pod.yaml

15- Get the keys from the volume mounted to the container with the secrets:

    kubectl exec secret-volume-pod -- ls /etc/certs

### Creating a Self-Healing Application

In this lesson, we’ll go through the power of ReplicaSets, which make your application self-healing by replicating pods and moving them around and spinning them up when nodes fail. We’ll also talk about StatefulSets and the benefit they provide.

The YAML for a ReplicaSet:

    apiVersion: apps/v1
    kind: ReplicaSet
    metadata:
      name: myreplicaset
      labels:
        app: app
        tier: frontend
    spec:
      replicas: 3
      selector:
        matchLabels:
          tier: frontend
      template:
        metadata:
          labels:
            tier: frontend
        spec:
          containers:
          - name: main
            image: linuxacademycontent/kubeserve

1- Create the ReplicaSet:

    kubectl apply -f replicaset.yaml

The YAML for a pod with the same label as a ReplicaSet:

    apiVersion: v1
    kind: Pod
    metadata:
      name: pod1
      labels:
        tier: frontend
    spec:
      containers:
      - name: main
        image: linuxacademycontent/kubeserve

2- Create the pod with the same label:

    kubectl apply -f pod-replica.yaml

3- Watch the pod get terminated:

    kubectl get pods -w 

4- The YAML for a StatefulSet:

    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: web
    spec:
      serviceName: "nginx"
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
              name: web
            volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
      volumeClaimTemplates:
      - metadata:
          name: www
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi

5- Create the StatefulSet:

    kubectl apply -f statefulset.yaml

6- View all StatefulSets in the cluster:

    kubectl get statefulsets

7- Describe the StatefulSets:

    kubectl describe statefulsets
