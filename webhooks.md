# Kubernetes Webhooks - Examples and Basics

*Practical examples and step-by-step guide to understanding Kubernetes webhooks*

-----

## 🎯 What Are Webhooks Really?

Think of webhooks as **smart gatekeepers** that intercept every request going to the Kubernetes API server. They can:

- **Look at** what you’re trying to create/update
- **Modify it** before it gets saved (Mutating)
- **Approve or reject it** based on rules (Validating)

## 🔄 Webhook Flow

```
You submit: kubectl apply -f pod.yaml
    ↓
1. API Server receives request
    ↓
2. Authentication & Authorization 
    ↓
3. **MUTATING WEBHOOKS** (modify the object)
    ↓
4. **VALIDATING WEBHOOKS** (approve/reject)
    ↓
5. Object stored in etcd
    ↓
6. Controllers act on the object
```

-----

## 🔧 Mutating Webhook Examples

### Example 1: Auto-Adding Resource Limits

**Problem**: Developers forget to add resource limits, causing cluster issues.

**Solution**: Mutating webhook that automatically adds them.

**What happens**:

```yaml
# Developer submits this simple pod:
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: nginx
```

**Webhook automatically transforms it to**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    managed-by: "resource-webhook"
spec:
  containers:
  - name: app
    image: nginx
    resources:              # ← AUTOMATICALLY ADDED
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
```

### Example 2: Istio Sidecar Injection

**What Istio does**: Automatically injects a sidecar container for service mesh.

**Before webhook**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: frontend
spec:
  containers:
  - name: app
    image: nginx
```

**After Istio mutating webhook**:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: frontend
  annotations:
    sidecar.istio.io/status: "injected"
spec:
  containers:
  - name: app
    image: nginx
  - name: istio-proxy          # ← SIDECAR ADDED
    image: docker.io/istio/proxyv2:1.17.2
    # ... sidecar configuration
  volumes:                     # ← VOLUMES ADDED
  - name: istio-envoy
  - name: istio-data
```

### Example 3: Security Context Injection

**Webhook adds security settings**:

```yaml
# Original pod
spec:
  containers:
  - name: app
    image: nginx

# After security webhook
spec:
  securityContext:           # ← ADDED FOR SECURITY
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: nginx
    securityContext:         # ← CONTAINER SECURITY
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

-----

## ✅ Validating Webhook Examples

### Example 1: Image Registry Validation

**Rule**: Only allow images from approved registries.

**Webhook logic**:

```python
def validate_image(pod):
    approved_registries = [
        "mycompany.azurecr.io",
        "registry.company.com", 
        "docker.io/library"  # Official images only
    ]
    
    for container in pod.spec.containers:
        image = container.image
        
        # Check if image starts with approved registry
        if not any(image.startswith(registry) for registry in approved_registries):
            return {
                "allowed": False,
                "message": f"Image {image} not from approved registry"
            }
    
    return {"allowed": True}
```

**Result**:

- ✅ `mycompany.azurecr.io/webapp:v1.0` → **ALLOWED**
- ✅ `docker.io/library/nginx:latest` → **ALLOWED**
- ❌ `docker.io/untrusted/malware:latest` → **REJECTED**
- ❌ `quay.io/random/image:v1` → **REJECTED**

### Example 2: Resource Requirements Validation

**Rule**: All production pods must have resource limits.

```python
def validate_resources(pod):
    # Check if this is production
    if pod.metadata.namespace == "production":
        for container in pod.spec.containers:
            if not container.resources or not container.resources.limits:
                return {
                    "allowed": False,
                    "message": "Production pods must have resource limits"
                }
    return {"allowed": True}
```

### Example 3: Naming Convention Validation

**Rule**: Deployments must follow naming pattern: `{team}-{app}-{env}`

```python
def validate_naming(deployment):
    name = deployment.metadata.name
    pattern = r"^[a-z]+\-[a-z]+\-(dev|staging|prod)$"
    
    if not re.match(pattern, name):
        return {
            "allowed": False,
            "message": f"Deployment name '{name}' must follow pattern: team-app-env"
        }
    return {"allowed": True}
```

**Results**:

- ✅ `frontend-webapp-prod` → **ALLOWED**
- ✅ `backend-api-staging` → **ALLOWED**
- ❌ `MyApp-Production` → **REJECTED** (wrong format)
- ❌ `random-name` → **REJECTED** (missing environment)

-----

## 🛠️ Building a Simple Webhook

### Basic Webhook Server Structure

