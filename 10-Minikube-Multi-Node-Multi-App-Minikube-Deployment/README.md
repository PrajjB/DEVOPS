# Exercise – Multi-Node Kubernetes Cluster with Multiple Applications and ReplicaSets

## Objective

The objective of this exercise was to deploy two Python Flask applications, a Product Catalog (AppA) and a Shopping Cart (AppB), on a three-node Minikube cluster, run them with 2 and 3 replicas respectively, make sure each replica runs on a different node, and expose and test both services.

## Scenario

An online store needs high availability for critical services such as product browsing and cart management. To avoid service disruptions, each service is deployed with replicas distributed across multiple nodes, so that even if one node fails the services remain available to customers.

| Service | Application | Replicas |
|---------|-------------|----------|
| Product Catalog | AppA | 2 |
| Shopping Cart | AppB | 3 |

## 1. Cleaned Up Any Existing Minikube Cluster

Stopped and deleted any existing Minikube cluster:

```bash
minikube stop
minikube delete
```

## 2. Created the Multi-Node Minikube Cluster

Created a Minikube cluster with three nodes in a profile named `devops-multinode`:

```bash
minikube start --nodes 3 -p devops-multinode --force
```

The `docker` driver should not normally be used with root privileges, for the security of the host. Since the exercise was run as root, the `--force` option was used.

For a multi-node cluster, `eval $(minikube -p devops-multinode docker-env)` cannot be used directly to share images with the nodes. The registry add-on was enabled for the profile:

```bash
minikube -p devops-multinode addons enable registry
```

Output recorded in the exercise:

```text
💡  registry is an addon maintained by minikube. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
╭──────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                                                                      │
│    Registry addon with docker driver uses port 32775 please use that instead of default port 5000    │
│                                                                                                      │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────╯
📘  For more information see: https://minikube.sigs.k8s.io/docs/drivers/docker
    ▪ Using image gcr.io/k8s-minikube/kube-registry-proxy:0.0.6
    ▪ Using image docker.io/registry:2.8.3
🔎  Verifying registry addon...
🌟  The 'registry' addon is enabled
```

## 3. Created the Product Catalog Application (AppA)

Created `product_catalog.py`, a Flask application with an endpoint that lists the available products:

```python
# product_catalog.py
from flask import Flask, jsonify

app = Flask(__name__)

products = [
    {"id": 1, "name": "Laptop", "price": 1200},
    {"id": 2, "name": "Phone", "price": 800},
    {"id": 3, "name": "Headphones", "price": 150},
]

@app.route("/products", methods=["GET"])
def get_products():
    return jsonify(products)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

## 4. Created the Shopping Cart Application (AppB)

Created `shopping_cart.py`, a Flask application with endpoints to view the cart and add items to it:

```python
# shopping_cart.py
from flask import Flask, jsonify, request

app = Flask(__name__)

cart = []

@app.route("/cart", methods=["GET"])
def get_cart():
    return jsonify(cart)

@app.route("/cart", methods=["POST"])
def add_to_cart():
    item = request.json
    cart.append(item)
    return jsonify(cart), 201

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

## 5. Dockerized the Applications

Created `Dockerfile.product` for the Product Catalog:

```dockerfile
# Dockerfile for Product Catalog
FROM python:3.9-slim
WORKDIR /app
COPY product_catalog.py /app/
RUN pip install flask
CMD ["python", "product_catalog.py"]
```

Created `Dockerfile.shopping` for the Shopping Cart:

```dockerfile
# Dockerfile for Shopping Cart
FROM python:3.9-slim
WORKDIR /app
COPY shopping_cart.py /app/
RUN pip install flask
CMD ["python", "shopping_cart.py"]
```

Built both images:

```bash
docker build -t product-catalog:latest -f Dockerfile.product .
docker build -t shopping-cart:latest -f Dockerfile.shopping .
```

Checked that the images were available locally:

```bash
docker images
```

```text
REPOSITORY              TAG        IMAGE ID       CREATED          SIZE
shopping-cart           latest     cdfedc6f31ff   8 seconds ago    137MB
product-catalog         latest     2e47c49941d7   53 seconds ago   137MB
```

