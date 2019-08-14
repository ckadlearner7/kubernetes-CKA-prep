# Certified Kubernetes Administrator (CKA) Exam Preparation Notes
This is a guide that will contain the technical notes & commands about the steps that will help you in the Certified Kubernetes Adminstrator Exam (CKA)

## Core Concepts 19%
1- Get a yaml spec of a running nginx deployment:

	kubectl get deployment nginx-deployment -o yaml
		
2- Show labels of a specific pod

	kubectl get pod nginx-pod --show-lables
		
3- Add a specific label to a pod

	kubectl label pod nginx-pod env=pod
		
4- Show a specific label"env" for the pods

	kubectl get pods -L env
	
NOTE: You can perform specific bulk actions on a group of pods that have a specific label (ex: prod, testing)

5- Add an annotation to a pod (Annotation is similar to a label, but unlike labels, it is used only for information, and thus can't be used to perform actions)

	kubectl annotate deployment nginx-deployment mycompany.com/someannotation="chad"
	
NOTE: You can also filter objects using the field selectors ex:

6- Filter objects that are RUNNING using a field selector:

	kubectl get pods --field-selector status.phase=Running
		
   Filter services that are running in the current namespace:
		
	kubectl get services --field-selector metadata.namespace=default
		
   Combine the previous 2 commands 
		
	kubectl get pods --field-selector status.phase=Running,metadata.namespace!=default
	
## Installation, Configuration & Validation 12%

### Cluster Installation:

1- Get the Docker gpg key:

	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

2- Add the Docker repository:

	sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

3- Get the Kubernetes gpg key:

	curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

4- Add the Kubernetes repository:

	cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
	deb https://apt.kubernetes.io/ kubernetes-xenial main
	EOF

5- Update your packages:

	sudo apt-get update

6- Install Docker, kubelet, kubeadm, and kubectl:

	sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.13.5-00 kubeadm=1.13.5-00 kubectl=1.13.5-00

7- Hold them at the current version:

	sudo apt-mark hold docker-ce kubelet kubeadm kubectl

8- Add the iptables rule to sysctl.conf:

	echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf

9- Enable iptables immediately:

	sudo sysctl -p

10- Initialize the cluster (run only on the master):

	sudo kubeadm init --pod-network-cidr=10.244.0.0/16

11- Set up local kubeconfig:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

12- Apply Flannel CNI network overlay:

	kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml

13- Join the worker nodes to the cluster: (You run this command on all worker nodes, you usually get it from the output of the cluster initialization)

	kubeadm join [your unique string from the kubeadm init command]

14- Verify the worker nodes have joined the cluster successfully:

	kubectl get nodes

		Compare this result of the kubectl get nodes command:
		NAME                            STATUS   ROLES    AGE   VERSION
		chadcrowell1c.mylabserver.com   Ready    master   4m18s v1.13.5
		chadcrowell2c.mylabserver.com   Ready    none     82s   v1.13.5
		chadcrowell3c.mylabserver.com   Ready    none     69s   v1.13.5

### Building a Highly Available Kubernetes Cluster:

View the pods in the default namespace with a custom view:

	kubectl get pods -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system

View the kube-scheduler YAML:

	kubectl get endpoints kube-scheduler -n kube-system -o yaml

Create a stacked etcd topology using kubeadm:

	kubeadm init --config=kubeadm-config.yaml

Watch as pods are created in the default namespace:

	kubectl get pods -n kube-system -w

### Configuring Secure Cluster Communications:

1- View the kube-config:

	cat .kube/config | more

2- View the service account token:

	kubectl get secrets

3- Create a new namespace named my-ns:

	kubectl create ns my-ns

4- Run the kube-proxy pod in the my-ns namespace:

	kubectl run test --image=chadmcrowell/kubectl-proxy -n my-ns

5- List the pods in the my-ns namespace:

	kubectl get pods -n my-ns

6- Run a shell in the newly created pod:

	kubectl exec -it <name-of-pod> -n my-ns sh

7- List the services in the namespace via API call:

	curl localhost:8001/api/v1/namespaces/my-ns/services

8- View the token file from within a pod:

	cat /var/run/secrets/kubernetes.io/serviceaccount/token

9- List the service account resources in your cluster:

	kubectl get serviceaccounts

### Running End-to-End Tests on Your Cluster:
		
1- Run a simple nginx deployment:

	kubectl run nginx --image=nginx

2- View the deployments in your cluster:

	kubectl get deployments

3- View the pods in the cluster:

	kubectl get pods

4- Use port forwarding to access a pod directly:

	kubectl port-forward $pod_name 8081:80

5- Get a response from the nginx pod directly:

	curl --head http://127.0.0.1:8081

6- View the logs from a pod:

	kubectl logs $pod_name

7- Run a command directly from the container:

	kubectl exec -it nginx -- nginx -v

8- Create a service by exposing port 80 of the nginx deployment:

	kubectl expose deployment nginx --port 80 --type NodePort

9- List the services in your cluster:

	kubectl get services

10- Get a response from the service:

	curl -I localhost:$node_port

11- List the nodes' status:

	kubectl get nodes

12- View detailed information about the nodes:

	kubectl describe nodes

13- View detailed information about the pods:

	kubectl describe pods
	
## Managing the Kubernetes Cluster

### Upgrading the Kubernetes Cluster
The upgrade plan from v13 to v14 should go as below:
1. Upgrade the primary control plane node.
2. Upgrade additional control plane nodes.
3. Upgrade worker nodes.

#### Upgrading the Kube Master
Get the version of the API server:
	
	kubectl version --short

View the version of kubelet:

	kubectl describe nodes 

View the version of controller-manager pod:

	kubectl get po [controller_pod_name] -o yaml -n kube-system

Release the hold on versions of kubeadm and kubelet and kubectl:

	sudo apt-mark unhold kubeadm kubelet kubectl

Install version 1.14.1 of kubeadm:

	sudo apt install -y kubeadm=1.14.1-00

Hold the version of kubeadm at 1.14.1:

	sudo apt-mark hold kubeadm

Verify the version of kubeadm:

	kubeadm version

Plan the upgrade of all the controller components:

	sudo kubeadm upgrade plan

Upgrade the controller components:

	sudo kubeadm upgrade apply v1.14.1

Upgrade kubectl:

	sudo apt install -y kubectl=1.14.1-00

Upgrade the version of kubelet:

	sudo apt install -y kubelet=1.14.1-00

Hold the versions of kubelet & kubectl at 1.14.1:

	sudo apt-mark hold kubelet kubectl
	
#### Upgrading the Kube Worker Nodes
Apply this on each and every node

Release the hold on versions of kubeadm and kubelet and kubectl:

	sudo apt-mark unhold kubeadm kubelet kubectl
	
Install version 1.14.1 of kubeadm, kubelet & kubectl: (Note: You might want to install the kubectl last)

	sudo apt install -y kubeadm=1.14.1-00 kubelet=1.14.1-00 kubectl=1.14.1-00
	
Hold the versions of kubeadm, kubelet & kubectl at 1.14.1:

	sudo apt-mark hold kubeadm kubelet kubectl
	
On the Master Node, verify that all versions of the nodes have been upgraded:

	kubectl get nodes
	
		NAME                            STATUS   ROLES    AGE   VERSION
		chadcrowell1c.mylabserver.com   Ready    master   4m18s v1.14.1
		chadcrowell2c.mylabserver.com   Ready    none     82s   v1.14.1
		chadcrowell3c.mylabserver.com   Ready    none     69s   v1.14.1

## Operating System Upgrades within a Kubernetes Cluster

See which pods are running on which nodes:

	kubectl get pods -o wide

Evict the pods that are running the node that you want to upgrade: (This will move all the pods that are running on the node you want to upgrade, and place them on other nodes with available resources)

	kubectl drain [node_name] --ignore-daemonsets

Watch as the node changes status:

	kubectl get nodes -w
	
You will notice that the STATUS of the evected node will change: (In this case, this particular node is disabled from getting any new pods running on it)

	kubectl get nodes
	NAME                          STATUS                     ROLES    AGE    VERSION
	mhmdksh931c.mylabserver.com   Ready                      master   3h2m   v1.14.1
	mhmdksh932c.mylabserver.com   Ready                      <none>   34m    v1.14.1
	mhmdksh933c.mylabserver.com   Ready,SchedulingDisabled   <none>   166m   v1.14.1

Schedule pods to the node after maintenance is complete:

	kubectl uncordon [node_name]

You will notice that the STATUS changed again, and it is now ready to receive new pods

	NAME                          STATUS   ROLES    AGE    VERSION
	mhmdksh931c.mylabserver.com   Ready    master   3h5m   v1.14.1
	mhmdksh932c.mylabserver.com   Ready    <none>   36m    v1.14.1
	mhmdksh933c.mylabserver.com   Ready    <none>   168m   v1.14.1

Remove a node from the cluster:

	kubectl delete node [node_name]

Generate a new token:

	sudo kubeadm token generate

List the tokens:

	sudo kubeadm token list

Print the kubeadm join command to join a node to the cluster:

NOTE: This will only be needed when creating a new node machine, that hasn't joined the cluster yet. Unlike a machine that is already been joined and will connect to the cluster upon the next reboot.

	sudo kubeadm token create [token_name] --ttl 2h --print-join-command

## Backing Up and Restoring a Kubernetes Cluster

Get the etcd binaries:

	wget https://github.com/etcd-io/etcd/releases/download/v3.3.12/etcd-v3.3.12-linux-amd64.tar.gz

Unzip the compressed binaries:

	tar -zxvf etcd-v3.3.12-linux-amd64.tar.gz

Move the files into /usr/local/bin:

	sudo mv etcd-v3.3.12-linux-amd64/etcd* /usr/local/bin

Take a snapshot of the etcd datastore using etcdctl:

	sudo ETCDCTL_API=3 etcdctl snapshot save snapshot.db --cacert /etc/kubernetes/pki/etcd/server.crt --cert /etc/kubernetes/pki/etcd/ca.crt --key /etc/kubernetes/pki/etcd/ca.key

View the help page for etcdctl:

	ETCDCTL_API=3 etcdctl --help

Browse to the folder that contains the certificate files:

	cd /etc/kubernetes/pki/etcd/
	
Note: You'll also want to backup the certificate files and keys:

	sudo tar -zcvf etcs-certificates_$(date -I).tar.gz /etc/kubernetes/pki/etcd/

View that the snapshot was successful:

	ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshot.db

Zip up the contents of the etcd directory:

	sudo tar -zcvf etcd.tar.gz etcd

Copy the etcd directory to another server:

	scp etcd.tar.gz cloud_user@18.219.235.42:~/

## Networking (11%)

### Cluster Communications
#### Pods and Nodes Networking
See which node our pod is on:

	kubectl get pods -o wide

Log in to the node:

	ssh [node_name]

View the node's virtual network interfaces:

	ifconfig

View the containers in the pod:

	docker ps

Get the process ID for the container:

	docker inspect --format '{{ .State.Pid }}' [container_id]	
ex:

	docker inspect --format '{{ .State.Pid }}' 9e4aa2e0aa2b
		
		4349

Use nsenter to run a command in the process's network namespace:

	nsenter -t [container_pid] -n ip addr
ex:
	
	nsenter -t 4349 -n ip addr
	
		1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
		link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
		inet 127.0.0.1/8 scope host lo
		   valid_lft forever preferred_lft forever
		3: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 8951 qdisc noqueue state UP group default
		link/ether 86:0a:cc:cc:33:f9 brd ff:ff:ff:ff:ff:ff link-netnsid 0
		inet 10.244.2.9/24 scope global eth0
		   valid_lft forever preferred_lft forever
		   
You will notice that the virtual ethernet port of this particular pod is the port number 7 on the hosting node.

#### Container Network Interface (CNI)
#### Service Networking
Services allow our pods to move around, get deleted, and replicate, all without having to manually keep track of their IP addresses in the cluster. This is accomplished by creating one gateway to distribute packets evenly across all pods. In this lesson, we will see the differences between a NodePort service and a ClusterIP service and see how the iptables rules take effect when traffic is coming in.

YAML for the nginx NodePort service:

	apiVersion: v1
	kind: Service
	metadata:
	  name: nginx-nodeport
	spec:
	  type: NodePort
	  ports:
	  - protocol: TCP
	    port: 80
	    targetPort: 80
	    nodePort: 30080
	  selector:
	    app: nginx

Get the services YAML output for all the services in your cluster:

	kubectl get services -o yaml

Try and ping the clusterIP service IP address:

	ping 10.96.0.1

View the list of services in your cluster:

	kubectl get services

View the list of endpoints in your cluster that get created with a service:

	kubectl get endpoints

Look at the iptables rules for your services:

	sudo iptables-save | grep KUBE | grep nginx

#### Ingress Rules and Load Balancers		
When handling traffic from outside sources, there are two ways to direct that traffic to your pods: deploying a load balancer, and creating an ingress controller and an Ingress resource. In this lesson, we will talk about the benefits of each and how Kubernetes distributes traffic to the pods on a node to reduce latency and direct traffic to the appropriate services within your cluster.

View the list of services:

	kubectl get services

The load balancer YAML spec:

	apiVersion: v1
	kind: Service
	metadata:
	  name: nginx-loadbalancer
	spec:
	  type: LoadBalancer
	  ports:
	  - port: 80
	    targetPort: 80
	  selector:
	    app: nginx

Create a new deployment:

	kubectl run kubeserve2 --image=chadmcrowell/kubeserve2

View the list of deployments:

	kubectl get deployments

Scale the deployments to 2 replicas:

	kubectl scale deployment/kubeserve2 --replicas=2

View which pods are on which nodes:

	kubectl get pods -o wide

Create a load balancer from a deployment:

	kubectl expose deployment kubeserve2 --port 80 --target-port 8080 --type LoadBalancer

View the services in your cluster:

	kubectl get services

Watch as an external port is created for a service:

	kubectl get services -w

Look at the YAML for a service:

	kubectl get services kubeserve2 -o yaml

Curl the external IP of the load balancer:

	curl http://[external-ip]

View the annotation associated with a service:

	kubectl describe services kubeserve

Set the annotation to route load balancer traffic local to the node:

	kubectl annotate service kubeserve2 externalTrafficPolicy=Local

Igress: Unlike loadbalancers, which will act as a single point that will distibute the traffic to different nodes, the ingress rules are used as a single frontend point to access different services. It also provides different features from the loadbalancers, as it works on the application level.

The YAML for an Ingress resource:

	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	  name: service-ingress
	spec:
	  rules:
	  - host: kubeserve.example.com
	    http:
	      paths:
	      - backend:
		  serviceName: kubeserve2
		  servicePort: 80
	  - host: app.example.com
	    http:
	      paths:
	      - backend:
		  serviceName: nginx
		  servicePort: 80
	  - http:
	      paths:
	      - backend:
		  serviceName: httpd
		  servicePort: 80

Edit the ingress rules:

	kubectl edit ingress

View the existing ingress rules:

	kubectl describe ingress

Curl the hostname of your Ingress resource:

	curl http://kubeserve2.example.com

#### Cluster DNS
CoreDNS is now the new default DNS plugin for Kubernetes. In this lesson, we’ll go over the hostnames for pods and services. We will also discover how you can customize DNS to include your own nameservers.

View the CoreDNS pods in the kube-system namespace:

	kubectl get pods -n kube-system

View the CoreDNS deployment in your Kubernetes cluster:

	kubectl get deployments -n kube-system

View the service that performs load balancing for the DNS server:

	kubectl get services -n kube-system

Spec for the busybox pod:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: busybox
	  namespace: default
	spec:
	  containers:
	  - image: busybox:1.28.4
	    command:
	      - sleep
	      - "3600"
	    imagePullPolicy: IfNotPresent
	    name: busybox
	  restartPolicy: Always

View the resolv.conf file that contains the nameserver and search in DNS:

	kubectl exec -it busybox -- cat /etc/resolv.conf

Look up the DNS name for the native Kubernetes service:

	kubectl exec -it busybox -- nslookup kubernetes

Look up the DNS names of your pods:

	kubectl exec -ti busybox -- nslookup [pod-ip-address].default.pod.cluster.local

Look up a service in your Kubernetes cluster:

	kubectl exec -it busybox -- nslookup kube-dns.kube-system.svc.cluster.local

Get the logs of your CoreDNS pods:

	kubectl logs [coredns-pod-name]

YAML spec for a headless service:

	apiVersion: v1
	kind: Service
	metadata:
	  name: kube-headless
	spec:
	  clusterIP: None
	  ports:
	  - port: 80
	    targetPort: 8080
	  selector:
	    app: kubserve2

YAML spec for a custom DNS pod:

	apiVersion: v1
	kind: Pod
	metadata:
	  namespace: default
	  name: dns-example
	spec:
	  containers:
	    - name: test
	      image: nginx
	  dnsPolicy: "None"
	  dnsConfig:
	    nameservers:
	      - 8.8.8.8
	    searches:
	      - ns1.svc.cluster.local
	      - my.dns.search.suffix
	    options:
	      - name: ndots
		value: "2"
	      - name: edns0

## Pod Scheduling within the Kubernetes Cluster 
### Configuring the Kubernetes Scheduler
The default scheduler in Kubernetes attempts to find the best node for your pod by going through a series of steps. In this lesson, we will cover the steps in detail in order to better understand the scheduler’s function when placing pods on nodes to maximize uptime for the applications running in your cluster. We will also go through how to create a deployment with node affinity.

Label your node as being located in availability zone 1:

	kubectl label node chadcrowell1c.mylabserver.com availability-zone=zone1

Label your node as dedicated infrastructure:

	kubectl label node chadcrowell1c.mylabserver.com share-type=dedicated

Here is the YAML for the deployment to include the node affinity rules:

	apiVersion: extensions/v1beta1
	kind: Deployment
	metadata:
	  name: pref
	spec:
	  replicas: 5
	  template:
	    metadata:
	      labels:
		app: pref
	    spec:
	      affinity:
		nodeAffinity:
		  preferredDuringSchedulingIgnoredDuringExecution:
		  - weight: 80
		    preference:
		      matchExpressions:
		      - key: availability-zone
			operator: In
			values:
			- zone1
		  - weight: 20
		    preference:
		      matchExpressions:
		      - key: share-type
			operator: In
			values:
			- dedicated
	      containers:
	      - args:
		- sleep
		- "99999"
		image: busybox
		name: main

Create the deployment:

	kubectl create -f pref-deployment.yaml

View the deployment:

	kubectl get deployments

View which pods landed on which nodes:

	kubectl get pods -o wide
	
### Running Multiple Schedulers for Multiple Pods

In Kubernetes, you can run multiple schedulers simultaneously. You can then use different schedulers to schedule different pods. You may, for example, want to set different rules for the scheduler to run all of your pods on one node. In this lesson, I will show you how to deploy a new scheduler alongside your default scheduler and then schedule three different pods using the two schedulers.

ClusterRole.yaml

	apiVersion: rbac.authorization.k8s.io/v1beta1
	kind: ClusterRole
	metadata:
	  name: csinodes-admin
	rules:
	- apiGroups: ["storage.k8s.io"]
	  resources: ["csinodes"]
	  verbs: ["get", "watch", "list"]

ClusterRoleBinding.yaml

	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	  name: read-csinodes-global
	subjects:
	- kind: ServiceAccount
	  name: my-scheduler
	  namespace: kube-system
	roleRef:
	  kind: ClusterRole
	  name: csinodes-admin
	  apiGroup: rbac.authorization.k8s.io

Role.yaml

	apiVersion: rbac.authorization.k8s.io/v1
	kind: Role
	metadata:
	  name: system:serviceaccount:kube-system:my-scheduler
	  namespace: kube-system
	rules:
	- apiGroups:
	  - storage.k8s.io
	  resources:
	  - csinodes
	  verbs:
	  - get
	  - list
	  - watch

RoleBinding.yaml

	apiVersion: rbac.authorization.k8s.io/v1
	kind: RoleBinding
	metadata:
	  name: read-csinodes
	  namespace: kube-system
	subjects:
	- kind: User
	  name: kubernetes-admin
	  apiGroup: rbac.authorization.k8s.io
	roleRef:
	  kind: Role 
	  name: system:serviceaccount:kube-system:my-scheduler
	  apiGroup: rbac.authorization.k8s.io

Edit the existing kube-scheduler cluster role with kubectl edit clusterrole system:kube-scheduler and add the following:

	- apiGroups:
	  - ""
	  resourceNames:
	  - kube-scheduler
	  - my-scheduler
	  resources:
	  - endpoints
	  verbs:
	  - delete
	  - get
	  - patch
	  - update
	- apiGroups:
	  - storage.k8s.io
	  resources:
	  - storageclasses
	  verbs:
	  - watch
	  - list
	  - get

My-scheduler.yaml

	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  name: my-scheduler
	  namespace: kube-system
	---
	apiVersion: rbac.authorization.k8s.io/v1
	kind: ClusterRoleBinding
	metadata:
	  name: my-scheduler-as-kube-scheduler
	subjects:
	- kind: ServiceAccount
	  name: my-scheduler
	  namespace: kube-system
	roleRef:
	  kind: ClusterRole
	  name: system:kube-scheduler
	  apiGroup: rbac.authorization.k8s.io
	---
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  labels:
	    component: scheduler
	    tier: control-plane
	  name: my-scheduler
	  namespace: kube-system
	spec:
	  selector:
	    matchLabels:
	      component: scheduler
	      tier: control-plane
	  replicas: 1
	  template:
	    metadata:
	      labels:
		component: scheduler
		tier: control-plane
		version: second
	    spec:
	      serviceAccountName: my-scheduler
	      containers:
	      - command:
		- /usr/local/bin/kube-scheduler
		- --address=0.0.0.0
		- --leader-elect=false
		- --scheduler-name=my-scheduler
		image: chadmcrowell/custom-scheduler
		livenessProbe:
		  httpGet:
		    path: /healthz
		    port: 10251
		  initialDelaySeconds: 15
		name: kube-second-scheduler
		readinessProbe:
		  httpGet:
		    path: /healthz
		    port: 10251
		resources:
		  requests:
		    cpu: '0.1'
		securityContext:
		  privileged: false
		volumeMounts: []
	      hostNetwork: false
	      hostPID: false
	      volumes: []

Run the deployment for my-scheduler:

	kubectl create -f my-scheduler.yaml

View your new scheduler in the kube-system namespace:

	kubectl get pods -n kube-system

pod1.yaml

	apiVersion: v1
	kind: Pod
	metadata:
	  name: no-annotation
	  labels:
	    name: multischeduler-example
	spec:
	  containers:
	  - name: pod-with-no-annotation-container
	    image: k8s.gcr.io/pause:2.0

pod2.yaml

	apiVersion: v1
	kind: Pod
	metadata:
	  name: annotation-default-scheduler
	  labels:
	    name: multischeduler-example
	spec:
	  schedulerName: default-scheduler
	  containers:
	  - name: pod-with-default-annotation-container
	    image: k8s.gcr.io/pause:2.0

pod3.yaml

	apiVersion: v1
	kind: Pod
	metadata:
	  name: annotation-second-scheduler
	  labels:
	    name: multischeduler-example
	spec:
	  schedulerName: my-scheduler
	  containers:
	  - name: pod-with-second-annotation-container
	    image: k8s.gcr.io/pause:2.0

View the pods as they are created:

	kubectl get pods -o wide
	
### Scheduling Pods with Resource Limits and Label Selectors

In order to share the resources of your node properly, you can set resource limits and requests in Kubernetes. This allows you to reserve enough CPU and memory for your application while still maintaining system health. In this lesson, we will create some requests and limits in our pod YAML to show how it’s used by the node.

View the capacity and the allocatable info from a node:

	kubectl describe nodes

The pod YAML for a pod with requests:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: resource-pod1
	spec:
	  nodeSelector:
	    kubernetes.io/hostname: "chadcrowell3c.mylabserver.com"
	  containers:
	  - image: busybox
	    command: ["dd", "if=/dev/zero", "of=/dev/null"]
	    name: pod1
	    resources:
	      requests:
		cpu: 800m
		memory: 20Mi

Create the requests pod:

	kubectl create -f resource-pod1.yaml

View the pods and nodes they landed on:

	kubectl get pods -o wide

The YAML for a pod that has a large request:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: resource-pod2
	spec:
	  nodeSelector:
	    kubernetes.io/hostname: "chadcrowell3c.mylabserver.com"
	  containers:
	  - image: busybox
	    command: ["dd", "if=/dev/zero", "of=/dev/null"]
	    name: pod2
	    resources:
	      requests:
		cpu: 1000m
		memory: 20Mi

Create the pod with 1000 millicore request:

	kubectl create -f resource-pod2.yaml

See why the pod with a large request didn’t get scheduled:

	kubectl describe resource-pod2

Look at the total requests per node:

	kubectl describe nodes chadcrowell3c.mylabserver.com

Delete the first pod to make room for the pod with a large request:

	kubectl delete pods resource-pod1

Watch as the first pod is terminated and the second pod is started:

	kubectl get pods -o wide -w

The YAML for a pod that has limits:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: limited-pod
	spec:
	  containers:
	  - image: busybox
	    command: ["dd", "if=/dev/zero", "of=/dev/null"]
	    name: main
	    resources:
	      limits:
		cpu: 1
		memory: 20Mi

Create a pod with limits:

	kubectl create -f limited-pod.yaml

Use the exec utility to use the top command:

	kubectl exec -it limited-pod top	

### DaemonSets and Manually Scheduled Pods

DaemonSets do not use a scheduler to deploy pods. In fact, there are currently DaemonSets in the Kubernetes cluster that we made. In this lesson, I will show you where to find those and how to create your own DaemonSet pods to deploy without the need for a scheduler.

Find the DaemonSet pods that exist in your kubeadm cluster:

	kubectl get pods -n kube-system -o wide

Delete a DaemonSet pod and see what happens:

	kubectl delete pods [pod_name] -n kube-system

Give the node a label to signify it has SSD:

	kubectl label node[node_name] disk=ssd

The YAML for a DaemonSet:

	apiVersion: apps/v1beta2
	kind: DaemonSet
	metadata:
	  name: ssd-monitor
	spec:
	  selector:
	    matchLabels:
	      app: ssd-monitor
	  template:
	    metadata:
	      labels:
		app: ssd-monitor
	    spec:
	      nodeSelector:
		disk: ssd
	      containers:
	      - name: main
		image: linuxacademycontent/ssd-monitor

Create a DaemonSet from a YAML spec:

	kubectl create -f ssd-monitor.yaml

Label another node to specify it has SSD:

	kubectl label node chadcrowell2c.mylabserver.com disk=ssd

View the DaemonSet pods that have been deployed:

	kubectl get pods -o wide

Remove the label from a node and watch the DaemonSet pod terminate:

	kubectl label node chadcrowell3c.mylabserver.com disk-

Change the label on a node to change it to spinning disk:

	kubectl label node chadcrowell2c.mylabserver.com disk=hdd --overwrite

Pick the label to choose for your DaemonSet:

	kubectl get nodes chadcrowell3c.mylabserver.com --show-labels

### Displaying Scheduler Events

There are multiple ways to view the events related to the scheduler. In this lesson, we’ll look at ways in which you can troubleshoot any problems with your scheduler or just find out more information.

View the name of the scheduler pod:

	kubectl get pods -n kube-system

Get the information about your scheduler pod events:

	kubectl describe pods [scheduler_pod_name] -n kube-system

View the events in your default namespace:

	kubectl get events

View the events in your kube-system namespace:

	kubectl get events -n kube-system

Delete all the pods in your default namespace:

	kubectl delete pods --all

Watch events as they are appearing in real time:

	kubectl get events -w

View the logs from the scheduler pod:

	kubectl logs [kube_scheduler_pod_name] -n kube-system

The location of a systemd service scheduler pod:

	/var/log/kube-scheduler.log
