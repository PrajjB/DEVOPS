# Exercise 3 – Scaling Flask App on a Single Node using ReplicaSets

## Objective

Deploy a Flask-based "Flash Sale" application on Minikube using a ReplicaSet, scale it up and down, delete a Pod to observe self-healing, and view Pod distribution across the node.

## Real-Life Use Case: E-commerce Flash Sale

During a flash sale (e.g. Big Billion Days, Prime Day), traffic on a Flask service can spike from ~100 requests/minute to ~10,000 requests/minute. A single Pod would crash under that load. A ReplicaSet lets the system scale out to 10–20 Pods to distribute requests, then scale back down once traffic normalizes — saving resources without manual server provisioning.

## Environment

* OS: Windows 11
* Kubernetes: v1.31.0
* Minikube: v1.34.0
* Container Runtime: Docker
* Application: Flask (Python 3.11) served via Gunicorn

## 1. Create the Flask Application

Created `app.py`:

```python
from flask import Flask, request
import socket, time, random

app = Flask(__name__)

@app.get("/")
def homepage():
    return {
        "message": "Welcome to Big Sale!",
        "pod": socket.gethostname(),
        "ts": time.time()
    }

@app.get("/buy")
def buy():
    # simulate a flash sale checkout
    item = random.choice(["Smartphone", "Shoes", "Headphones", "Laptop"])
    user = request.args.get("user", f"user{random.randint(1,1000)}")
    return {
        "status": "success",
        "item": item,
        "user": user,
        "served_by_pod": socket.gethostname(),
        "time": time.strftime("%H:%M:%S")
    }

@app.get("/health")
def health():
    return {"status": "healthy", "pod": socket.gethostname()}
```

**App routes:**
* `/` → Welcome message for the Big Sale.
* `/buy` → Simulates a flash-sale checkout, returns a random product and the Pod that served the request (useful for observing load distribution).
* `/health` → Used for readiness/liveness probes.

## 2. Create the Dockerfile

Created `Dockerfile`:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY ex3-flash-sale.py .
RUN pip install --no-cache-dir flask gunicorn
CMD ["gunicorn","-b","0.0.0.0:5000","app:app","--workers","1","--threads","2"]
```

## 3. Build and Push the Docker Image

```bash
docker build -t <your-dockerhub-username>/flashsale:1.0 .
docker push <your-dockerhub-username>/flashsale:1.0
```

The image was built and pushed successfully to the Docker Hub repository.

## 4. Clean Up Any Existing Minikube Cluster

```bash
minikube stop
minikube delete
```

Any previously running cluster was stopped and removed to start fresh.

## 5. Start Minikube with a Single Node

```bash
minikube start --nodes=1
```

Output:

```text
😄  minikube v1.34.0 on Ubuntu 24.04 (amd64)
    ▪ MINIKUBE_ACTIVE_DOCKERD=minikube
✨  Automatically selected the docker driver
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.45 ...
🔥  Creating docker container (CPUs=2, Memory=2400MB) ...
🐳  Preparing Kubernetes v1.31.0 on Docker 27.2.0 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

Verified the node was ready:

```bash
kubectl get nodes
```

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   42s   v1.31.0
```

## 6. Create the ReplicaSet and Service YAML

Created `flashsale-replicaset.yaml`:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: flashsale-rs
  labels:
    app: flashsale
spec:
  replicas: 3  ## Number of replicas (copies of flashsale app) to run
  selector:
    matchLabels:
      app: flashsale
  template:
    metadata:
      labels:
        app: flashsale
    spec:
      containers:
      - name: flashsale-container
        image: flashsale:1.0
        ports:
        - containerPort: 5000
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 2
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: flashsale-svc
spec:
  selector:
    app: flashsale
  ports:
  - name: http
    port: 80
    targetPort: 5000
  type: ClusterIP
```

## 7. Apply the ReplicaSet Configuration

```bash
kubectl apply -f flashsale-replicaset.yaml
```

Output:

```text
replicaset.apps/flashsale-rs created
service/flashsale-svc created
```

## 8. Point Docker at Minikube and Build the Image Inside the Cluster

```bash
minikube docker-env
eval $(minikube docker-env)
docker build -t flask-app .
```

The image was built directly inside Minikube's Docker daemon so the ReplicaSet could use it locally.

## 9. Verify the ReplicaSet and Pods

```bash
kubectl get pods
```

```text
NAME                 READY   STATUS    RESTARTS   AGE
flashsale-rs-8gbfp   1/1     Running   0          3m35s
flashsale-rs-f4gsl   1/1     Running   0          3m35s
flashsale-rs-nb5kl   1/1     Running   0          3m35s
```

```bash
kubectl get rs
```

```text
NAME           DESIRED   CURRENT   READY   AGE
flask-app-rs   3         3         0       40s
```

All 3 desired Pods were created and running.

## 10. Scale the ReplicaSet to 5 Replicas

```bash
kubectl scale rs flask-app-rs --replicas=5
```

Output:

```text
replicaset.apps/flask-app-rs scaled
```

## 11. Verify the Scaled ReplicaSet

```bash
kubectl get rs
```

```text
NAME           DESIRED   CURRENT   READY   AGE
flask-app-rs   5         5         5       7m38s
```

## 12. Verify the Scaled Pods

