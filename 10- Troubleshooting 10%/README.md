# Troubleshooting
## Troubleshooting Application Failure
Application failure can happen for many reasons, but there are ways within Kubernetes that make it a little easier to discover why. In this lesson, we’ll fix some broken pods and show common methods to troubleshoot.

The YAML for a pod with a termination reason:

    apiVersion: v1
    kind: Pod
    metadata:
      name: pod2
    spec:
      containers:
      - image: busybox
        name: main
        command:
        - sh
        - -c
        - 'echo "I''ve had enough" > /var/termination-reason ; exit 1'
        terminationMessagePath: /var/termination-reason

One of the first steps in troubleshooting is usually to describe the pod:

    kubectl describe po pod2

The YAML for a liveness probe that checks for pod health:

    apiVersion: v1
    kind: Pod
    metadata:
      name: liveness
    spec:
      containers:
      - image: linuxacademycontent/candy-service:2
        name: kubeserve
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8081

View the logs for additional detail:

    kubectl logs pod-with-defaults

Export the YAML of a running pod, in the case that you are unable to edit it directly:

    kubectl get po pod-with-defaults -o yaml --export > defaults-pod.yaml

Edit a pod directly (i.e., changing the image):

    kubectl edit po nginx
## Troubleshooting Control Plane Failure
The Kubernetes Control Plane is an important component to back up and protect against failure. There are certain best practices you can take to ensure you don’t have a single point of failure. If your Control Plane components are not effectively communicating, there are a few things you can check to ensure your cluster is operating efficiently.

Check the events in the kube-system namespace for errors:

    kubectl get events -n kube-system

Get the logs from the individual pods in your kube-system namespace and check for errors:

    kubectl logs [kube_scheduler_pod_name] -n kube-system

Check the status of the Docker service:

    sudo systemctl status docker

Start up and enable the Docker service, so it starts upon bootup:

    sudo systemctl enable docker && systemctl start docker

Check the status of the kubelet service:

    sudo systemctl status kubelet

Start up and enable the kubelet service, so it starts up when the machine is rebooted:

    sudo systemctl enable kubelet && systemctl start kubelet

Turn off swap on your machine:

    sudo su -
    swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab

Check if you have a firewall running:

    sudo systemctl status firewalld

Disable the firewall and stop the firewalld service:

    sudo systemctl disable firewalld && systemctl stop firewalld
