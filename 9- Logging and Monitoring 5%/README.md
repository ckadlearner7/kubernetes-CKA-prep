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

## Managing Cluster Component Logs
There are many ways to manage the logs that can accumulate from both applications and system components. In this lesson, we’ll go through a few different approaches to organizing your logs.

The directory where the continainer logs reside:

    /var/log/containers

The directory where kubelet stores its logs:

    /var/log

The YAML for a pod that has two different log streams:

    apiVersion: v1
    kind: Pod
    metadata:
      name: counter
    spec:
      containers:
      - name: count
        image: busybox
        args:
        - /bin/sh
        - -c
        - >
          i=0;
          while true;
          do
            echo "$i: $(date)" >> /var/log/1.log;
            echo "$(date) INFO $i" >> /var/log/2.log;
            i=$((i+1));
            sleep 1;
          done
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        emptyDir: {}

Create a pod that has two different log streams to the same directory:

    kubectl apply -f twolog.yaml

View the logs in the /var/log directory of the container:

    kubectl exec counter -- ls /var/log

The YAML for a sidecar container that will tail the logs for each type:

    apiVersion: v1
    kind: Pod
    metadata:
      name: counter
    spec:
      containers:
      - name: count
        image: busybox
        args:
        - /bin/sh
        - -c
        - >
          i=0;
          while true;
          do
            echo "$i: $(date)" >> /var/log/1.log;
            echo "$(date) INFO $i" >> /var/log/2.log;
            i=$((i+1));
            sleep 1;
          done
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      - name: count-log-1
        image: busybox
        args: [/bin/sh, -c, 'tail -n+1 -f /var/log/1.log']
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      - name: count-log-2
        image: busybox
        args: [/bin/sh, -c, 'tail -n+1 -f /var/log/2.log']
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        emptyDir: {}

View the first type of logs separately:

    kubectl logs counter count-log-1

View the second type of logs separately:

    kubectl logs counter count-log-2
## Managing Application Logs
Containerized applications usually write their logs to standard out and standard error instead of writing their logs to files. Docker then redirects those streams to files. You can retrieve those files with the kubectl logs command in Kubernetes. In this lesson, we’ll go over the many ways to manipulate the output of your logs and redirect them to a file.

Get the logs from a pod:

    kubectl logs nginx

Get the logs from a specific container on a pod:

    kubectl logs counter -c count-log-1

Get the logs from all containers on the pod:

    kubectl logs counter --all-containers=true

Get the logs from containers with a certain label:

    kubectl logs -lapp=nginx

Get the logs from a previously terminated container within a pod:

    kubectl logs -p -c nginx nginx

Stream the logs from a container in a pod:

    kubectl logs -f -c count-log-1 counter

Tail the logs to only view a certain number of lines:

    kubectl logs --tail=20 nginx

View the logs from a previous time duration:

    kubectl logs --since=1h nginx

View the logs from a container within a pod within a deployment:

    kubectl logs deployment/nginx -c nginx

Redirect the output of the logs to a file:

    kubectl logs counter -c count-log-1 > count.log
