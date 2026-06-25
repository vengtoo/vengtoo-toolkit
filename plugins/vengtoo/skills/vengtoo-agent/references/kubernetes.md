# Vengtoo Agent — Kubernetes Deployment Reference

Two deployment patterns: **sidecar** (recommended for high-throughput services) and
**DaemonSet** (one agent per node, shared by all pods on the node).

---

## Sidecar (recommended)

The agent runs as a second container in the same pod as your application. Evaluations
happen over localhost — zero network hop.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      containers:
        # Your application
        - name: app
          image: your-org/my-service:latest
          env:
            - name: VENGTOO_BASE_URL
              value: "http://localhost:8181"
            - name: VENGTOO_API_KEY
              valueFrom:
                secretKeyRef:
                  name: vengtoo-credentials
                  key: api-key

        # Vengtoo Agent sidecar
        - name: vengtoo-agent
          image: vengtoo/agent:latest
          ports:
            - containerPort: 8181
          env:
            - name: VENGTOO_API_KEY
              valueFrom:
                secretKeyRef:
                  name: vengtoo-credentials
                  key: api-key
            - name: VENGTOO_DECISION_LOG
              value: "true"
            - name: VENGTOO_AUDIT_FORWARD
              value: "true"
            - name: VENGTOO_CACHE_DIR
              value: /var/lib/vengtoo/bundles
          volumeMounts:
            - name: vengtoo-cache
              mountPath: /var/lib/vengtoo/bundles
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8181
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8181
            initialDelaySeconds: 10
            periodSeconds: 30

      volumes:
        - name: vengtoo-cache
          emptyDir: {}
```

---

## Secret (API key)

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: vengtoo-credentials
type: Opaque
stringData:
  api-key: "azx_..."
```

```bash
# Or create from CLI (preferred — keeps key out of git)
kubectl create secret generic vengtoo-credentials \
  --from-literal=api-key="$VENGTOO_API_KEY"
```

---

## DaemonSet (shared agent per node)

Each node in the cluster runs one agent. Pods on the same node access it via the node's
internal IP. Higher memory efficiency; slightly more network latency than sidecar.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: vengtoo-agent
  namespace: vengtoo-system
spec:
  selector:
    matchLabels:
      app: vengtoo-agent
  template:
    metadata:
      labels:
        app: vengtoo-agent
    spec:
      containers:
        - name: vengtoo-agent
          image: vengtoo/agent:latest
          ports:
            - containerPort: 8181
              hostPort: 8181
          env:
            - name: VENGTOO_API_KEY
              valueFrom:
                secretKeyRef:
                  name: vengtoo-credentials
                  key: api-key
            - name: VENGTOO_HOST
              value: "0.0.0.0"
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /readyz
              port: 8181
```

In your application pods, set:
```yaml
env:
  - name: VENGTOO_BASE_URL
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
```

Then construct the URL: `http://${VENGTOO_BASE_URL}:8181`

---

## Service (for DaemonSet or standalone)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: vengtoo-agent
  namespace: vengtoo-system
spec:
  selector:
    app: vengtoo-agent
  ports:
    - port: 8181
      targetPort: 8181
  type: ClusterIP
```

Access from other pods:
```
http://vengtoo-agent.vengtoo-system.svc.cluster.local:8181
```

---

## PodDisruptionBudget (for sidecar at scale)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-service
```

---

## Init container (pre-warm cache before app starts)

Ensures the agent has a bundle loaded before the application begins serving traffic:

```yaml
initContainers:
  - name: vengtoo-agent-warmup
    image: vengtoo/agent:latest
    command: ["vengtoo-agent", "--warmup-only"]
    env:
      - name: VENGTOO_API_KEY
        valueFrom:
          secretKeyRef:
            name: vengtoo-credentials
            key: api-key
      - name: VENGTOO_CACHE_DIR
        value: /var/lib/vengtoo/bundles
    volumeMounts:
      - name: vengtoo-cache
        mountPath: /var/lib/vengtoo/bundles
```

The init container exits after the bundle is downloaded and cached. The main agent sidecar
starts with the bundle already on disk — zero startup latency.

---

## Helm values snippet

If using a Helm chart:
```yaml
vengtooAgent:
  enabled: true
  image: vengtoo/agent:latest
  apiKeySecret:
    name: vengtoo-credentials
    key: api-key
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi
  config:
    decisionLog: true
    auditForward: true
    bundleSyncInterval: 30s
```
