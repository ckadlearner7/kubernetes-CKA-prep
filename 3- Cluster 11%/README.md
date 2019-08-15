## Managing the Kubernetes Cluster 11%

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

### Operating System Upgrades within a Kubernetes Cluster

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

### Backing Up and Restoring a Kubernetes Cluster

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