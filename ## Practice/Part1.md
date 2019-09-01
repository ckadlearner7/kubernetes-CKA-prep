# Objectives
You have been given access to a three-node cluster. Within that cluster, you will be responsible for creating a service for end users of your web application. You will ensure the application meets the specifications set by the developers and the proper resources are in place to ensure maximum uptime for the app. You must perform the following tasks in order to complete this hands-on lab:

* All objects should be in the web namespace.
* The deployment name should be webapp.
* The deployment should have 3 replicas.
* The deployment’s pods should have one container using the linuxacademycontent/podofminerva image with the tag latest.
* The service should be named web-service.
* The service should forward traffic to port 80 on the pods.
* The service should be exposed externally by listening on port 30080 on each node.
* The pods should be configured to check the /healthz. endpoint on port 8081, and automatically restart the container if the check fails.
* The pods should be configured to not receive traffic until the endpoint on port 80 responds successfully.

Learning Objectives
Create a deployment named `webapp` in the `web` namespace and verify connectivity.

Use the following command to create a namespace named web:

kubectl create ns web

    Use the following command to create a deployment named webapp:

     kubectl run webapp --image=linuxacademycontent/podofminerva:latest --port=80 --replicas=3 -n web

check_circle
Create a service named `web-service` and forward traffic from the pods.
keyboard_arrow_up

    Use the following command to get the IP address of a pod that’s a part of the deployment:

     kubectl get po -o wide -n web

    Use the following command to create a temporary pod with a shell to its container:

     kubectl run busybox --image=busybox --rm -it --restart=Never -- sh

    Use the following command (from the container’s shell) to send a request to the web pod:

     wget -O- <pod_ip_address>:80

    Use the following command to create the YAML for the service named web-service:

     kubectl expose deployment/webapp --port=80 --target-port=80 --type=NodePort -n web --dry-run -o yaml > web-service.yaml

    Use Vim to add the namespace and the NodePort to the YAML:

     vim web-service.yaml

    Change the name to web-service, add the namespace web, and add nodePort: 30080.

    Use the following command to create the service:

     kubectl apply -f web-service.yaml

    Use the following command to verify that the service is responding on the correct port:

     curl localhost:30080

    Use the following command to modify the deployment:

     kubectl edit deploy webapp -n web

    Add the liveness probe and the readiness probe:

     livenessProbe:
       httpGet:
         path: /healthz
         port: 8081
     readinessProbe:
       httpGet:
         path: /
         port: 80

    Use the following command to check if the pods are running:

     kubectl get po -n web

    Use the following command to check if the probes were added to the pods:

     kubectl get po <pod_name> -o yaml -n web --export
