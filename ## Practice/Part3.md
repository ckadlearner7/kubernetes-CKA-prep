# Objectives

You have been given access to a three-node cluster. You will be responsible for creating a deployment and a service to serve as a front end for a web application. In addition to the web application, you must deploy a Redis database and make sure the web application can only access this database using the default port of 6379. You will first create a default-deny network policy, so all pods within your Kubernetes are not able to communicate with each other by default. Then you will create a second network policy that specifies the communication on port 6379 between the web application and the database using their label selectors. You must apply these specifications to your resources in order to complete this hands-on lab:

* Create a deployment named webfront-deploy.
* The deployment should use the image nginx with the tag 1.7.8.
* The deployment should expose container port 80 on each pod and contain 2 replicas.
* Create a service named webfront-service and expose port 80, target port 80.
* The service should be exposed externally by listening on port 30080 on each node.
* Create one pod named db-redis using the image redis and the tag latest.
* Verify that you can communicate to pods by default.
* Create a network policy named default-deny that will deny pod communication by default.
* Verify that you can no longer communicate between pods.
* Apply the label role=frontend to the web application pods and the label role=db to the database pod.
* Create a network policy that will apply an ingress rule for the pods labeled with role=db to allow traffic on port 6379 from the pods labeled role=frontend.
* Verify that you have applied the correct labels and created the correct network policies.

# Answers: Learning Objectives

## Create a deployment and a service to expose your web front end.

1. Use the following command to create the YAML for your deployment:

kubectl create deployment webfront-deploy  --image=nginx:1.7.8  --dry-run -o yaml > webfront-deploy.yaml

2. Add container port 80, to have your final YAML look like this:

apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: webfront-deploy
  name: webfront-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webfront-deploy
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: webfront-deploy
    spec:
      containers:
      - image: nginx:1.7.8
        name: nginx
        ports:
        - containerPort: 80
        resources: {}
status: {}

3. Use the following command to create your deployment:

kubectl apply -f webfront-deploy.yaml

4. Use the following command to scale up your deployment:

kubectl scale deployment/webfront-deploy --replicas=2

5. Use the following command to create the YAML for a service:

kubectl expose deployment/webfront-deploy --port=80 --target-port=80 --type=NodePort --dry-run -o yaml > webfront-service.yaml

6. Add the name and the nodePort, the complete YAML will look like this:

apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: webfront-deploy
  name: webfront-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
  selector:
    app: webfront-deploy
  type: NodePort
status:
  loadBalancer: {}

7. Use the following command to create the service:

kubectl apply -f webfront-service.yaml

8. Verify that you can communicate with your pod directly:

kubectl run busybox --rm -it --image=busybox /bin/sh

# wget -O- [pod_ip_address]:80
# wget --spider --timeout=1 webfront-service

## Create a database server to serve as the backend database.

Use the following command to create a Redis pod:

kubectl run db-redis --image=redis --restart=Never

## Create a network policy that will deny communication by default.

1. Use the following YAML (this is where you can use kubernetes.io and search "network policies" and then search for the text "default"):

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress

2. Use the following command to apply the network policy:

kubectl apply default-deny.yaml

3. Verify that communication has been disabled by default:

    kubectl run busybox --rm -it --image=busybox /bin/sh
    # wget -O- [pod_ip_address]:80

## Apply the labels and create a communication over port 6379 to the database server.

1. Use the following commands to apply the labels:

kubectl get po
kubectl label po <pod_name> role=frontend
kubectl label po db-redis role=db
kubectl get po --show-labels

2. Use the following YAML to create a network policy for the communication between the two labeled pods (copy from kubernetes.io website):

     apiVersion: networking.k8s.io/v1
     kind: NetworkPolicy
     metadata:
       name: redis-netpolicy
     spec:
       podSelector:
         matchLabels:
           role: db
       ingress:
       - from:
         - podSelector:
             matchLabels:
               role: frontend
         ports:
         - port: 6379

3. Use the following command to create the network policy:

     kubectl apply -f redis-netpolicy.yaml

4. Use the following command to view the network policies:

     kubectl get netpol

5. Use the following command to describe the custom network policy:

     kubectl describe netpol redis-netpolicy

6. Use the following command to show the labels on the pods:

     kubectl get po --show-labels
