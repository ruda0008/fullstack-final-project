

# Best Buy Cloud-Native Application


##  Demo Video

**[Watch the Full Demo on YouTube](https://youtu.be/iKE3WCEGcmk)**





---

##  Architecture Overview

https://drive.google.com/file/d/1jauWVfz6036q7fffDUwdZQP5QUAO1mUV/view?usp=drive_link

![alt text](<Untitled Diagram.drawio (3).png>)

### System Design

This application implements a **microservices architecture** with **independent services** orchestrated on Azure Kubernetes Service (AKS).

#### Core Microservices (5)

| Service | Technology | Purpose |
|---------|-----------|---------|
| **Store-Front** | Vue.js  | Customer-facing web application | 
| **Store-Admin** | Vue.js | Employee management portal | 
| **Order-Service** | Node.js  | Order submission and queue publishing |
| **Product-Service** | Node.js | Product catalog management | 
| **Makeline-Service** | Go | Background order processing worker | 

#### Infrastructure Components (Stateful)

| Component | Type | Purpose | Replicas |
|-----------|------|---------|----------|
| **RabbitMQ** | Message Broker | AMQP async message queue | 1 |
| **MongoDB** | NoSQL Database | Order and product data persistence | 3 (Replica Set) |





### Design Decisions

#### Why Microservices?

1. **Independent Scaling**: Each service scales based on its own load patterns
   - Peak hours? Scale Order-Service to handle 10,000 req/sec
   - Normal day? Run single replicas to save costs

2. **Technology Flexibility**: Best tool for each job
   - Vue.js for  UI
   - Node.js for I/O-intensive APIs
   - Go for high-performance background processing

3. **Fault Isolation**: Service failures don't cascade
   - Product-Service crash? Customers can still place orders
   - RabbitMQ ensures orders aren't lost

#### Why RabbitMQ Message Queue?

The Order-Service and Makeline-Service codebases are **designed to work with a message queue** (AMQP protocol). This isn't optional it's architectural.

**Benefits:**

1. **Async Processing**: Customers see "Order Placed" instantly while fulfillment happens in background
2. **Reliability**: If Makeline crashes, orders remain safe in the queue
3. **Load Leveling**: Queue absorbs traffic spikes (10,000 orders/min → processes 100/sec)
4. **Decoupling**: Services communicate via messages, not tight HTTP coupling


#### Why MongoDB Replica Set?

1. **High Availability**: 3 replicas eliminate single point of failure
   - 1 PRIMARY for writes
   - 2 SECONDARY for read scaling
   - Automatic PRIMARY election if node fails

2. **Data Persistence**: Each replica has a PersistentVolumeClaim
   - Data survives pod deletions
   - Data survives node failures
   - Data survives cluster upgrades

3. **Performance**: Read queries distributed across SECONDARY nodes

#### Why StatefulSets for Databases?

- **Stable Network Identity**: Each pod has predictable DNS name (mongodb-0, mongodb-1)
- **Ordered Deployment**: Pods start in sequence, not simultaneously
- **Persistent Storage**: Each pod bound to its own PersistentVolume

---



### Cloud Platform

| Component | Azure Service | Purpose |
|-----------|--------------|---------|
| Kubernetes | Azure AKS | Container orchestration |
| Networking | Azure Load Balancer | External service access |
| CI/CD | GitHub Actions | Automated build & deploy |

---

##  Repository Links

### Source Code Repositories

| Service | GitHub Repository | Docker Hub Image |
|---------|-------------------|------------------|
| **Store-Front** | https://github.com/ruda0008/store-front-final-project| [venom0836/store-front-final-project:1.0.0](https://hub.docker.com/r/venom0836/store-front-final-project) |
| **Store-Admin** | https://github.com/ruda0008/store-admin-final-project | [venom0836/store-admin-final-project:1.0.0](https://hub.docker.com/r/venom0836/store-admin-final-project) |
| **Order-Service** | https://github.com/ruda0008/order-service-final-project | [venom0836/order-service-final-project:1.0.0](https://hub.docker.com/r/venom0836/order-service-final-project) |
| **Product-Service** | https://github.com/ruda0008/product-service-final-project | [venom0836/product-service-final-project:1.0.0](https://hub.docker.com/r/venom0836/product-service-final-project) |
| **Makeline-Service** | https://github.com/ruda0008/makeline-service-final-project | [venom0836/makeline-service-final-project:1.0.0](https://hub.docker.com/r/venom0836/makeline-service-final-project) |



---

##  Deployment Instructions

### Prerequisites

- Azure CLI installed and authenticated
- kubectl installed
- Docker Hub account
- Git

### Step 1: Create Azure Kubernetes Cluster

```bash
# Set variables
RESOURCE_GROUP="bestbuy-rg"
CLUSTER_NAME="bestbuy-aks"
LOCATION="eastus"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create AKS cluster with 2 nodes
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME

# Verify connection
kubectl get nodes
```




### Step 2: Clone Deployment Repository


### Step 3: Deploy ConfigMaps 

```bash
# Deploy RabbitMQ configuration
kubectl apply -f config-maps.yaml

# Verify
kubectl get configmaps
```

### Step 4: Deploy All Services

```bash
# Deploy everything
kubectl apply -f aps-all-in-one.yaml

# Watch pods start (takes 2-3 minutes)
kubectl get pods -w
```

**Wait until all pods show `Running` status:**


Press `Ctrl+C` to exit watch mode.

### Step 5: Initialize MongoDB Replica Set

```bash
# Wait for all MongoDB pods to be ready
kubectl wait --for=condition=ready pod/mongodb-0 pod/mongodb-1 pod/mongodb-2 --timeout=300s

# Initialize replica set
kubectl exec -it mongodb-0 -- mongo --eval '
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-0.mongodb:27017" },
    { _id: 1, host: "mongodb-1.mongodb:27017" },
    { _id: 2, host: "mongodb-2.mongodb:27017" }
  ]
})
'

# Wait for replica set election (15 seconds)
sleep 15

# Verify replica set status
kubectl exec -it mongodb-0 -- mongo --eval 'rs.status()' | grep stateStr
```

**Expected output:**
```
"stateStr" : "PRIMARY",
"stateStr" : "SECONDARY",
"stateStr" : "SECONDARY",
```

### Step 6: Expose Frontend Services

```bash
# Create LoadBalancer services for external access
kubectl expose deployment store-front --type=LoadBalancer --port=80 --target-port=8080
kubectl expose deployment store-admin --type=LoadBalancer --port=80 --target-port=8081

# Wait for external IPs (1-2 minutes)
kubectl get services store-front store-admin -w
```



### Step 7: Access the Application

```bash
# Get service URLs
kubectl get services store-front store-admin
```

**Open in browser:**
- **Customer Store:** `http://<store-front-external-ip>`
- **Admin Portal:** `http://<store-admin-external-ip>`

---

##  CI/CD Pipeline

### GitHub Actions Workflow

Each microservice repository contains a `.github/workflows/ci-cd.yml` file that automates:

1. **Code checkout** on push to `main` branch
2. **Docker image build** with multi-platform support (linux/amd64)
3. **Image push** to Docker Hub with version tagging
4. **Deployment update** (optional webhook trigger)

### Pipeline Configuration

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
```

### Testing the Pipeline

```bash
# Make a change to any service
cd store-front-L8
echo "<!-- Updated: $(date) -->" >> public/index.html

# Commit and push
git add .
git commit -m "feat: update build timestamp"
git push origin main

# Watch GitHub Actions tab for pipeline execution
# New image will appear on Docker Hub within 2-3 minutes
```

### Pipeline Benefits

- **Zero Manual Steps**: Developers push code, pipeline handles the rest
- **Consistent Builds**: Same environment every time
- **Version Tracking**: Every commit SHA becomes an image tag
- **Rollback Ready**: Previous versions available on Docker Hub
- **Fast Feedback**: Builds complete in 2-3 minutes

---

##  Application Features

### Customer Store (Store-Front)

- **Product Catalog**: Browse available products with images and prices
- **Shopping Cart**: Add products to cart
- **Order Placement**: Submit orders with instant confirmation
- **Responsive Design**: Works on mobile, tablet, and desktop

### Admin Portal (Store-Admin)

- **Order Dashboard**: Real-time view of all orders
- **Order Management**: Update order status
- **Order History**: Filter and search past orders
- **Product Management**: Add/edit/delete products (via Product-Service)

### Backend Services

**Order-Service:**
- RESTful API for order submission
- Publishes orders to RabbitMQ queue
- Input validation and error handling

**Product-Service:**
- Product CRUD operations
- Catalog management
- Integration ready for AI-generated descriptions

**Makeline-Service:**
- Background worker pattern
- Consumes orders from RabbitMQ
- Persists to MongoDB
- Order status updates

---

## Testing & Validation

### Persistence Test (MongoDB)

```bash
# Insert test data
kubectl exec -it mongodb-0 -- mongo orderdb --eval '
db.persistenceTest.insertOne({
  testId: "PERSISTENCE_DEMO",
  message: "This must survive pod deletion",
  timestamp: new Date()
})
'

# Delete MongoDB pod
kubectl delete pod mongodb-0

# Wait for recreation
kubectl wait --for=condition=ready pod/mongodb-0 --timeout=120s

# Verify data survived
kubectl exec -it mongodb-0 -- mongo orderdb --eval '
db.persistenceTest.findOne({testId: "PERSISTENCE_DEMO"})
'
```

**Expected Result:** Test document still exists, proving PersistentVolume works.

### High Availability Test (Replica Set)

```bash
# Check current PRIMARY
kubectl exec -it mongodb-0 -- mongo --eval 'rs.isMaster().primary'

# Simulate PRIMARY node failure
kubectl delete pod mongodb-0

# Wait for failover (automatic)
sleep 20

# Check new PRIMARY
kubectl exec -it mongodb-1 -- mongo --eval 'rs.isMaster().primary'

# Query data (should still work)
kubectl exec -it mongodb-1 -- mongo orderdb --eval 'db.orders.count()'
```

** Expected Result:** New PRIMARY elected, queries work without manual intervention.

### Scaling Test

```bash
# Scale Order-Service to handle more load
kubectl scale deployment order-service --replicas=5

# Verify
kubectl get pods -l app=order-service

# Scale back down
kubectl scale deployment order-service --replicas=1
```

---

##  Monitoring & Troubleshooting

### Service-Specific Debugging

```bash
# View logs
kubectl logs deployment/ORDER_SERVICE_NAME --tail=50

# Live log streaming
kubectl logs -f deployment/ORDER_SERVICE_NAME

# Pod details
kubectl describe pod POD_NAME

# Execute commands in pod
kubectl exec -it POD_NAME -- /bin/sh
```

### Common Issues

**Issue: Makeline-Service crashing**

```bash
# Check logs
kubectl logs deployment/makeline-service

# Common causes:
# 1. MongoDB connection string incorrect
# 2. Replica set not initialized
# 3. RabbitMQ not accessible

# Fix: Verify env vars
kubectl get deployment makeline-service -o yaml | grep -A 20 "env:"
```



**Issue: No external IP for frontends**

```bash
# Check service type
kubectl get services store-front store-admin

# If ClusterIP, change to LoadBalancer
kubectl expose deployment store-front --type=LoadBalancer --port=80 --target-port=8080
kubectl expose deployment store-admin --type=LoadBalancer --port=80 --target-port=8081
```

### RabbitMQ Management UI

```bash
# Port forward to management interface
kubectl port-forward service/rabbitmq 15672:15672

# Open browser: http://localhost:15672
```

---

##  Cleanup



### Delete AKS Cluster

```bash
# Delete cluster (saves costs)
az aks delete \
  --resource-group bestbuy-rg \
  --name bestbuy-aks \
  --yes \
  --no-wait

# Delete resource group (deletes all Azure resources)
az group delete --name bestbuy-rg --yes --no-wait

# Verify deletion
az group list --output table
```

** Warning:** This permanently deletes all data and resources. Only run when project is complete.

---

##  Key Learnings

### Technical Skills Acquired

1. **Microservices Architecture**
   - Designing independent, loosely-coupled services
   - Inter-service communication patterns (HTTP, AMQP)
   - Service discovery and load balancing

2. **Kubernetes Orchestration**
   - Deployments, Services, StatefulSets
   - ConfigMaps, Secrets management
   - PersistentVolumes and storage classes
   - Replica sets and high availability

3. **Message-Driven Architecture**
   - Async processing with RabbitMQ
   - Producer-consumer patterns
   - Queue durability and message persistence

4. **Database High Availability**
   - MongoDB replica sets
   - Automatic failover
   - Data persistence across failures

5. **CI/CD Automation**
   - GitHub Actions workflows
   - Docker multi-platform builds
   - Automated testing and deployment

6. **Cloud Infrastructure**
   - Azure Kubernetes Service (AKS)
   - Azure Managed Disks
   - Load Balancers and networking

### Best Practices Implemented

- **Infrastructure as Code**: All resources defined in YAML
- **Immutable Infrastructure**: Containers never modified, always replaced
- **Twelve-Factor App**: Environment-based configuration, stateless services
- **Health Checks**: Liveness and readiness probes on all services
- **Resource Limits**: CPU/memory constraints prevent resource exhaustion
- **Semantic Versioning**: Git commits tagged as Docker image versions


---

##  Author

**Aryan Rudani**  
Program: Cloud Development & Operations  
Course: CST8915 - Full-stack Cloud-native Development  
Institution: Algonquin College

---

##  Acknowledgments

- Based on [Algonquin Pet Store (On Steroids)](https://github.com/ramymohamed10/algonquin-pet-store-on-steroids) by Professor Ramy Mohamed
- Azure Kubernetes Service documentation and best practices


## AI Disclosure
- AI tool `Claude` was used to debug code, fix errors, understanding concepts, Modifying Frontend UI, modifying certain parts for organizing current `readme` file.