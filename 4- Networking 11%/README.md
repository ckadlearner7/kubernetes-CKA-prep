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