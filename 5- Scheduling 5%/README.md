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
