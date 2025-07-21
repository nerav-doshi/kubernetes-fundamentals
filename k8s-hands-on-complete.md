# Complete Kubernetes Hands-On Example
## Building an E-Commerce Application on Azure Red Hat OpenShift

*A comprehensive start-to-finish example covering all Kubernetes concepts*

---

## 🎯 What We'll Build

A complete e-commerce application with:
- **Frontend**: React web application
- **Backend**: Node.js API server  
- **Database**: PostgreSQL with persistent storage
- **Cache**: Redis for session management
- **Monitoring**: Prometheus and Grafana
- **Security**: RBAC, Network Policies, Secrets
- **Scaling**: HPA (Horizontal Pod Autoscaler)
- **Jobs**: Database migration and cleanup tasks

## 📋 Prerequisites

```bash
# Verify you're connected to your OpenShift cluster
oc whoami
oc project

# Create our project namespace
oc new-project ecommerce-demo
```

---

## 🗂️ Project Structure

```
ecommerce-k8s/
├── 01-namespace/
├── 02-secrets-configmaps/
├── 03-storage/
├── 04-database/
├── 05-backend/
├── 06-frontend/
├── 07-cache/
├── 08-services/
├── 09-ingress/
├── 10-monitoring/
├── 11-security/
├── 12-scaling/
├── 13-jobs/
└── 14-cleanup/
```

---

## 1️⃣ Namespace and Basic Setup

### `01-namespace/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce-demo
  labels:
    name: ecommerce-demo
    environment: development
    team: platform
  annotations:
    description: "E-commerce demo application"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ecommerce-quota
  namespace: ecommerce-demo
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8" 
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: ecommerce-limits
  namespace: ecommerce-demo
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

**Apply:**
```bash
oc apply -f 01-namespace/namespace.yaml
oc project ecommerce-demo
```

---

## 2️⃣ Secrets and ConfigMaps

### `02-secrets-configmaps/database-secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: ecommerce-demo
type: Opaque
data:
  # postgres
  username: cG9zdGdyZXM=
  # password123  
  password: cGFzc3dvcmQxMjM=
  # ecommerce_db
  database: ZWNvbW1lcmNlX2Ri
---
apiVersion: v1
kind: Secret
metadata:
  name: api-keys
  namespace: ecommerce-demo
type: Opaque
data:
  # your-jwt-secret-key-here
  jwt-secret: eW91ci1qd3Qtc2VjcmV0LWtleS1oZXJl
  # your-stripe-key-here
  stripe-key: eW91ci1zdHJpcGUta2V5LWhlcmU=
```

### `02-secrets-configmaps/app-config.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: ecommerce-demo
data:
  NODE_ENV: "development"
  PORT: "3000"
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
  LOG_LEVEL: "info"
  SESSION_TIMEOUT: "3600"
  API_VERSION: "v1"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: ecommerce-demo
data:
  REACT_APP_API_URL: "http://backend-service:3000/api"
  REACT_APP_ENVIRONMENT: "development"
  REACT_APP_VERSION: "1.0.0"
```

**Apply:**
```bash
oc apply -f 02-secrets-configmaps/
```

---

## 3️⃣ Persistent Storage

### `03-storage/postgres-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-storage
  namespace: ecommerce-demo
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: managed-csi  # Azure Red Hat OpenShift default
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-storage
  namespace: ecommerce-demo
  labels:
    app: redis
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: managed-csi
```

**Apply:**
```bash
oc apply -f 03-storage/postgres-pvc.yaml
```

---

## 4️⃣ Database (PostgreSQL with StatefulSet)

### `04-database/postgres-statefulset.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce-demo
  labels:
    app: postgres
    tier: database
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
        tier: database
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: database
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
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
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U $POSTGRES_USER -d $POSTGRES_DB
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-storage
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: ecommerce-demo
  labels:
    app: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: ecommerce-demo
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres
  type: ClusterIP