```python
from flask import Flask, request, jsonify
import json
import base64

app = Flask(__name__)

@app.route('/mutate', methods=['POST'])
def mutate():
    # Get the admission request
    admission_request = request.get_json()
    
    # Extract the object being created/updated
    obj = admission_request['request']['object']
    
    # Create patch to modify the object
    patches = []
    
    # Add a label
    patches.append({
        "op": "add",
        "path": "/metadata/labels/webhook-modified",
        "value": "true"
    })
    
    # Add resource limits if missing
    if 'resources' not in obj['spec']['containers'][0]:
        patches.append({
            "op": "add", 
            "path": "/spec/containers/0/resources",
            "value": {
                "limits": {"memory": "256Mi", "cpu": "200m"},
                "requests": {"memory": "128Mi", "cpu": "100m"}
            }
        })
    
    # Create admission response
    admission_response = {
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionResponse", 
        "response": {
            "uid": admission_request['request']['uid'],
            "allowed": True,
            "patchType": "JSONPatch",
            "patch": base64.b64encode(
                json.dumps(patches).encode()
            ).decode()
        }
    }
    
    return jsonify(admission_response)

@app.route('/validate', methods=['POST'])
def validate():
    admission_request = request.get_json()
    obj = admission_request['request']['object']
    
    # Validation logic
    allowed = True
    message = ""
    
    # Check if image is from approved registry
    image = obj['spec']['containers'][0]['image']
    if not image.startswith('mycompany.azurecr.io'):
        allowed = False
        message = f"Image {image} not from approved registry"
    
    # Create response
    admission_response = {
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionResponse",
        "response": {
            "uid": admission_request['request']['uid'],
            "allowed": allowed,
            "result": {"message": message} if not allowed else {}
        }
    }
    
    return jsonify(admission_response)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8443, ssl_context='adhoc')
```

### Webhook Configuration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingAdmissionWebhook
metadata:
  name: resource-injector
webhooks:
- name: inject-resources.example.com
  clientConfig:
    service:
      name: webhook-service
      namespace: webhook-system
      path: "/mutate"
  rules:
  - operations: [ "CREATE" ]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail  # Reject if webhook is down
```

-----

## 🚨 Common Webhook Pitfalls & Best Practices

### ❌ Common Mistakes

1. **Webhook Down = Cluster Down**
   
   ```yaml
   failurePolicy: Fail  # If webhook fails, all pod creation stops!
   ```
1. **Infinite Loops**
   
   ```python
   # BAD: Webhook modifies its own pods
   def mutate(obj):
       if obj.namespace == "webhook-system":  # Webhook's own namespace
           # This could create infinite loops!
   ```
1. **Performance Issues**
   
   ```python
   # BAD: Slow external API calls
   def validate(obj):
       response = requests.get("https://slow-api.com/validate")  # 30 second timeout!
   ```

### ✅ Best Practices

1. **Use Failure Policies Wisely**
   
   ```yaml
   failurePolicy: Ignore  # For non-critical webhooks
   # OR
   failurePolicy: Fail    # For security-critical webhooks
   ```
1. **Exclude Webhook Namespaces**
   
   ```yaml
   namespaceSelector:
     matchExpressions:
     - key: name
       operator: NotIn
       values: ["kube-system", "webhook-system"]
   ```
1. **Fast Response Times**
   
   ```python
   # GOOD: Fast, simple logic
   def validate(obj):
       # Simple checks only
       return {"allowed": True}
   
   # GOOD: Async processing for complex logic
   async def complex_validation(obj):
       # Use async for I/O operations
   ```
1. **Proper Error Handling**
   
   ```python
   try:
       result = validate_image(obj)
       return result
   except Exception as e:
       # Log error but don't block cluster
       logger.error(f"Validation error: {e}")
       return {"allowed": True, "warnings": ["Validation skipped due to error"]}
   ```

-----

## 🎯 When to Use Webhooks

### ✅ Good Use Cases

- **Security enforcement** (image scanning, security contexts)
- **Resource management** (adding limits, quotas)
- **Compliance** (labeling, naming conventions)
- **Service mesh** (sidecar injection)
- **Monitoring** (adding observability tools)

### ❌ When NOT to Use Webhooks

- **Simple policies** (use OPA Gatekeeper instead)
- **Business logic** (belongs in your application)
- **Heavy processing** (use controllers/operators instead)
- **External dependencies** (makes cluster fragile)

-----

## 🚀 Getting Started

1. **Start Simple**: Use existing tools like OPA Gatekeeper
1. **Learn by Example**: Deploy Istio to see sidecar injection
1. **Practice**: Build a simple label-adding mutating webhook
1. **Production**: Only after thorough testing and proper monitoring

Remember: **With great power comes great responsibility!** Webhooks can break your entire cluster if not implemented carefully. Start small and test thoroughly! 🛡️
