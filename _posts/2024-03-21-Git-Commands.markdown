---
layout: post
title:  "K8S Commands"
date:   2024-03-21 12:13:26 +0000
tags: [AWS,ECS]
---

### Usefull commands

**Switch Context**

```
kubectl config get-contexts
aws eks --region us-west-2 update-kubeconfig --name my-eks-cluster
aws eks --region ap-east-1 update-kubeconfig --name infosys-hktd-cluster-test-cluster
kubectl config use-context infosys-hktd-cluster-test-cluster
```


**Check Cluster permissions**
kubectl describe configmap aws-auth -n kube-system
kubectl describe rolebinding 

kubectl edit -n kube-system configmap/aws-auth

kubectl config view --minify

dashboard-dev-grafana