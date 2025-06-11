# Kubernetes Canary & Blue/Green Deployment Tutorial with GoLang

This tutorial walks you through implementing **Canary** and **Blue/Green** deployments on **Kubernetes** using a simple **GoLang** web application.

---

## 📦 Project Structure

```
k8s-deployment-strategies/
├── app/
│   └── main.go
├── Dockerfile
├── manifests/
│   ├── bluegreen/
│   │   ├── deployment-blue.yaml
│   │   ├── deployment-green.yaml
│   │   └── service.yaml
│   └── canary/
│       ├── deployment-v1.yaml
│       ├── deployment-v2.yaml
│       ├── service.yaml
├── README.md
```

---

## ⚙️ 1. GoLang Web App

`app/main.go`

```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func handler(w http.ResponseWriter, r *http.Request) {
    version := os.Getenv("VERSION")
    fmt.Fprintf(w, "Hello from version: %s\n", version)
}

func main() {
    http.HandleFunc("/", handler)
    fmt.Println("Server started on :8080")
    http.ListenAndServe(":8080", nil)
}
```

---

## 🐳 2. Dockerfile

```dockerfile
FROM golang:1.20-alpine
WORKDIR /app
COPY app/main.go .
RUN go build -o server main.go
ENV VERSION="v1"
CMD ["./server"]
```

---

## 🔵 3. Blue/Green Deployment

### `manifests/bluegreen/deployment-blue.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: server
        image: your-dockerhub-user/myapp:v1
        env:
        - name: VERSION
          value: "blue"
        ports:
        - containerPort: 8080
```

### `manifests/bluegreen/deployment-green.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: server
        image: your-dockerhub-user/myapp:v2
        env:
        - name: VERSION
          value: "green"
        ports:
        - containerPort: 8080
```

### `manifests/bluegreen/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    version: blue  # Change to green when switching
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

### 🔁 Switching Traffic in Blue/Green

To redirect traffic from the blue version to the green version:

1. Edit the `service.yaml` file:

```yaml
spec:
  selector:
    app: myapp
    version: green
```

2. Apply the updated service:

```sh
kubectl apply -f manifests/bluegreen/service.yaml
```

This switches all traffic from the old version to the new one.

---

## 🟡 4. Canary Deployment

### `manifests/canary/deployment-v1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: server
        image: your-dockerhub-user/myapp:v1
        env:
        - name: VERSION
          value: "v1"
        ports:
        - containerPort: 8080
```

### `manifests/canary/deployment-v2.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: server
        image: your-dockerhub-user/myapp:v2
        env:
        - name: VERSION
          value: "v2"
        ports:
        - containerPort: 8080
```

### `manifests/canary/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

The service will now load balance between both versions based on the number of replicas. For example, 3 replicas of `v1` and 1 replica of `v2` means \~25% of traffic goes to the canary release.

To gradually increase the traffic to `v2`, increase the replica count for `app-v2` and decrease it for `app-v1`.

---

## ☁️ 5. Deploy to Kubernetes

```sh
# Build and push your image
docker build -t your-dockerhub-user/myapp:v1 .
docker push your-dockerhub-user/myapp:v1
# Repeat for v2 after changing ENV VERSION in Dockerfile

# Apply blue/green or canary manifests
kubectl apply -f manifests/bluegreen/  # or manifests/canary/
```

---

## ✅ Final Tips

- Monitor logs: `kubectl logs -l app=myapp`
- Access service: `kubectl port-forward service/myapp-service 8080:80` — This command forwards the service's port 80 from inside the cluster to your local machine's port 8080. You can then access the application in your browser at [http://localhost:8080](http://localhost:8080), even if you're not using an external LoadBalancer or Ingress.

### 🧭 Alternative Ways to Access the Service (If Not Using `port-forward`)

1. **LoadBalancer Service** (for cloud environments like AWS, GCP, Azure):

```yaml
spec:
  type: LoadBalancer
  selector:
    app: myapp
    version: blue
```

This will expose your app via an external IP managed by your cloud provider.

2. **NodePort Service** (for local testing or Minikube):

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

Then access your app via:

```
http://<NODE_IP>:30080
```

3. **Ingress Controller + Ingress Resource** (for production-ready routing): Set up an ingress controller like **NGINX** or **Traefik**, then expose services via subdomains or paths using Ingress rules.

This is ideal for managing multiple services with custom domains (e.g. `myapp.example.com`).. You can then access the application in your browser at [http://localhost:8080](http://localhost:8080), even if you're not using an external LoadBalancer or Ingress.

- Use `kubectl rollout` to track status or undo

---

## 📄 LICENSE

MIT

## 📚 Credits

Based on Kubernetes Deployment Best Practices for Progressive Delivery.

