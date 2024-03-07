---
layout: post
title:  "K8S - Contour Ingress Controller"
date:   2024-03-04 12:13:26 +0000
tags: [K8S]
---

## Background 

Contour is an open source Kubernetes ingress controller that works by deploying the Envoy proxy as a reverse proxy and load balancer.

## Contour Service (ClusterIP):

- **Type:** ClusterIP
- **ClusterIP:** 10.100.238.213
- **External-IP:** `<none>` (This is an internal ClusterIP service, not exposed externally.)
- **Ports:** 8001/TCP
- **Purpose:** The `contour` service provides access to the Contour admin interface. It is a ClusterIP service, meaning it is only accessible from within the cluster.

## Envoy Service (LoadBalancer):

- **Type:** LoadBalancer
- **ClusterIP:** 10.100.201.252
- **External-IP:** k8s-projectc-envoy-0e42196471-b91bf5fcd4308ca2.elb.ap-east-1.amazonaws.com
- **Ports:** 80:30693/TCP, 443:31732/TCP
- **Purpose:** The `envoy` service is the external-facing service that exposes the Envoy proxy to the outside world. It is of type LoadBalancer, and in your case, it has an external IP provided by AWS Elastic Load Balancer (ELB). It is accessible externally on ports 80 (HTTP) and 443 (HTTPS).

The separation of `contour` (ClusterIP) and `envoy` (LoadBalancer) services is a common architecture in Ingress controllers like Contour. The `contour` service is used for administrative purposes, while the `envoy` service serves as the entry point for incoming traffic, routing it to the appropriate backend services based on the Ingress rules.


## Basic Comamnds

```
kubectl get namespaces
kubectl get all --namespace projectcontour
kubectl logs -n projectcontour -c envoy envoy-gwjlh
kubectl describe pod envoy-gwjlh -n projectcontour
kubectl describe svc envoy -n projectcontour
kubectl get svc envoy -n projectcontour -o yaml
```

## Contour ingress Controller install

I had to switch off hostport and it was causing conflocts 

helm install contour-ingress bitnami/contour \
  --namespace projectcontour \
  --create-namespace \
  --set service.type=LoadBalancer \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-internal"="false" \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing" \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="external" \
  --set envoy.useHostPort=false


  helm upgrade contour-ingress bitnami/contour \
  --namespace projectcontour \
  --set service.type=LoadBalancer \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-internal"=false \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing" \
  --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="external" \
  --set envoy.useHostPort=false



helm install contour-ingress bitnami/contour --namespace projectcontour --create-namespace --set service.type=LoadBalancer --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-internal"=false --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing" --set awsLoadBalancer.subnets[0]=subnet-050e58abe7dda67d4 --set awsLoadBalancer.subnets[1]=subnet-0cc832eeb9b1a4474 --set envoy.useHostPort=false


helm repo add bitnami https://charts.bitnami.com/bitnami
helm install contour-ingress --set envoy.useHostPort=false --set contour.envoy.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-internal"=false bitnami/contour --namespace projectcontour --create-namespace 


helm install my-release bitnami/contour   --namespace projectcontour   --create-namespace   --set service.type=LoadBalancer   --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-internal"=false   --set service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"="internet-facing"   --set awsLoadBalancer.subnets[0]=subnet-050e58abe7dda67d4   --set awsLoadBalancer.subnets[1]=subnet-0cc832eeb9b1a4474


This is te only way i managed to install the ingress controller with an external facing Loadbalancer

```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml -n projectcontour
kubectl annotate svc envoy -n projectcontour --overwrite service.beta.kubernetes.io/aws-load-balancer-internal=false
```


```
kubectl apply -f local-contour1.yaml -n projectcontour
kubectl annotate svc envoy -n projectcontour --overwrite service.beta.kubernetes.io/aws-load-balancer-internal=false
```


## Toubleshooting 

Check the status of services and pods 

You may bet teh follwoing error :

4m56s       Warning   FailedScheduling   pod/my-release-contour-envoy-v485g   0/3 nodes are available: 1 node(s) didn't have free ports for the requested pod ports. preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.


```
kubectl -n projectcontour get po,svc
kubectl logs my-release-contour-envoy-lc4l7 -n projectcontour  # replace the pod name 
kubectl -n projectcontour logs contour-ingress-envoy-d8mbr -c envoy
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

  --set envoy.readinessProbe.successThreshold=5 
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


## References 

https://projectcontour.io/getting-started/#option-2-helm
https://projectcontour.io/docs/v1.0.1/deploy-options/
