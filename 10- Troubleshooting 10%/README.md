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
## Troubleshooting Worker Node Failure
Troubleshooting worker node failure is a lot like troubleshooting a non-responsive server, in addition to the kubectl tools we have at our disposal. In this lesson, we’ll learn how to recover a node and add it back to the cluster and find out how to identify when the kublet service is down.

Listing the status of the nodes should be the first step:

    kubectl get nodes

Find out more information about the nodes with kubectl describe:

    kubectl describe nodes chadcrowell2c.mylabserver.com

You can try to log in to your server via SSH:

    ssh chadcrowell2c.mylabserver.com

Get the IP address of your nodes:

    kubectl get nodes -o wide

Use the IP address to further probe the server:

    ssh cloud_user@172.31.29.182

Generate a new token after spinning up a new server:

    sudo kubeadm token generate

Create the kubeadm join command for your new worker node:

    sudo kubeadm token create [token_name] --ttl 2h --print-join-command

View the journalctl logs:

    sudo journalctl -u kubelet

View the syslogs:

    sudo more syslog | tail -120 | grep kubelet