```bash
kubectl get pods
```

```text
NAME                 READY   STATUS    RESTARTS   AGE
flask-app-rs-4nr6q   1/1     Running   0          7m55s
flask-app-rs-84v7x   1/1     Running   0          7m55s
flask-app-rs-nsmlx   1/1     Running   0          32s
flask-app-rs-rbwr4   1/1     Running   0          7m55s
flask-app-rs-wfbb4   1/1     Running   0          32s
```

Kubernetes created 2 additional Pods to meet the new desired count of 5.

## 13. Delete One Pod to Test Self-Healing

```bash
kubectl delete pod flask-app-rs-84v7x
```

Output:

```text
pod "flask-app-rs-84v7x" deleted
```

## 14. Verify the Pod Was Replaced

```bash
kubectl get pods
```

```text
NAME                 READY   STATUS    RESTARTS   AGE
flask-app-rs-4nr6q   1/1     Running   0          9m19s
flask-app-rs-hqtm7   1/1     Running   0          51s
flask-app-rs-nsmlx   1/1     Running   0          116s
flask-app-rs-rbwr4   1/1     Running   0          9m19s
flask-app-rs-wfbb4   1/1     Running   0          116s
```

The ReplicaSet automatically created `flask-app-rs-hqtm7` to replace the deleted Pod, maintaining 5 replicas as expected.

## 15. View Pod Distribution Across Nodes

```bash
kubectl get pods -o wide
```

```text
NAME                 READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
flask-app-rs-4nr6q   1/1     Running   0          10m     10.244.0.9    minikube   <none>           <none>
flask-app-rs-hqtm7   1/1     Running   0          109s    10.244.0.12   minikube   <none>           <none>
flask-app-rs-nsmlx   1/1     Running   0          2m54s   10.244.0.11   minikube   <none>           <none>
flask-app-rs-rbwr4   1/1     Running   0          10m     10.244.0.7    minikube   <none>           <none>
flask-app-rs-wfbb4   1/1     Running   0          2m54s   10.244.0.10   minikube   <none>           <none>
```

All 5 Pods were scheduled onto the single `minikube` node, each with its own Pod IP.

## 16. Key Observations and Learnings

* **Pod Distribution** – Each Pod is an identical worker; scaling means creating clones of the app.
* **Resiliency** – When a Pod is deleted, the ReplicaSet automatically creates a replacement, so users don't notice downtime.
* **Efficiency** – Pods are added under load and removed when demand drops, avoiding permanent over-provisioning.
* **Real-World Parallel** – This is the same mechanism companies like Netflix, YouTube, and Swiggy use to handle peak traffic hours.

## 17. Q&A

**Q1. What is the initial number of replicas in the ReplicaSet?**
A1: 3

**Q2. How many pods are running after applying the ReplicaSet configuration?**
A2: 3

**Q3. What happens when you scale the ReplicaSet to 5 replicas?**
A3: Kubernetes creates 2 additional pods to meet the desired number of replicas (5). The ReplicaSet now has 5 running pods.

**Q4. What happens when you delete one pod?**
A4: Kubernetes automatically creates a new pod to replace the deleted one, maintaining the desired number of replicas (5).

**Q5. How does Kubernetes maintain the desired number of replicas?**
A5: Kubernetes continuously monitors the number of running pods and compares it to the desired number of replicas. If there's a discrepancy, Kubernetes creates or deletes pods to maintain the desired state.

**Q6. How many nodes are running?**
A6: 1

**Q7. Where are the pods running with respect to nodes?**
A7: All 5 pods are running on the single node (`minikube`):
- Pod 1 (flask-app-rs-4nr6q)
- Pod 2 (flask-app-rs-hqtm7)
- Pod 3 (flask-app-rs-nsmlx)
- Pod 4 (flask-app-rs-rbwr4)
- Pod 5 (flask-app-rs-wfbb4)

## 18. Commands Used

```bash
docker build -t <your-dockerhub-username>/flashsale:1.0 .
docker push <your-dockerhub-username>/flashsale:1.0

minikube stop
minikube delete

minikube start --nodes=1

kubectl get nodes

kubectl apply -f flashsale-replicaset.yaml

minikube docker-env
eval $(minikube docker-env)
docker build -t flask-app .

kubectl get pods
kubectl get rs

kubectl scale rs flask-app-rs --replicas=5

kubectl get rs
kubectl get pods

kubectl delete pod flask-app-rs-84v7x

kubectl get pods
kubectl get pods -o wide
```

## 19. Result

A Flask "Flash Sale" application was successfully deployed on a single-node Minikube cluster using a ReplicaSet with an initial 3 replicas.

The ReplicaSet was scaled up to 5 replicas, one Pod was deleted to test self-healing, and Kubernetes automatically replaced it to maintain the desired replica count. All Pods were confirmed running on the single available node.

## Additional Challenges

* Update `replicaset.yaml` to use a different image.
* Create a Deployment instead of a ReplicaSet, and compare behavior.
* Use `kubectl describe` to inspect the ReplicaSet and its Pods in detail.

## Tips and Variations

* Use `kubectl get pods -o wide` to see Pod distribution across nodes.
* Use `kubectl logs <pod-name>` to view individual Pod logs.
* Use `kubectl exec -it <pod-name> -- /bin/sh` to access a Pod's container shell.