# Complete Ecommerce Application for ROSA

This example creates a microservices-based ecommerce application with:
- Frontend (React-based web app)
- Product Service (Node.js API)
- Order Service (Python Flask API)
- User Service (Node.js API)
- PostgreSQL Database
- Redis Cache
- MongoDB for Product Catalog

## Architecture Overview

```
Internet → Route → Frontend Service → Frontend Pod
                ↓
         API Gateway Service → Product/Order/User Services
                ↓
         Database Services (PostgreSQL, MongoDB, Redis)
```

## 1. Namespace and Basic Setup

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    name: ecommerce
```

## 2. ConfigMaps for Application Configuration

```yaml
# configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ecommerce
data:
  NODE_ENV: "production"
  API_VERSION: "v1"
  POSTGRES_DB: "ecommerce"
  MONGO_DB: "products"
  REDIS_DB: "0"
  LOG_LEVEL: "info"
  FRONTEND_PORT: "3000"
  PRODUCT_SERVICE_PORT: "3001"
  ORDER_SERVICE_PORT: "3002"
  USER_SERVICE_PORT: "3003"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: ecommerce
data:
  POSTGRES_DB: "ecommerce"
  POSTGRES_USER: "ecommerce_user"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
  namespace: ecommerce
data:
  MONGO_INITDB_DATABASE: "products"
```

## 3. Secrets for Sensitive Data

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
  namespace: ecommerce
type: Opaque
data:
  postgres-password: ZWNvbW1lcmNlMTIz  # ecommerce123
  mongo-password: bW9uZ29wYXNzMTIz      # mongopass123
  redis-password: cmVkaXNwYXNzMTIz      # redispass123
  jwt-secret: c3VwZXJzZWNyZXRqd3R0b2tlbg==  # supersecretjwttoken
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: ecommerce
type: Opaque
data:
  stripe-api-key: c2tfdGVzdF8xMjM0NTY3ODkw  # sk_test_1234567890
  email-password: ZW1haWxwYXNzd29yZA==      # emailpassword
```

## 4. Persistent Volume Claims

```yaml
# storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: ecommerce
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3-csi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  namespace: ecommerce
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: gp3-csi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: ecommerce
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: gp3-csi
```

## 5. Database Services

### PostgreSQL for Orders and Users

```yaml
# postgres.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: ecommerce
  labels:
    app: postgres
    tier: database
spec:
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
        image: postgres:15
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-config
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: postgres-password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
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
        readinessProbe:
          exec:
            command:
              - pg_isready
              - -U
              - ecommerce_user
              - -d
              - ecommerce
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          exec:
            command:
              - pg_isready
              - -U
              - ecommerce_user
              - -d
              - ecommerce
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: ecommerce
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
```

### MongoDB for Product Catalog

```yaml
# mongodb.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  namespace: ecommerce
  labels:
    app: mongodb
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
        tier: database
    spec:
      containers:
      - name: mongodb
        image: mongo:6.0
        env:
        - name: MONGO_INITDB_DATABASE
          valueFrom:
            configMapKeyRef:
              name: mongo-config
              key: MONGO_INITDB_DATABASE
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: mongo-password
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-storage
          mountPath: /data/db
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          exec:
            command:
              - mongo
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          exec:
            command:
              - mongo
              - --eval
              - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: mongo-storage
        persistentVolumeClaim:
          claimName: mongo-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: ecommerce
  labels:
    app: mongodb
spec:
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017
  type: ClusterIP
```

### Redis for Caching and Sessions

```yaml
# redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: ecommerce
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
        args:
          - redis-server
          - --requirepass
          - $(REDIS_PASSWORD)
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: redis-password
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-storage
          mountPath: /data
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        readinessProbe:
          exec:
            command:
              - redis-cli
              - --no-auth-warning
              - -a
              - $(REDIS_PASSWORD)
              - ping
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          exec:
            command:
              - redis-cli
              - --no-auth-warning
              - -a
              - $(REDIS_PASSWORD)
              - ping
          initialDelaySeconds: 30
          periodSeconds: 10
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: ecommerce
  labels:
    app: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
  type: ClusterIP
```

## 6. Microservices

### Product Service (Node.js)

