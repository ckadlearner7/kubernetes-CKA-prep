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