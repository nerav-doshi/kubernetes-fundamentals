# Kubernetes Core Concepts for OpenShift on AWS

## Prerequisites
Before starting, ensure you have:
- ROSA cluster set up and running
- `oc` CLI tool installed and configured
- Access to your OpenShift cluster

## 1. Pod - The Smallest Unit

A **Pod** is the smallest deployable unit in Kubernetes. It contains one or more containers that share storage and network.

### Example: Simple Nginx Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Deploy it:**
```bash
oc apply -f nginx-pod.yaml
oc get pods
oc describe pod my-nginx-pod
```

## 2. Deployment - Managing Pod Replicas

A **Deployment** manages multiple replicas of pods and provides rolling updates.

### Example: Nginx Deployment with 3 Replicas

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
```

**Deploy it:**
```bash
oc apply -f nginx-deployment.yaml
oc get deployments
oc get pods -l app=nginx
oc scale deployment nginx-deployment --replicas=5
```

## 3. Service - Network Access to Pods

A **Service** provides stable network access to a set of pods.

### Example: ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

**Deploy it:**
```bash
oc apply -f nginx-service.yaml
oc get services
oc describe service nginx-service
```

## 4. Route (OpenShift) - External Access

OpenShift uses **Routes** instead of Ingress for external access.

### Example: Exposing Your Service

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-route
spec:
  to:
    kind: Service
    name: nginx-service
  port:
    targetPort: 80
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**Or use the CLI:**
```bash
oc expose service nginx-service
oc get routes
```

## 5. ConfigMap - Configuration Data

**ConfigMaps** store configuration data that pods can consume.

### Example: Nginx Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen 80;
        location / {
            return 200 'Hello from OpenShift on AWS!\n';
            add_header Content-Type text/plain;
        }
    }
  app.properties: |
    environment=production
    log.level=info
```

### Using ConfigMap in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-config
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-config
  template:
    metadata:
      labels:
        app: nginx-config
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
        env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: nginx-config
              key: app.properties
      volumes:
      - name: config-volume
        configMap:
          name: nginx-config
```

## 6. Secret - Sensitive Data

**Secrets** store sensitive information like passwords, tokens, and keys.

### Example: Database Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded 'admin'
  password: MWYyZDFlMmU2N2Rm  # base64 encoded password
```

**Create from command line:**
```bash
oc create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret
```

## 7. Persistent Volume Claim - Storage

**PVCs** request storage resources for your applications.

### Example: Storage for a Database

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: gp2  # AWS EBS storage class
```

### PostgreSQL with Persistent Storage

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: "myapp"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

## 8. Complete Application Example

Let's put it all together with a web application that connects to a database:

### Frontend Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: DATABASE_URL
          value: "postgresql://postgres:5432/myapp"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: webapp-route
spec:
  to:
    kind: Service
    name: webapp-service
  tls:
    termination: edge
```

## Deployment Commands

Save each YAML section to separate files and deploy:

```bash
# Create a new project (namespace)
oc new-project my-k8s-learning

# Deploy in order
oc apply -f configmap.yaml
oc apply -f secret.yaml
oc apply -f pvc.yaml
oc apply -f postgres-deployment.yaml
oc apply -f webapp-deployment.yaml

# Check everything
oc get all
oc get pvc
oc get secrets
oc get configmaps
oc get routes

# Get the external URL
oc get route webapp-route
```

## Key OpenShift Differences

1. **Routes** instead of Ingress for external access
2. **Projects** instead of just Namespaces (though namespaces still exist)
3. **Security Context Constraints (SCCs)** for additional security
4. **Built-in registry** for container images
5. **Source-to-Image (S2I)** for building applications from source

## Next Steps

1. Try scaling your deployments: `oc scale deployment webapp --replicas=5`
2. Update applications: change image tags and apply
3. Monitor with: `oc logs -f deployment/webapp`
4. Access the OpenShift web console for a GUI experience

## Useful Commands

```bash
# View resources
oc get pods,svc,routes
oc describe deployment webapp
oc logs -f deployment/webapp

# Debug
oc exec -it <pod-name> -- /bin/bash
oc port-forward service/webapp-service 8080:80

# Cleanup
oc delete project my-k8s-learning
```

This gives you hands-on experience with the core Kubernetes concepts adapted for OpenShift on AWS!