```yaml
# product-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    app: product-service
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
        tier: backend
    spec:
      containers:
      - name: product-service
        image: node:18-alpine
        command: ["/bin/sh"]
        args: ["-c", "npm start"]
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: PRODUCT_SERVICE_PORT
        - name: MONGODB_URI
          value: "mongodb://admin:$(MONGO_PASSWORD)@mongodb-service:27017/products?authSource=admin"
        - name: MONGO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: mongo-password
        - name: REDIS_URL
          value: "redis://:$(REDIS_PASSWORD)@redis-service:6379/0"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: redis-password
        ports:
        - containerPort: 3001
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        # In a real scenario, you'd mount your app code or use a custom image
        workingDir: /app
        volumeMounts:
        - name: app-code
          mountPath: /app
      volumes:
      - name: app-code
        configMap:
          name: product-service-code
      initContainers:
      - name: install-deps
        image: node:18-alpine
        command: ["/bin/sh", "-c"]
        args: ["cd /app && npm install express mongoose redis cors helmet"]
        volumeMounts:
        - name: app-code
          mountPath: /app
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    app: product-service
spec:
  selector:
    app: product-service
  ports:
  - port: 3001
    targetPort: 3001
  type: ClusterIP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: product-service-code
  namespace: ecommerce
data:
  package.json: |
    {
      "name": "product-service",
      "version": "1.0.0",
      "main": "server.js",
      "scripts": {
        "start": "node server.js"
      }
    }
  server.js: |
    const express = require('express');
    const mongoose = require('mongoose');
    const redis = require('redis');
    const cors = require('cors');
    const helmet = require('helmet');
    
    const app = express();
    const PORT = process.env.PORT || 3001;
    
    // Middleware
    app.use(helmet());
    app.use(cors());
    app.use(express.json());
    
    // Connect to MongoDB
    mongoose.connect(process.env.MONGODB_URI)
      .then(() => console.log('Connected to MongoDB'))
      .catch(err => console.error('MongoDB connection error:', err));
    
    // Connect to Redis
    const redisClient = redis.createClient({
      url: process.env.REDIS_URL
    });
    redisClient.connect();
    
    // Product Schema
    const productSchema = new mongoose.Schema({
      name: { type: String, required: true },
      description: String,
      price: { type: Number, required: true },
      category: String,
      stock: { type: Number, default: 0 },
      imageUrl: String,
      createdAt: { type: Date, default: Date.now }
    });
    
    const Product = mongoose.model('Product', productSchema);
    
    // Routes
    app.get('/health', (req, res) => {
      res.json({ status: 'healthy', service: 'product-service' });
    });
    
    app.get('/api/products', async (req, res) => {
      try {
        const cacheKey = 'products:all';
        const cached = await redisClient.get(cacheKey);
        
        if (cached) {
          return res.json(JSON.parse(cached));
        }
        
        const products = await Product.find();
        await redisClient.setEx(cacheKey, 300, JSON.stringify(products));
        res.json(products);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    app.get('/api/products/:id', async (req, res) => {
      try {
        const product = await Product.findById(req.params.id);
        if (!product) {
          return res.status(404).json({ error: 'Product not found' });
        }
        res.json(product);
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    app.post('/api/products', async (req, res) => {
      try {
        const product = new Product(req.body);
        await product.save();
        await redisClient.del('products:all'); // Invalidate cache
        res.status(201).json(product);
      } catch (error) {
        res.status(400).json({ error: error.message });
      }
    });
    
    // Sample data initialization
    app.post('/api/init-data', async (req, res) => {
      try {
        const count = await Product.countDocuments();
        if (count === 0) {
          const sampleProducts = [
            {
              name: "Laptop Pro 15",
              description: "High-performance laptop for professionals",
              price: 1299.99,
              category: "Electronics",
              stock: 25,
              imageUrl: "/images/laptop-pro.jpg"
            },
            {
              name: "Wireless Headphones",
              description: "Premium noise-canceling headphones",
              price: 299.99,
              category: "Electronics",
              stock: 50,
              imageUrl: "/images/headphones.jpg"
            },
            {
              name: "Coffee Maker Deluxe",
              description: "Programmable coffee maker with grinder",
              price: 199.99,
              category: "Home",
              stock: 30,
              imageUrl: "/images/coffee-maker.jpg"
            }
          ];
          
          await Product.insertMany(sampleProducts);
          res.json({ message: 'Sample data initialized' });
        } else {
          res.json({ message: 'Data already exists' });
        }
      } catch (error) {
        res.status(500).json({ error: error.message });
      }
    });
    
    app.listen(PORT, () => {
      console.log(`Product Service running on port ${PORT}`);
    });
```

