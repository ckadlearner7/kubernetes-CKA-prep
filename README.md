# Certified Kubernetes Administrator (CKA) Exam Preparation Notes
This is a guide that will contain the technical notes & commands about the steps that will help you in the Certified Kubernetes Adminstrator Exam (CKA)
## Qucik Command Notes
Create a namespace:

    kubectl create namespace web

Run a deployment with 3 replicas on the namespace web
    
    kubectl run webapp --image=linuxacademycontent/podofminerva:latest --port=80 --replicas=3 -n web

Access pods from another busybox containers to verify their ability to handle request

    kubectl run busybox --image=nusybox -it --rm --restart=Never -- sh
    /# wget -qO-[POD_IP_ADDRESS]:[POD_PORT]

Create a service to expose the deployment on port 30080 and export it to a yaml file

    kubectl expose deployment webapp --port=80 --target-port=80 --type=NodePort -n web --dry-run -o yaml > web-service.yaml
    
Create the service

    kubectl apply -f web-service.yaml
