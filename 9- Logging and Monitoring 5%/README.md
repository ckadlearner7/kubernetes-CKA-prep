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