## 6. Loaded the Images into the Minikube Nodes

Loaded both images into the `devops-multinode` cluster so that all its nodes can access them:

```bash
minikube -p devops-multinode image load product-catalog:latest
minikube -p devops-multinode image load shopping-cart:latest
```

Verified that the images were loaded:

```bash
minikube -p devops-multinode ssh -- docker images
```

Output recorded in the exercise:

```text
REPOSITORY                                TAG                  IMAGE ID       CREATED          SIZE
shopping-cart                             latest               cdfedc6f31ff   17 minutes ago   137MB
product-catalog                           latest               2e47c49941d7   18 minutes ago   137MB
kindest/kindnetd                          v20240813-c6f155d6   12968670680f   3 months ago     85.8MB
registry.k8s.io/kube-scheduler            v1.31.0              1766f54c897f   3 months ago     67.4MB
registry.k8s.io/kube-controller-manager   v1.31.0              045733566833   3 months ago     88.4MB
registry.k8s.io/kube-apiserver            v1.31.0              604f5db92eaa   3 months ago     94.2MB
registry.k8s.io/kube-proxy                v1.31.0              ad83b2ca7b09   3 months ago     91.5MB
registry.k8s.io/etcd                      3.5.15-0             2e96e5913fc0   4 months ago     148MB
registry.k8s.io/pause                     3.10                 873ed7510279   6 months ago     736kB
gcr.io/k8s-minikube/kube-registry-proxy   <none>               38c5e506fa55   8 months ago     187MB
registry.k8s.io/coredns/coredns           v1.11.1              cbb01a7bd410   15 months ago    59.8MB
gcr.io/k8s-minikube/storage-provisioner   v5                   6e38f40d628d   3 years ago      31.5MB
```

The `product-catalog` and `shopping-cart` images appear in the list.

## 7. Deployed the Applications

Created `product_catalog_deployment.yaml` with 2 replicas. A `podAntiAffinity` rule on `kubernetes.io/hostname` forces each replica onto a different node, and `imagePullPolicy: Never` makes Kubernetes use the locally loaded image.

```yaml
# product_catalog_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-catalog
  template:
    metadata:
      labels:
        app: product-catalog
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: product-catalog
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: product-catalog-container
        image: product-catalog:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

Created `shopping_cart_deployment.yaml` with 3 replicas, using the same anti-affinity rule:

```yaml
# shopping_cart_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopping-cart
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shopping-cart
  template:
    metadata:
      labels:
        app: shopping-cart
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: shopping-cart
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: shopping-cart-container
        image: shopping-cart:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

Applied both deployments:

```bash
kubectl apply -f product_catalog_deployment.yaml
kubectl apply -f shopping_cart_deployment.yaml
```

## 8. Exposed the Services

Created `product_catalog_service.yaml`:

```yaml
# product_catalog_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: product-catalog-service
  namespace: default
spec:
  selector:
    app: product-catalog
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

Created `shopping_cart_service.yaml`:

```yaml
# shopping_cart_service.yaml
apiVersion: v1
kind: Service
metadata:
  name: shopping-cart-service
  namespace: default
