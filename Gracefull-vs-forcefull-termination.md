# Hands-On: Graceful vs Forceful Termination in Kubernetes

---

## **Terminals Setup**

| Terminal       | Purpose                                               |
| -------------- | ----------------------------------------------------- |
| **Terminal-1** | Run kubectl commands (create/delete pods, watch pods) |
| **Terminal-2** | Watch pod logs (`kubectl logs -f`)                    |
| **Terminal-3** | Simulate user requests (`kubectl exec -it`)           |

---

## **Step 1: Run the Pod**

**Terminal-1**:

```bash
kubectl run graceful-web --image=busybox --restart=Never --command -- sh -c "while true; do echo -e 'HTTP/1.1 200 OK\n\nHello World' | nc -l -p 8080; done"
kubectl get pods -w
```

**Expected output**:

```
NAME           READY   STATUS    RESTARTS   AGE
graceful-web   1/1     Running   0          5s
```

---

## **Step 2: Watch Logs**

**Terminal-2**:

```bash
kubectl logs -f graceful-web
```

*No logs appear yet because no requests have been sent.*

---

## **Step 3: Send a Request**

**Terminal-3**:

```bash
kubectl exec -it graceful-web -- sh -c "echo 'Hello' | nc localhost 8080"
```

**Expected output**:

```
HTTP/1.1 200 OK

Hello World
```

*Terminal-2 now shows request handling logs.*

---

# **Scenario 1: Graceful Termination**

## **Step 4: Delete Pod Gracefully**

**Terminal-1**:

```bash
kubectl delete pod graceful-web
```

**Expected output**:

```
pod "graceful-web" deleted
```

*Pod enters `Terminating` state with a default grace period of 30 seconds.*

---

## **Step 5: Send a Request During Grace Period**

**Terminal-3** (within 30 seconds):

```bash
kubectl exec -it graceful-web -- sh -c "echo 'Hello again' | nc localhost 8080"
```

**Expected output**:

```
HTTP/1.1 200 OK

Hello World
```

*Terminal-2 logs show that the request was processed.*

### **Observation**

The pod successfully handled the request while terminating.

### **Conclusion**

**Graceful termination allows already running processes inside the pod to complete within the grace period.**

---

# **Scenario 2: Forceful Termination**

## **Step 6: Recreate the Pod**

**Terminal-1**:

```bash
kubectl run graceful-web --image=busybox --restart=Never --command -- sh -c "while true; do echo -e 'HTTP/1.1 200 OK\n\nHello World' | nc -l -p 8080; done"
kubectl get pods
```

---

## **Step 7: Send a Request**

**Terminal-3**:

```bash
kubectl exec -it graceful-web -- sh -c "echo 'Hello' | nc localhost 8080"
```

**Expected output**:

```
HTTP/1.1 200 OK

Hello World
```

*Terminal-2 logs confirm request handling.*

---

## **Step 8: Force Delete Pod**

**Terminal-1**:

```bash
kubectl delete pod graceful-web --grace-period=0 --force
```

**Expected output**:

```
pod "graceful-web" force deleted
```

---

## **Step 9: Send a Request After Force Delete**

**Terminal-3**:

```bash
kubectl exec -it graceful-web -- sh -c "echo 'Hello again' | nc localhost 8080"
```

**Expected output**:

```
Error from server: pod graceful-web not found
```

*Terminal-2 logs disappear immediately.*

### **Observation**

The pod is killed instantly and cannot process any running or pending work.

### **Conclusion**

**Forceful termination skips the grace period, causing running processes to be terminated in between.**

---

# ✅ **Key Concept (Very Important)**

This behavior is **not about new requests**.

In real Kubernetes applications:

* New requests are stopped quickly by **Service endpoints and readiness**
* That happens in **both graceful and forceful deletion**

The **real difference** is what happens to:

> **Existing, already running processes and in-flight requests inside the pod**

---

## **Correct Mental Model**

### Graceful Termination

* Pod receives **SIGTERM**
* Kubernetes waits up to **30 seconds**
* Existing processes finish cleanly
* Database writes, logs, and transactions complete
* Pod exits safely

### Forceful Termination

* Pod receives **SIGKILL immediately**
* No waiting
* No cleanup
* Running processes are stopped in the middle

---


## **Summary Table**

| Scenario | Command                                       | Grace Period | Effect on Running Processes | Outcome             |
| -------- | --------------------------------------------- | ------------ | --------------------------- | ------------------- |
| Graceful | `kubectl delete pod`                          | 30s          | Allowed to complete         | No in-flight loss   |
| Forceful | `kubectl delete pod --force --grace-period=0` | 0s           | Killed immediately          | In-flight work lost |

---

This is **exactly how real-time applications behave in production Kubernetes clusters**.