```

**Apply:**
```bash
oc apply -f 04-database/postgres-statefulset.yaml
```

---

## 5️⃣ Backend API (Deployment with Rolling Updates)

### `05-backend/backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: ecommerce-demo
  labels:
    app: backend
    tier: api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: api
        version: v1
    spec:
      containers:
      - name: backend
        image: node:18-alpine
        command: ["sh", "-c"]
        args:
        - |
          npm init -y
          npm install express pg redis cors helmet
          cat > server.js << 'EOF'
          const express = require('express');
          const { Pool } = require('pg');
          const redis = require('redis');
          const cors = require('cors');
          const helmet = require('helmet');
          
          const app = express();
          const PORT = process.env.PORT || 3000;
          
          // Middleware
          app.use(helmet());
          app.use(cors());
          app.use(express.json());
          
          // Database connection
          const pool = new Pool({
            user: process.env.DB_USER,
            host: process.env.DB_HOST,
            database: process.env.DB_NAME,
            password: process.env.DB_PASSWORD,
            port: process.env.DB_PORT,
          });
          
          // Redis connection
          const redisClient = redis.createClient({
            host: process.env.REDIS_HOST,
            port: process.env.REDIS_PORT
          });
          
          // Health check endpoints
          app.get('/health', (req, res) => {
            res.status(200).json({ 
              status: 'healthy', 
              timestamp: new Date().toISOString(),
              version: process.env.API_VERSION 
            });
          });
          
          app.get('/ready', async (req, res) => {
            try {
              await pool.query('SELECT 1');
              res.status(200).json({ status: 'ready' });
            } catch (err) {
              res.status(503).json({ status: 'not ready', error: err.message });
            }
          });
          
          // API endpoints
          app.get('/api/products', async (req, res) => {
            try {
              const result = await pool.query('SELECT * FROM products LIMIT 10');
              res.json(result.rows);
            } catch (err) {
              res.status(500).json({ error: 'Database error' });
            }
          });
          
          app.get('/api/info', (req, res) => {
            res.json({
              service: 'ecommerce-backend',
              version: process.env.API_VERSION,
              environment: process.env.NODE_ENV,
              hostname: require('os').hostname()
            });
          });
          
          // Graceful shutdown
          process.on('SIGTERM', () => {
            console.log('SIGTERM received, shutting down gracefully');
            server.close(() => {
              pool.end();
              redisClient.quit();
              process.exit(0);
            });
          });
          
          const server = app.listen(PORT, '0.0.0.0', () => {
            console.log(`Backend server running on port ${PORT}`);
          });
          EOF
          node server.js
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: NODE_ENV
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: PORT
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DB_PORT
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: password
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: database
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: REDIS_HOST
        - name: REDIS_PORT
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: REDIS_PORT
        - name: API_VERSION
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: API_VERSION
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-keys
              key: jwt-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          failureThreshold: 30
          periodSeconds: 10
