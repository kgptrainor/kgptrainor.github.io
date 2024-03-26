---
layout: post
title:  "Docker-Gunicorn"
date:   2024-03-21 12:13:26 +0000
tags: [AWS,ECS]
---

### Issue/Problem 

We are using the following base image for our ECS Containers and are seeing very high RAM and CPU usage (Causing task crashes).

Image : tiangolo/uvicorn-gunicorn-fastapi:python3.11

### Soultion 

We should not be using tiangolo/uvicorn-gunicorn-fastapi:python3.11 as the base image for our docker container. We need to build our own image, See the following explanation. 

***Reference taken from the from the fastapi.tiangolo.com offical site:*** 
https://fastapi.tiangolo.com/deployment/docker/

```
One Process per Container
In this type of scenario, you probably would want to have a single (Uvicorn) process per container, as you would already be handling replication at the cluster level.

So, in this case, you would not want to have a process manager like Gunicorn with Uvicorn workers, or Uvicorn using its own Uvicorn workers. You would want to have just a single Uvicorn process per container (but probably multiple containers).

Having another process manager inside the container (as would be with Gunicorn or Uvicorn managing Uvicorn workers) would only add unnecessary complexity that you are most probably already taking care of with your cluster system.
```

```
When to Use
You should probably not use this official base image (or any other similar one) if you are using Kubernetes (or others) and you are already setting replication at the cluster level, with multiple containers. In those cases, you are better off building an image from scratch as described above: Build a Docker Image for FastAPI.

This image would be useful mainly in the special cases described above in Containers with Multiple Processes and Special Cases. For example, if your application is simple enough that setting a default number of processes based on the CPU works well, you don't want to bother with manually configuring the replication at the cluster level, and you are not running more than one container with your app. Or if you are deploying with Docker Compose, running on a single server, etc.*
```

### Build a Docker Image for FastAPI


requirements.txt:

fastapi>=0.68.0,<0.69.0
pydantic>=1.8.0,<2.0.0
uvicorn>=0.15.0,<0.16.0


### Dockerfile
```
FROM python:3.9
WORKDIR /code
COPY ./requirements.txt /code/requirements.txt
RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt
COPY ./app /code/app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```


Example :
```
# Fixing release version to 2022-11-10 
# we found an issues with the later releases which requires h11==0.14 (we use h11==0.12)
# h11 can be upgraded when unicorn updates its dependencies
FROM python:3.9

WORKDIR /code

COPY ./requirements.txt /requirements.txt

RUN pip3 install --no-cache-dir --upgrade -r /requirements.txt

# Install additional dependencies for Opentelementry
RUN opentelemetry-bootstrap --action=install

COPY ./app /code/app

CMD ["opentelemetry-instrument", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```


Reference :

[https://fastapi.tiangolo.com/deployment/docker/](https://fastapi.tiangolo.com/deployment/docker/)  
[https://fastapi.tiangolo.com/deployment/docker/#build-a-docker-image-for-fastapi](https://fastapi.tiangolo.com/deployment/docker/#build-a-docker-image-for-fastapi)



## Running on fargate 
AWS Fargate launches each of the containers that you want to run into it’s own isolated micro VM. And that micro VM is actually sized specifically for the needs of that container. So you see here, rather than having large EC2 instances that have a larger boundary and multiple applications within that boundary, each of these containers has it’s own micro VM that is sized perfectly for that container.