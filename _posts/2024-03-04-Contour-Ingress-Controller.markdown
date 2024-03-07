---
layout: post
title:  "K8S - Contour Ingress Controller"
date:   2024-03-04 12:13:26 +0000
tags: [K8S]
---

## Background 

Contour is an open source Kubernetes ingress controller that works by deploying the Envoy proxy as a reverse proxy and load balancer.

## Install on Amazon EKS

```
helm upgrade contour-ingress bitnami/contour   --namespace projectcontour  -f values.yaml
```

Values.yaml file :

I had to switch off HostPort as it was causing conflocts. 

```
envoy:
  useHostPort: false
  service:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-internal: "false"
      service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
      service.beta.kubernetes.io/aws-load-balancer-type: "external"
```

Check your deployment :

```
kubectl get all --namespace projectcontour
```

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


## Test your ingress

Simple test application :

```
kubectl apply -f https://projectcontour.io/examples/kuard.yaml
```

Get Loadblancer dns name and test your application

```
kubectl get po,svc,ing -l app=kuard
```

Remove when tested
```
kubectl delete -f https://projectcontour.io/examples/kuard.yaml
```

# Example app to test the ingress

Modified example based on the follwoing blog , we use the yellow and green service to test our rules: 
https://aws.amazon.com/blogs/containers/how-to-expose-multiple-applications-on-amazon-eks-using-a-single-application-load-balancer/

## 1. Application and Docker Image Creation Process

### Create Directories
```bash
mkdir green/ yellow/
```
### Copy Application Code
Green Application
```
echo '<html style="background-color: green;"></html>' > green/index.html
```
Yellow Application
```
echo '<html style="background-color: yellow;"></html>' > yellow/index.html
```
### Dockerfile Creation

Green Application Dockerfile
```
FROM public.ecr.aws/nginx/nginx:1.20-alpine
RUN mkdir -p /usr/share/nginx/html/green
COPY ./index.html /usr/share/nginx/html/green/index.html 
EXPOSE 80
```
Yellow Application Dockerfile
```
FROM nginx:alpine
RUN mkdir -p /usr/share/nginx/html/yellow
COPY ./index.html /usr/share/nginx/html/yellow/index.html 
EXPOSE 80
```
### Amazon ECR Repository Creation
```
aws ecr create-repository --repository-name green
aws ecr create-repository --repository-name yellow
```
### Push to ECR

```
export AWS_REGION=$(aws ec2 describe-availability-zones --output text --query 'AvailabilityZones[0].[RegionName]')
export AWS_REGISTRY_ID=$(aws ecr describe-registry --query registryId --output text)
export AWS_ECR_REPO=${AWS_REGISTRY_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ECR_REPO

cd yellow/
docker build . -t yellow
docker tag yellow:latest $AWS_ECR_REPO/yellow:latest
docker push $AWS_ECR_REPO/yellow:latest
cd ..

cd green/
docker build . -t green
docker tag green:latest $AWS_ECR_REPO/green:latest
docker push $AWS_ECR_REPO/green:latest
cd ..
```
### Deploy your application to K8S Cluster 

**kubectl apply -f app-deployment.yaml**

```
apiVersion: v1
kind: Namespace
metadata:
  name: color-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: green-app
  namespace: color-app
  labels:
    app: green-app
spec:
  selector:
    matchLabels:
      app: green-app
  replicas: 2
  template:
    metadata:
      labels:
        app: green-app
    spec:
      containers:
      - name: green-container
        image: 788148918336.dkr.ecr.ap-east-1.amazonaws.com/green:latest
        ports:
          - containerPort: 80
        resources:
          limits:
            memory: "100Mi"
            cpu: "200m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yellow-app
  namespace: color-app
  labels:
    app: yellow-app
spec:
  selector:
    matchLabels:
      app: yellow-app
  replicas: 2
  template:
    metadata:
      labels:
        app: yellow-app
    spec:
      containers:
      - name: yellow-container
        image: 788148918336.dkr.ecr.ap-east-1.amazonaws.com/yellow:latest
        ports:
          - containerPort: 80
        resources:
          limits:
            memory: "100Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  namespace: color-app
  name: green-service
  labels:
    app: green-app
spec:
  type: NodePort
  selector:
    app: green-app
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: color-app
  name: yellow-service
  labels:
    app: yellow-app
spec:
  type: NodePort
  selector:
    app: yellow-app
  ports:
    - port: 80
      targetPort: 80
```

### Create your Ingress 

**kubectl apply -f nginx-ingress.yaml**

```
apiVersion: v1
kind: Namespace
metadata:
  name: color-app
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  labels:
    app: shared-app
spec:
  rules:
    - host: k8s-projectc-contouri-957094c67c-936aee38a4c347de.elb.ap-east-1.amazonaws.com
      http:
        paths:
          - path: /yellow
            pathType: Prefix
            backend:
              service:
                name: yellow-service
                port:
                  number: 80
          - path: /green
            pathType: Prefix
            backend:
              service:
                name: green-service
                port:
                  number: 80
```



## Configure your ingress for your Microservice 

Create your ingress file app-ingress.yaml
kubectl apply -f test-ingress.yaml -n color-app

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  labels:
    app: shared-app
spec:
  rules:
    - host: k8s-projectc-contouri-957094c67c-936aee38a4c347de.elb.ap-east-1.amazonaws.com
      http:
        paths:
          - path: /yellow
            pathType: Prefix
            backend:
              service:
                name: yellow-service
                port:
                  number: 80
          - path: /green
            pathType: Prefix
            backend:
              service:
                name: green-service
                port:
                  number: 80
          # Add more paths for other apps
```


## Toubleshooting 

Usefull commands : 

```
kubectl -n projectcontour get po,svc
kubectl get all --namespace projectcontour
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

# Running Contour in tandem with another ingress controller
If you’re running multiple ingress controllers, or running on a cloudprovider that natively handles ingress, you can specify the annotation kubernetes.io/ingress.class: "contour" on all ingresses that you would like Contour to claim. You can customize the class name with the --ingress-class-name flag at runtime. If the kubernetes.io/ingress.class annotation is present with a value other than "contour", Contour will ignore that ingress.



## References

- [Contour Getting Started](https://projectcontour.io/getting-started/#option-2-helm)
- [Contour Deployment Options](https://projectcontour.io/docs/v1.0.1/deploy-options/)
- [Deploying Contour with AWS NLB](https://projectcontour.io/guides/deploy-aws-nlb/)
- [Testing Your Contour Installation](https://projectcontour.io/docs/1.28/deploy-options/#testing-your-installation)
- [How to Expose Multiple Applications on Amazon EKS using a Single Application Load Balancer](https://aws.amazon.com/blogs/containers/how-to-expose-multiple-applications-on-amazon-eks-using-a-single-application-load-balancer/)
