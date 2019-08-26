# Monitoring Cluster Components
## Monitoring Cluster Components
We are able to monitor the CPU and memory utilization of our pods and nodes by using the metrics server. In this lesson, we’ll install the metrics server and see how the kubectl top command works.

Clone the metrics server repository:

    git clone https://github.com/linuxacademy/metrics-server

Install the metrics server in your cluster:

    kubectl apply -f ~/metrics-server/deploy/1.8+/

Get a response from the metrics server API:

    kubectl get --raw /apis/metrics.k8s.io/

Get the CPU and memory utilization of the nodes in your cluster:

    kubectl top node

Get the CPU and memory utilization of the pods in your cluster:

    kubectl top pods

Get the CPU and memory of pods in all namespaces:

    kubectl top pods --all-namespaces

Get the CPU and memory of pods in only one namespace:

    kubectl top pods -n kube-system

Get the CPU and memory of pods with a label selector:

    kubectl top pod -l run=pod-with-defaults

Get the CPU and memory of a specific pod:

    kubectl top pod pod-with-defaults

Get the CPU and memory of the containers inside the pod:

    kubectl top pods group-context --containers

## Monitoring the Applications Running within a Cluster
There are ways Kubernetes can automatically monitor your apps for you and, furthermore, fix them by either restarting or preventing them from affecting the rest of your service. You can insert liveness probes and readiness probes to do just this for custom monitoring of your applications.

The pod YAML for a liveness probe:

    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness
    spec:
      containers:
      - image: linuxacademycontent/kubeserve
        name: kubeserve
        livenessProbe:
          httpGet:
            path: /
            port: 80

The YAML for a service and two pods with readiness probes:

    apiVersion: v1
    kind: Service
    metadata:
      name: nginx
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 80
      selector:
        app: nginx
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginxpd
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:191
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5

Create the service and two pods with readiness probes:

    kubectl apply -f readiness.yaml

Check if the readiness check passed or failed:

    kubectl get pods

Check if the failed pod has been added to the list of endpoints:

    kubectl get ep

Edit the pod to fix the problem and enter it back into the service:

    kubectl edit pod [pod_name]

Get the list of endpoints to see that the repaired pod is part of the service again:

    kubectl get ep
