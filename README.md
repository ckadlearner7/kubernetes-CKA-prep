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
