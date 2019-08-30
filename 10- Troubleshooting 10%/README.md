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
