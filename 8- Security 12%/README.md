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