### Order Service (Python Flask)

```yaml
# order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    app: order-service
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        tier: backend
    spec:
      containers:
      - name: order-service
        image: python:3.11-slim
        command: ["/bin/sh"]
        args: ["-c", "pip install -r requirements.txt && python app.py"]
        env:
        - name: FLASK_ENV
          value: "production"
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: ORDER_SERVICE_PORT
        - name: DATABASE_URL
          value: "postgresql://ecommerce_user:$(POSTGRES_PASSWORD)@postgres-service:5432/ecommerce"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: postgres-password
        - name: REDIS_URL
          value: "redis://:$(REDIS_PASSWORD)@redis-service:6379/0"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: redis-password
        ports:
        - containerPort: 3002
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3002
          initialDelaySeconds: 15
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3002
          initialDelaySeconds: 30
          periodSeconds: 10
        workingDir: /app
        volumeMounts:
        - name: app-code
          mountPath: /app
      volumes:
      - name: app-code
        configMap:
          name: order-service-code
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: ecommerce
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  ports:
  - port: 3002
    targetPort: 3002
  type: ClusterIP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-service-code
  namespace: ecommerce
data:
  requirements.txt: |
    Flask==2.3.2
    psycopg2-binary==2.9.6
    redis==4.5.5
    SQLAlchemy==2.0.15
    Flask-SQLAlchemy==3.0.5
    requests==2.31.0
    python-dotenv==1.0.0
  app.py: |
    from flask import Flask, request, jsonify
    import psycopg2
    import redis
    import os
    import json
    from datetime import datetime
    import requests
    
    app = Flask(__name__)
    
    # Database connection
    def get_db_connection():
        return psycopg2.connect(os.getenv('DATABASE_URL'))
    
    # Redis connection
    redis_client = redis.from_url(os.getenv('REDIS_URL'))
    
    # Initialize database tables
    def init_db():
        conn = get_db_connection()
        cur = conn.cursor()
        
        cur.execute('''
            CREATE TABLE IF NOT EXISTS orders (
                id SERIAL PRIMARY KEY,
                user_id INTEGER NOT NULL,
                total_amount DECIMAL(10, 2) NOT NULL,
                status VARCHAR(50) DEFAULT 'pending',
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        cur.execute('''
            CREATE TABLE IF NOT EXISTS order_items (
                id SERIAL PRIMARY KEY,
                order_id INTEGER REFERENCES orders(id),
                product_id VARCHAR(50) NOT NULL,
                quantity INTEGER NOT NULL,
                price DECIMAL(10, 2) NOT NULL
            )
        ''')
        
        conn.commit()
        cur.close()
        conn.close()
    
    @app.route('/health')
    def health():
        return jsonify({'status': 'healthy', 'service': 'order-service'})
    
    @app.route('/api/orders', methods=['GET'])
    def get_orders():
        user_id = request.args.get('user_id')
        conn = get_db_connection()
        cur = conn.cursor()
        
        if user_id:
            cur.execute('SELECT * FROM orders WHERE user_id = %s ORDER BY created_at DESC', (user_id,))
        else:
            cur.execute('SELECT * FROM orders ORDER BY created_at DESC LIMIT 100')
        
        orders = cur.fetchall()
        cur.close()
        conn.close()
        
        return jsonify([{
            'id': order[0],
            'user_id': order[1],
            'total_amount': float(order[2]),
            'status': order[3],
            'created_at': order[4].isoformat() if order[4] else None
        } for order in orders])
    
    @app.route('/api/orders', methods=['POST'])
    def create_order():
        data = request.get_json()
        user_id = data.get('user_id')
        items = data.get('items', [])
        
        if not user_id or not items:
            return jsonify({'error': 'user_id and items are required'}), 400
        
        conn = get_db_connection()
        cur = conn.cursor()
        
        try:
            # Calculate total
            total_amount = 0
            for item in items:
                total_amount += item['price'] * item['quantity']
            
            # Create order
            cur.execute(
                'INSERT INTO orders (user_id, total_amount, status) VALUES (%s, %s, %s) RETURNING id',
                (user_id, total_amount, 'pending')
            )
            order_id = cur.fetchone()[0]
            
            # Add order items
            for item in items:
                cur.execute(
                    'INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (%s, %s, %s, %s)',
                    (order_id, item['product_id'], item['quantity'], item['price'])
                )
            
            conn.commit()
            
            # Cache order for quick access
            order_data = {
                'id': order_id,
                'user_id': user_id,
                'total_amount': total_amount,
                'status': 'pending',
                'items': items
            }
            redis_client.setex(f'order:{order_id}', 3600, json.dumps(order_data))
            
            return jsonify(order_data), 201
            
        except Exception as e:
            conn.rollback()
            return jsonify({'error': str(e)}), 500
        finally:
            cur.close()
            conn.close()
    
    @app.route('/api/orders/<int:order_id>', methods=['GET'])
    def get_order(order_id):
        # Try cache first
        cached_order = redis_client.get(f'order:{order_id}')
        if cached_order:
            return jsonify(json.loads(cached_order))
        
        conn = get_db_connection()
        cur = conn.cursor()
        
        cur.execute('SELECT * FROM orders WHERE id = %s', (order_id,))
        order = cur.fetchone()
        
        if not order:
            return jsonify({'error': 'Order not found'}), 404
        
        cur.execute('SELECT * FROM order_items WHERE order_id = %s', (order_id,))
        items = cur.fetchall()
        
        cur.close()
        conn.close()
        
        order_data = {
            'id': order[0],
            'user_id': order[1],
            'total_amount': float(order[2]),
            'status': order[3],
            'created_at': order[4].isoformat() if order[4] else None,
            'items': [{
                'product_id': item[2],
                'quantity': item[3],
                'price': float(item[4])
            } for item in items]
        }
        
        return jsonify(order_data)
    
    if __name__ == '__main__':
        init_db()
        app.run(host='0.0.0.0', port=int(os.getenv('PORT', 3002)), debug=False)
```

