# Horizontal Pod Autoscaling (HPA) Using Metrics Server in Kubernetes

## Overview

Horizontal Pod Autoscaler (HPA) automatically scales the number of pod replicas in a Kubernetes Deployment based on observed resource usage, such as CPU or memory. This document explains how to set up HPA using the Metrics Server, deploy a sample application, generate load, and observe scaling behavior.

---

## Prerequisites

* A running Kubernetes cluster
* `kubectl` configured with cluster access
* Cluster-admin or sufficient privileges to install system components

---

## Metrics Server Requirement for HPA

> **Important:** HPA depends on the Metrics Server. Kubernetes does not collect CPU and memory usage metrics by default.

Without Metrics Server:

* `kubectl top` commands will not work
* HPA will not function

---

## Installing Metrics Server

### Official Repository

* GitHub: [https://github.com/kubernetes-sigs/metrics-server](https://github.com/kubernetes-sigs/metrics-server)

### Installation Command

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Required Configuration Change

In some environments (especially self-managed clusters), kubelets use **self-signed TLS certificates**. Metrics Server will fail to collect metrics unless explicitly allowed to trust these certificates.

Add the following argument to the `metrics-server` Deployment under `args`:

```yaml
- --kubelet-insecure-tls
```

### Why This Flag Is Required

* Metrics Server communicates with kubelets on worker nodes to fetch CPU and memory metrics
* Kubelets typically present self-signed TLS certificates
* Metrics Server does not trust these certificates by default
* The `--kubelet-insecure-tls` flag instructs Metrics Server to accept self-signed certificates

---

## Verifying Metrics Server Installation

### Check Metrics Server Pod

```bash
kubectl get pods -n kube-system
```

Example output:

```text
metrics-server-59d465df9f-vt22r   1/1   Running   0   3m45s
```

### Verify Metrics API Availability

```bash
kubectl get apiservice
```

Expected output:

```text
v1beta1.metrics.k8s.io   kube-system/metrics-server   True   4m24s
```

---

## Deploying a Sample Application

### Deployment and Service Manifest

```yaml
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
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

This configuration:

* Deploys an NGINX application with CPU requests and limits
* Exposes the application using a NodePort service

---

## Configuring Horizontal Pod Autoscaler

### HPA Manifest

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 60

  
```

### HPA Behavior

* Scales pods when average CPU utilization exceeds **50%** and Memory exceeds **60%**
* Maintains a minimum of **1 replica**
* Scales up to a maximum of **5 replicas**

---

## Monitoring HPA and Metrics

Use the following commands to observe scaling behavior:

```bash
kubectl get hpa
kubectl describe hpa hpa-demo
kubectl top pods
kubectl get pods -w
```

---

## Generating Load

### Load Generator Pod Manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      while true; do
        wget -q -O- http://nginx-service;
        wget -q -O- http://nginx-service;
        wget -q -O- http://nginx-service;
      done
```

This pod continuously sends HTTP requests to the NGINX service, increasing CPU utilization.

---

## Observing Autoscaling Behavior

* When the `load-generator` pod is running:

  * CPU usage of NGINX pods increases
  * HPA automatically scales up the number of replicas

* After deleting the `load-generator` pod:

  * CPU utilization decreases
  * HPA gradually scales down the replicas
  * Scale-down typically takes **around 5 minutes**, as per Kubernetes default stabilization behavior

---


### 📌 Important Note on Deployment Replicas vs HPA Control

Once a Horizontal Pod Autoscaler (HPA) is applied to a Deployment, **HPA becomes the authoritative controller for the replica count**.

* If a Deployment is initially created with `replicas: 2` and the associated HPA is configured with `minReplicas: 1`, the HPA **can scale the Deployment down to 1 pod** based on observed metrics.
* After HPA is attached, the `replicas` value defined in the Deployment manifest is **no longer enforced continuously**.
* The Deployment’s replica count is primarily used **only at creation or update time**.
* From that point onward, Kubernetes adjusts the replica count **exclusively according to HPA rules** (`minReplicas`, `maxReplicas`, and metrics).

**In summary:**

> When HPA is enabled, it overrides the Deployment’s replica setting and manages scaling decisions. The Deployment’s replica count acts as an initial value, while HPA controls the runtime scaling behavior.

---


## Summary

* Metrics Server is mandatory for HPA to function
* Self-signed kubelet certificates require `--kubelet-insecure-tls` in many environments
* HPA dynamically scales pods based on real-time resource utilization
* Scale-up is fast, while scale-down is intentionally delayed to ensure stability

---