```

**Apply:**
```bash
oc apply -f 05-backend/backend-deployment.yaml
```

---

## 6️⃣ Frontend (React App)

### `06-frontend/frontend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce-demo
  labels:
    app: frontend
    tier: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        command: ["sh", "-c"]
        args:
        - |
          # Create a simple React-like static site
          mkdir -p /usr/share/nginx/html
          cat > /usr/share/nginx/html/index.html << 'EOF'
          <!DOCTYPE html>
          <html lang="en">
          <head>
              <meta charset="UTF-8">
              <meta name="viewport" content="width=device-width, initial-scale=1.0">
              <title>E-Commerce Demo</title>
              <style>
                  body { font-family: Arial, sans-serif; margin: 0; padding: 20px; background: #f5f5f5; }
                  .container { max-width: 1200px; margin: 0 auto; }
                  .header { background: #2196F3; color: white; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
                  .card { background: white; padding: 20px; margin: 10px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
                  .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; }
                  button { background: #4CAF50; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
                  button:hover { background: #45a049; }
                  .status { padding: 10px; border-radius: 4px; margin: 10px 0; }
                  .success { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
                  .error { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
              </style>
          </head>
          <body>
              <div class="container">
                  <div class="header">
                      <h1>🛒 E-Commerce Demo Application</h1>
                      <p>Kubernetes Complete Example - Running on OpenShift</p>
                  </div>
                  
                  <div class="grid">
                      <div class="card">
                          <h3>📊 System Status</h3>
                          <div id="status">Checking...</div>
                          <button onclick="checkHealth()">Check Health</button>
                      </div>
                      
                      <div class="card">
                          <h3>🛍️ Products</h3>
                          <div id="products">Loading...</div>
                          <button onclick="loadProducts()">Load Products</button>
                      </div>
                      
                      <div class="card">
                          <h3>ℹ️ Application Info</h3>
                          <div id="appinfo">Loading...</div>
                          <button onclick="getAppInfo()">Get Info</button>
                      </div>
                  </div>
              </div>
              
              <script>
                  const API_URL = window.location.protocol + '//' + window.location.hostname + ':' + (window.location.port || '80');
                  
                  async function checkHealth() {
                      try {
                          const response = await fetch(API_URL + '/health');
                          const data = await response.json();
                          document.getElementById('status').innerHTML = 
                              '<div class="status success">✅ Backend is healthy!<br>Status: ' + data.status + '<br>Time: ' + data.timestamp + '</div>';
                      } catch (error) {
                          document.getElementById('status').innerHTML = 
                              '<div class="status error">❌ Backend is not responding<br>Error: ' + error.message + '</div>';
                      }
                  }
                  
                  async function loadProducts() {
                      try {
                          const response = await fetch(API_URL + '/api/products');
                          const products = await response.json();
                          document.getElementById('products').innerHTML = 
                              '<div class="status success">Found ' + products.length + ' products</div>';
                      } catch (error) {
                          document.getElementById('products').innerHTML = 
                              '<div class="status error">Failed to load products: ' + error.message + '</div>';
                      }
                  }
                  
                  async function getAppInfo() {
                      try {
                          const response = await fetch(API_URL + '/api/info');
                          const info = await response.json();
                          document.getElementById('appinfo').innerHTML = 
                              '<div class="status success">' +
                              'Service: ' + info.service + '<br>' +
                              'Version: ' + info.version + '<br>' +
                              'Environment: ' + info.environment + '<br>' +
                              'Pod: ' + info.hostname + 
                              '</div>';
                      } catch (error) {
                          document.getElementById('appinfo').innerHTML = 
                              '<div class="status error">Failed to get info: ' + error.message + '</div>';
                      }
                  }
                  
                  // Auto-check health on load
                  window.onload = function() {
                      checkHealth();
                      getAppInfo();
                  };
              </script>
          </body>
          </html>
          EOF
          nginx -g 'daemon off;'
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

**Apply:**
```bash
oc apply -f 06-frontend/frontend-deployment.yaml
```

---

## 7️⃣ Cache (Redis)

### `07-cache/redis-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ecommerce-demo
  labels:
    app: redis
    tier: cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
        tier: cache
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server"]
        args: ["--appendonly", "yes", "--bind", "0.0.0.0"]
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: redis-storage
          mountPath: /data
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-storage
```

**Apply:**
```bash
oc apply -f 07-cache/redis-deployment.yaml
```

---

## 8️⃣ Services (Communication Layer)

### `08-services/services.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: ecommerce-demo
  labels:
    app: backend
spec:
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
    name: http
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: ecommerce-demo
  labels:
    app: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    name: http
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: ecommerce-demo
  labels:
    app: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
    name: redis
  type: ClusterIP
```

**Apply:**
```bash
oc apply -f 08-services/services.yaml
```

---

## 9️⃣ Ingress (External Access)

### `09-ingress/ingress.yaml`
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ecommerce-frontend
  namespace: ecommerce-demo
  labels:
    app: frontend
spec:
  to:
    kind: Service
    name: frontend-service
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ecommerce-api
  namespace: ecommerce-demo
  labels:
    app: backend
spec:
  path: /api
  to:
    kind: Service
    name: backend-service
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ecommerce-health
  namespace: ecommerce-demo
  labels:
    app: backend
spec:
  path: /health
  to:
    kind: Service
    name: backend-service
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

**Apply:**
```bash
oc apply -f 09-ingress/ingress.yaml
```

---

## 🔟 Monitoring (Observability)

### `10-monitoring/monitoring.yaml`
```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: backend-metrics
  namespace: ecommerce-demo
  labels:
    app: backend
spec:
  selector:
    matchLabels:
      app: backend
  endpoints:
  - port: http
    path: /health
    interval: 30s
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard
  namespace: ecommerce-demo
data:
  dashboard.json: |
    {
      "dashboard": {
        "title": "E-Commerce Application",
        "panels": [
          {
            "title": "Pod Status",
            "type": "stat",
            "targets": [
              {
                "expr": "up{job=\"backend-metrics\"}"
              }
            ]
          }
        ]
      }
    }
```

**Apply:**
```bash
oc apply -f 10-monitoring/monitoring.yaml
```

---

## 1️⃣1️⃣ Security (RBAC & Network Policies)

### `11-security/rbac.yaml`
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ecommerce-backend
  namespace: ecommerce-demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ecommerce-demo
  name: ecommerce-backend-role
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ecommerce-backend-binding
  namespace: ecommerce-demo
subjects:
- kind: ServiceAccount
  name: ecommerce-backend
  namespace: ecommerce-demo
roleRef:
  kind: Role
  name: ecommerce-backend-role
  apiGroup: rbac.authorization.k8s.io
```

### `11-security/network-policy.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-netpol
  namespace: ecommerce-demo
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-netpol
  namespace: ecommerce-demo
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
```

**Apply:**
```bash
oc apply -f 11-security/rbac.yaml
oc apply -f 11-security/network-policy.yaml
```

---

## 1️⃣2️⃣ Scaling (HPA)

### `12-scaling/hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: ecommerce-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
```

**Apply:**
```bash
oc apply -f 12-scaling/hpa.yaml
```

---

## 1️⃣3️⃣ Jobs (Database Migration & Cleanup)

### `13-jobs/database-migration.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: ecommerce-demo
spec:
  template:
    spec:
      containers:
      - name: migration
        image: postgres:15-alpine
        command: ["psql"]
        args: 
        - "-h"
        - "$(DB_HOST)"
        - "-U"
        - "$(DB_USER)"
        - "-d"
        - "$(DB_NAME)"
        - "-c"
        - |
          CREATE TABLE IF NOT EXISTS products (
            id SERIAL PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            price DECIMAL(10,2) NOT NULL,
            description TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
          );
          
          INSERT INTO products (name, price, description) VALUES
          ('Laptop', 999.99, 'High-performance laptop'),
          ('Phone', 599.99, 'Smartphone with great camera'),
          ('Headphones', 199.99, 'Wireless noise-canceling headphones'),
          ('Tablet', 399.99, '10-inch tablet for productivity'),
          ('Watch',