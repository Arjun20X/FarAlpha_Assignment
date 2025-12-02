Flask + MongoDB Kubernetes Deployment (FarAlpha Assignment)

This project deploys a Python Flask application connected to an authenticated MongoDB instance on a Kubernetes cluster running on Minikube 

It includes:

Docker image builds

Kubernetes Deployments

Kubernetes Services

MongoDB StatefulSet

Secrets

Persistent Volumes

Horizontal Pod Autoscaling

DNS-based service discovery

Resource Requests & Limits

## 1. Architecture Overview
Flask Application

/ → returns current timestamp

/data → GET: retrieve documents

/data → POST: insert documents

Deployed as a Deployment with 2 replicas (autoscalable to 5)

Connects to MongoDB using internal Kubernetes DNS

MongoDB

Runs as a StatefulSet

Has authentication (root user + password from Secret)

Persistent storage via PV/PVC (5Gi)

## 2. Prerequisites

Install these on Windows:

Tool	Status
Python 3.8+	✔
Docker Desktop	✔
Minikube	✔
kubectl	✔
Metrics Server	must enable

Enable metrics server (Windows PowerShell):

minikube addons enable metrics-server

## 3. Build Docker Image (Windows PowerShell Version)
Option A — Build inside Minikube (RECOMMENDED)

This avoids DockerHub completely.

minikube -p minikube docker-env | Invoke-Expression
docker build -t flask-mongo-app:1.0 .

Option B — Using DockerHub
docker build -t <your-dockerhub-username>/flask-mongo-app:1.0 .
docker push <your-dockerhub-username>/flask-mongo-app:1.0

## 4. Kubernetes Deployment (Windows Commands)

All commands must be executed from project root:

flask-mongodb-app/

1) Create Namespace
kubectl apply -f k8s/namespace.yaml

2) Create MongoDB Secret
kubectl apply -f k8s/mongo-secret.yaml

3) Create Persistent Volume & Claim
kubectl apply -f k8s/mongo-pv-pvc.yaml

4) Deploy MongoDB
kubectl apply -f k8s/mongo-service.yaml
kubectl apply -f k8s/mongo-statefulset.yaml


Wait for pod:

kubectl get pods -n fa-assignment

5) Deploy Flask Application
kubectl apply -f k8s/flask-deployment.yaml
kubectl apply -f k8s/flask-service.yaml

6) Deploy Horizontal Pod Autoscaler
kubectl apply -f k8s/hpa.yaml

## 5. Access Flask Application

Expose service using Minikube:

minikube service -n fa-assignment flask-service --url


Example output:

http://127.0.0.1:55275


Open in browser.

## 6. Test Autoscaling (Windows Version)
Start load generator (IMPORTANT: Windows-compatible)
kubectl run -it --rm load --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://flask-service.fa-assignment.svc.cluster.local:5000/; done"

Watch HPA
kubectl get hpa -n fa-assignment -w

Watch pods scale
kubectl get pods -n fa-assignment -w


Expected:

CPU rises

Replicas scale from 2 → up to 5

## 7. DNS Resolution in Kubernetes

Kubernetes CoreDNS provides internal DNS names for services.

MongoDB DNS:

mongo.fa-assignment.svc.cluster.local


Flask connects via environment variables:

MONGO_HOST = mongo.fa-assignment.svc.cluster.local
MONGO_PORT = 27017


This avoids IP-based connections and ensures reliability.

## 8. Resource Requests & Limits
Pod resource configuration:
requests:
  cpu: "0.2"
  memory: "250Mi"
limits:
  cpu: "0.5"
  memory: "500Mi"

Why?

Requests = guaranteed resources (scheduling)

Limits = maximum allowed usage

Prevents noisy-neighbor issues

## 9. Project Structure
flask-mongodb-app/
│
├── app.py
├── Dockerfile
├── requirements.txt
├── README.md
│
└── k8s/
    ├── namespace.yaml
    ├── mongo-secret.yaml
    ├── mongo-pv-pvc.yaml
    ├── mongo-statefulset.yaml
    ├── mongo-service.yaml
    ├── flask-deployment.yaml
    ├── flask-service.yaml
    ├── hpa.yaml

## 10. Design Choices Summary

StatefulSet for MongoDB → stable identity + persistent storage

ClusterIP for MongoDB → internal-only database

NodePort for Flask → external access via Minikube

HPA → automatic scaling under load

PV/PVC → persistent data for MongoDB

Secrets → secure credentials

Internal DNS → reliable service-to-service communication