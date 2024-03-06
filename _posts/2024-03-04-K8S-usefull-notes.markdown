---
layout: post
title:  "K8S-usefull-notes"
date:   2024-03-05 12:13:26 +0000
tags: [K8S]
---

## Basic Comamnds

## Contour ingress Controller install

This is the only way I could get the ingress controller with an external facing Loadbalancer and no-host ports. Will revist and try to use helm instead.

Download https://projectcontour.io/quickstart/contour.yaml , save localy edit the following config. 

```
Envoy hostport 
  ports:
        - containerPort: 8080
          hostPort: null  # Set hostPort to null to disable
          name: http
          protocol: TCP
        - containerPort: 8443
          hostPort: null  # Set hostPort to null to disable
          name: https
          protocol: TCP

```

Then run the follwoing command : 

```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml -n projectcontour
kubectl annotate svc envoy -n projectcontour --overwrite service.beta.kubernetes.io/aws-load-balancer-internal=false
```

## Toubleshooting 

Check the status of services and pods 

This error was fixed by turrning off Hostports

4m56s       Warning   FailedScheduling   pod/my-release-contour-envoy-v485g   0/3 nodes are available: 1 node(s) didn't have free ports for the requested pod ports. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.

Usefull commands :

```
kubectl -n projectcontour get po,svc
kubectl logs my-release-contour-envoy-lc4l7 -n projectcontour  # replace the pod name 
kubectl -n projectcontour describe pod my-release-contour-envoy-lc4l7
kubectl get nodes --all-namespaces -o wide
kubectl get pods --all-namespaces -o wide
kubectl get services --all-namespaces -o wide
kubectl logs -n projectcontour my-release-contour-envoy-5228s
kubectl get events -n projectcontour
kubectl get po --all-namespaces -o=jsonpath="{range .items[*]}{.spec.nodeName}{'\t'}{.spec.hostNetwork}{'\t'}{.metadata.namespace}{'\t'}{.metadata.name}{'\t'}{.spec.hostNetwork}{'\t'}{.spec.containers..containerPort}{'\n'}{end}"
kubectl get ingress --all-namespaces
kubectl delete pod my-pod -n my-namespace

```

## Deploy a test ingress application

# Running Contour in tandem with another ingress controller
If you’re running multiple ingress controllers, or running on a cloudprovider that natively handles ingress, you can specify the annotation kubernetes.io/ingress.class: "contour" on all ingresses that you would like Contour to claim. You can customize the class name with the --ingress-class-name flag at runtime. If the kubernetes.io/ingress.class annotation is present with a value other than "contour", Contour will ignore that ingress.

# Example app to test the ingress
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
``

kubectl apply -f nginx-deployment.yaml


## Ingress 

``
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: projectcontour
  annotations:
    kubernetes.io/ingress.class: "contour"
spec:
  rules:
  - host: nginx.example.com  # Update with your desired host
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80
```

kubectl apply -f nginx-ingress.yaml
43.198.25.162:31270


## Error

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install kubeapps --namespace kubeapps bitnami/kubeapps \
    --set global.imageRegistry=217273820646.dkr.ecr.us-east-1.amazonaws.com/7762bbba-0563-4863-bb12-2b1dfb26d3c6/cg-1813167126 \
    --set apprepository.image.repository=kubeapps-apprepository-controller \
    --set apprepository.image.tag=1.0.0-0-latest \
    --set apprepository.syncImage.repository=kubeapps-chart-repo \
    --set apprepository.syncImage.tag=1.0.0-0-latest \
    --set chartsvc.image.repository=kubeapps-chartsvc \
    --set chartsvc.image.tag=1.0.0-0-latest \
    --set dashboard.image.repository=kubeapps-dashboard \
    --set dashboard.image.tag=1.0.0-0-latest \
    --set hooks.image.repository=kubectl \
    --set hooks.image.tag=1.0.0-0-latest \
    --set frontend.image.repository=nginx \
    --set frontend.image.tag=1.0.0-0-latest \
    --set postgresql.image.repository=postgresql \
    --set postgresql.image.tag=1.0.0-0-latest
```


## References 

https://projectcontour.io/docs/v1.0.1/deploy-options/