spec:
  selector:
    app: shopping-cart
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
```

Applied both services:

```bash
kubectl apply -f product_catalog_service.yaml
kubectl apply -f shopping_cart_service.yaml
```

## 9. Verified the Deployment and Pod Distribution

Listed the pods together with the nodes they were scheduled on:

```bash
kubectl get pods -o wide
```

Output recorded in the exercise:

```text
NAME                               READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
product-catalog-5fdfc7f5fb-2jlnv   1/1     Running   0          29s   10.244.1.3   devops-multinode-m02   <none>           <none>
product-catalog-5fdfc7f5fb-7v556   1/1     Running   0          29s   10.244.2.4   devops-multinode-m03   <none>           <none>
shopping-cart-788c48675b-7l4lx     1/1     Running   0          25s   10.244.0.4   devops-multinode       <none>           <none>
shopping-cart-788c48675b-d48w2     1/1     Running   0          25s   10.244.1.4   devops-multinode-m02   <none>           <none>
shopping-cart-788c48675b-hml9r     1/1     Running   0          25s   10.244.2.5   devops-multinode-m03   <none>           <none>
```

All five pods are `Running`, and the replicas of each service are on different nodes:

| Service | Pod | Node |
|---------|-----|------|
| Product Catalog | `product-catalog-5fdfc7f5fb-2jlnv` | `devops-multinode-m02` |
| Product Catalog | `product-catalog-5fdfc7f5fb-7v556` | `devops-multinode-m03` |
| Shopping Cart | `shopping-cart-788c48675b-7l4lx` | `devops-multinode` |
| Shopping Cart | `shopping-cart-788c48675b-d48w2` | `devops-multinode-m02` |
| Shopping Cart | `shopping-cart-788c48675b-hml9r` | `devops-multinode-m03` |

## 10. Accessed the Services

Used Minikube's service forwarding to access each service. Each command was run in a separate Linux or WSL terminal, which must stay open while the tunnel is in use (the Docker driver on Linux requires this).

**Product Catalog service**

```bash
minikube -p devops-multinode service product-catalog-service
```

```text
|-----------|-------------------------|-------------|---------------------------|
| NAMESPACE |          NAME           | TARGET PORT |            URL            |
|-----------|-------------------------|-------------|---------------------------|
| default   | product-catalog-service |          80 | http://192.168.49.2:32316 |
|-----------|-------------------------|-------------|---------------------------|
🏃  Starting tunnel for service product-catalog-service.
|-----------|-------------------------|-------------|------------------------|
| NAMESPACE |          NAME           | TARGET PORT |          URL           |
|-----------|-------------------------|-------------|------------------------|
| default   | product-catalog-service |             | http://127.0.0.1:35855 |
|-----------|-------------------------|-------------|------------------------|
🎉  Opening service default/product-catalog-service in default browser...
👉  http://127.0.0.1:35855
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```

**Shopping Cart service**

```bash
minikube -p devops-multinode service shopping-cart-service
```

```text
|-----------|-----------------------|-------------|---------------------------|
| NAMESPACE |         NAME          | TARGET PORT |            URL            |
|-----------|-----------------------|-------------|---------------------------|
| default   | shopping-cart-service |          80 | http://192.168.49.2:32382 |
|-----------|-----------------------|-------------|---------------------------|
🏃  Starting tunnel for service shopping-cart-service.
|-----------|-----------------------|-------------|------------------------|
| NAMESPACE |         NAME          | TARGET PORT |          URL           |
|-----------|-----------------------|-------------|------------------------|
| default   | shopping-cart-service |             | http://127.0.0.1:38975 |
|-----------|-----------------------|-------------|------------------------|
🎉  Opening service default/shopping-cart-service in default browser...
👉  http://127.0.0.1:38975
❗  Because you are using a Docker driver on linux, the terminal needs to be open to run it.
```

The URLs used for testing:

- Product Catalog: `http://127.0.0.1:35855`
- Shopping Cart: `http://127.0.0.1:38975`

## 11. Tested the Services with `curl`

**Product Catalog: retrieved the product list**

```bash
curl http://127.0.0.1:35855/products
```

Expected response:

```json
[
    {"id": 1, "name": "Laptop", "price": 1200},
    {"id": 2, "name": "Phone", "price": 800},
    {"id": 3, "name": "Headphones", "price": 150}
]
```

**Shopping Cart: retrieved the cart (empty initially)**

```bash
curl http://127.0.0.1:38975/cart
```

Expected response:

```json
[]
```

**Shopping Cart: added an item to the cart**

```bash
curl -X POST http://127.0.0.1:38975/cart -H "Content-Type: application/json" -d '{"id": 1, "name": "Laptop", "quantity": 1}'
```

Expected response:

```json
[
    {"id": 1, "name": "Laptop", "quantity": 1}
]
```

## 12. Result

The exercise covered creating a three-node Minikube cluster, building and loading Docker images for two Flask applications, and deploying them as Kubernetes Deployments (ReplicaSets) with 2 and 3 replicas. Pod anti-affinity on the node hostname placed every replica of a service on a different node, and both services were exposed with NodePort services and tested with `curl`. This setup maximizes availability and fault tolerance, because the loss of a single node still leaves running replicas of each service.