### User Service (Node.js with Authentication)

```yaml
# user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: ecommerce
  labels:
    app: user-service
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        tier: backend
    spec:
      containers:
      - name: user-service
        image: node:18-alpine
        command: ["/bin/sh"]
        args: ["-c", "npm start"]
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NODE_ENV
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: USER_SERVICE_PORT
        - name: DATABASE_URL
          value: "postgresql://ecommerce_user:$(POSTGRES_PASSWORD)@postgres-service:5432/ecommerce"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: postgres-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: jwt-secret
        - name: REDIS_URL
          value: "redis://:$(REDIS_PASSWORD)@redis-service:6379/0"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: redis-password
        ports:
        - containerPort: 3003
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health
            port: 3003
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 3003
          initialDelaySeconds: 30
          periodSeconds: 10
        workingDir: /app
        volumeMounts:
        - name: app-code
          mountPath: /app
      volumes:
      - name: app-code
        configMap:
          name: user-service-code
      initContainers:
      - name: install-deps
        image: node:18-alpine
        command: ["/bin/sh", "-c"]
        args: ["cd /app && npm install express pg bcryptjs jsonwebtoken redis cors helmet"]
        volumeMounts:
        - name: app-code
          mountPath: /app
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: ecommerce
  labels:
    app: user-service
spec:
  selector:
    app: user-service
  ports:
  - port: 3003
    targetPort: 3003
  type: ClusterIP
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-code
  namespace: ecommerce
data:
  package.json: |
    {
      "name": "user-service",
      "version": "1.0.0",
      "main": "server.js",
      "scripts": {
        "start": "node server.js"
      }
    }
  server.js: |
    const express = require('express');
    const { Pool } = require('pg');
    const bcrypt = require('bcryptjs');
    const jwt = require('jsonwebtoken');
    const redis = require('redis');
    const cors = require('cors');
    const helmet = require('helmet');
    
    const app = express();
    const PORT = process.env.PORT || 3003;
    
    // Middleware
    app.use(helmet());
    app.use(cors());
    app.use(express.json());
    
    // Database connection
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL,
    });
    
    // Redis connection
    const redisClient = redis.createClient({
      url: process.env.REDIS_URL
    });
    redisClient.connect();
    
    // Initialize database tables
    async function initDB() {
      const client = await pool.connect();
      try {
        await client.query(`
          CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            email VARCHAR(255) UNIQUE NOT NULL,
            password_hash VARCHAR(255) NOT NULL,
            first_name VARCHAR(100),
            last_name VARCHAR(100),
            created_at TIMESTAMP DEFAULT CURRENT_