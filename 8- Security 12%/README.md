# Securing the Kubernetes Cluster
## Kubernetes Security Primitives
Expanding on our discussion about securing the Kubernetes cluster, we’ll take a look at service accounts and user authentication. Also in this lesson, we will create a workstation for you to administer your cluster without logging in to the Kubernetes master server.

List the service accounts in your cluster:

    kubectl get serviceaccounts

Create a new jenkins service account:

    kubectl create serviceaccount jenkins

Use the abbreviated version of serviceAccount:

    kubectl get sa

View the YAML for our service account:

    kubectl get serviceaccounts jenkins -o yaml

View the secrets in your cluster:

    kubectl get secret [secret_name]

The YAML for a busybox pod using the jenkins service account:

    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
      namespace: default
    spec:
      serviceAccountName: jenkins
      containers:
      - image: busybox:1.28.4
        command:
          - sleep
          - "3600"
        imagePullPolicy: IfNotPresent
        name: busybox
      restartPolicy: Always

Create a new pod with the service account:

    kubectl apply -f busybox.yaml

View the cluster config that kubectl uses:

    kubectl config view

View the config file:

    cat ~/.kube/config

Set new credentials for your cluster:

    kubectl config set-credentials chad --username=chad --password=password

Create a role binding for anonymous users (not recommended):

    kubectl create clusterrolebinding cluster-system-anonymous --clusterrole=cluster-admin --user=system:anonymous

SCP the certificate authority to your workstation or server:

    scp ca.crt cloud_user@[pub-ip-of-remote-server]:~/

Set the cluster address and authentication:

    kubectl config set-cluster kubernetes --server=https://172.31.41.61:6443 --certificate-authority=ca.crt --embed-certs=true

Set the credentials for Chad:

    kubectl config set-credentials chad --username=chad --password=password

Set the context for the cluster:

    kubectl config set-context kubernetes --cluster=kubernetes --user=chad --namespace=default

Use the context:

    kubectl config use-context kubernetes

Run the same commands with kubectl:

    kubectl get nodes

## Cluster Authentication and Authorization
Once the API server has determined who you are (whether a pod or a user), the authorization is handled by RBAC. In this lesson, we will talk about roles, cluster roles, role bindings, and cluster role bindings.

Create a new namespace:

    kubectl create ns web

The YAML for a service role:

    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: web
      name: service-reader
    rules:
    - apiGroups: [""]
      verbs: ["get", "list"]
      resources: ["services"]

Create a new role from that YAML file:

    kubectl apply -f role.yaml

Create a RoleBinding:

    kubectl create rolebinding test --role=service-reader --serviceaccount=web:default -n web

Run a proxy for inter-cluster communications:

    kubectl proxy

Try to access the services in the web namespace:

    curl localhost:8001/api/v1/namespaces/web/services

Create a ClusterRole to access PersistentVolumes:

    kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes

Create a ClusterRoleBinding for the cluster role:

    kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=web:default

The YAML for a pod that includes a curl and proxy container:

    apiVersion: v1
    kind: Pod
    metadata:
      name: curlpod
      namespace: web
    spec:
      containers:
      - image: tutum/curl
        command: ["sleep", "9999999"]
        name: main
      - image: linuxacademycontent/kubectl-proxy
        name: proxy
      restartPolicy: Always

Create the pod that will allow you to curl directly from the container:

    kubectl apply -f curl-pod.yaml

Get the pods in the web namespace:

    kubectl get pods -n web

Open a shell to the container:

    kubectl exec -it curlpod -n web -- sh

Access PersistentVolumes (cluster-level) from the pod:

    curl localhost:8001/api/v1/persistentvolumes

## Configuring Network Policies
Network policies allow you to specify which pods can talk to other pods. This helps when securing communication between pods, allowing you to identify ingress and egress rules. You can apply a network policy to a pod by using pod or namespace selectors. You can even choose a CIDR block range to apply the network policy. In this lesson, we’ll go through each of these options for network policies.

Download the canal plugin:

    wget -O canal.yaml https://docs.projectcalico.org/v3.5/getting-started/kubernetes/installation/hosted/canal/canal.yaml

Apply the canal plugin:

    kubectl apply -f canal.yaml

The YAML for a deny-all NetworkPolicy:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: deny-all
    spec:
      podSelector: {}
      policyTypes:
      - Ingress

Run a deployment to test the NetworkPolicy:

    kubectl run nginx --image=nginx --replicas=2

Create a service for the deployment:

    kubectl expose deployment nginx --port=80

Attempt to access the service by using a busybox interactive pod:

    kubectl run busybox --rm -it --image=busybox /bin/sh
    #wget --spider --timeout=1 nginx

The YAML for a pod selector NetworkPolicy:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: db-netpolicy
    spec:
      podSelector:
        matchLabels:
          app: db
      ingress:
      - from:
        - podSelector:
            matchLabels:
              app: web
        ports:
        - port: 5432

Label a pod to get the NetworkPolicy:

    kubectl label pods [pod_name] app=db

The YAML for a namespace NetworkPolicy:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: ns-netpolicy
    spec:
      podSelector:
        matchLabels:
          app: db
      ingress:
      - from:
        - namespaceSelector:
            matchLabels:
              tenant: web
        ports:
        - port: 5432

The YAML for an IP block NetworkPolicy:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: ipblock-netpolicy
    spec:
      podSelector:
        matchLabels:
          app: db
      ingress:
      - from:
        - ipBlock:
            cidr: 192.168.1.0/24

The YAML for an egress NetworkPolicy:

    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: egress-netpol
    spec:
      podSelector:
        matchLabels:
          app: web
      egress:
      - to:
        - podSelector:
            matchLabels:
              app: db
        ports:
        - port: 5432

