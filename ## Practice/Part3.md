# Objectives

You have been given access to a three-node cluster. You will be responsible for creating a deployment and a service to serve as a front end for a web application. In addition to the web application, you must deploy a Redis database and make sure the web application can only access this database using the default port of 6379. You will first create a default-deny network policy, so all pods within your Kubernetes are not able to communicate with each other by default. Then you will create a second network policy that specifies the communication on port 6379 between the web application and the database using their label selectors. You must apply these specifications to your resources in order to complete this hands-on lab:

* Create a deployment named webfront-deploy.
* The deployment should use the image nginx with the tag 1.7.8.
* The deployment should expose container port 80 on each pod and contain 2 replicas.
* Create a service named webfront-service and expose port 80, target port 80.
* The service should be exposed externally by listening on port 30080 on each node.
* Create one pod named db-redis using the image redis and the tag latest.
* Verify that you can communicate to pods by default.
* Create a network policy named default-deny that will deny pod communication by default.
* Verify that you can no longer communicate between pods.
* Apply the label role=frontend to the web application pods and the label role=db to the database pod.
* Create a network policy that will apply an ingress rule for the pods labeled with role=db to allow traffic on port 6379 from the pods labeled role=frontend.
* Verify that you have applied the correct labels and created the correct network policies.

