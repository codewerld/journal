
<img width="535" height="436" alt="image" src="https://github.com/user-attachments/assets/002c4b3f-c149-479b-8f17-240780aa66b1" />

---

## Understanding Port Mapping: From Docker to Kubernetes

Based on the image provided, the annotation **`:3000/tcp`** indicates that the application inside the container is configured to run on **port 3000** using the TCP protocol.

### 1. Docker Port Mapping

In Docker, you map a **Host Port** to the **Container Port**. Think of it as creating a bridge:

* **Mapping `3000:3000`:** The app becomes accessible at `localhost:3000`.
* **Mapping `8080:3000`:** The app becomes accessible at `localhost:8080`.

How does Docker know to suggest port 3000? It retrieves this from the image metadata. In the **Dockerfile**, the developer includes an `EXPOSE 3000` instruction to signal which port the application uses.

---

### 2. Kubernetes Implementation

In Kubernetes, this logic is split between two objects: the **Pod** (which runs the container) and the **Service** (which directs traffic).

#### The Deployment (The Target)

The Deployment defines the container and specifies the `containerPort`. This must match the port the application is actually listening on.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: my-container
        image: my-app-image:latest
        ports:
        - containerPort: 3000  # Matches the app's internal port

```

#### The Service (The Exposer)

The Service acts as the entry point. It maps its own incoming port to the container’s port.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp        # Connects to Pods with the label "app: webapp"
  ports:
    - protocol: TCP
      port: 80         # The port the Service listens on (External/Cluster)
      targetPort: 3000 # The port the Pod is listening on (Internal)
  type: LoadBalancer   # Makes the service accessible via an external IP

```

---

### 3. Key Kubernetes Port Terms

| Term | Meaning |
| --- | --- |
| **`targetPort`** | The port **inside** the container (e.g., 3000). Must match the app code. |
| **`port`** | The port used **inside the cluster** to reach the service. |
| **`nodePort`** | A static port on each Node’s IP (usually 30000-32767) for external access. |

**The Traffic Flow:**

1. A user accesses the **LoadBalancer IP** on port `80`.
2. The **Service** receives the request and forwards it to the defined `targetPort` (`3000`).
3. The traffic reaches the **Pod** on port `3000`.

> **Important Note:** If your `targetPort` (3000) does not match the actual `containerPort` used by the app, the connection will time out. It’s like calling a phone number where no one is picking up!

---
