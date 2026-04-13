# Kubernetes Microservices Demo Guide

This guide demonstrates the implementation of a microservices architecture deployed on Kubernetes, covering namespace configuration, service deployment, persistent storage, and StatefulSets.

## Architecture Overview

The system consists of:
- **Resources Service**: Handles audio file uploads and metadata
- **Songs Service**: Manages song metadata
- **PostgreSQL Databases**: Separate databases for each service using StatefulSets

## Sub-task 2: Deploy Containers in Kubernetes

### What was implemented:

1. **Namespace**: `k8s-program` - isolated environment for all resources
2. **Services**: 
   - `resources-service` (NodePort on 8080)
   - `songs-service` (NodePort on 8081)
3. **Deployments**: 
   - Both services running with 2 replicas for high availability
   - Environment variables configured via ConfigMap and deployment specs

### Demo Steps:

```bash
# 1. View all deployed resources
kubectl get all -n k8s-program

# Expected output shows:
# - Pods (4 app pods + 2 database pods)
# - Services (2 NodePort services + 2 ClusterIP database services)
# - Deployments (2 application deployments)
# - ReplicaSets (manages pod replicas)
# - StatefulSets (2 database instances)
```

**Why do we see Pods and ReplicaSets?**
- **Pods**: The actual running containers created by Deployments
- **ReplicaSets**: Created automatically by Deployments to maintain desired replica count and handle rolling updates

```bash
# 2. Check namespace
kubectl get namespaces
# Shows k8s-program namespace

# 3. View services details
kubectl get svc -n k8s-program
# Shows NodePort services with external ports for testing
```

## Sub-task 3: Persistent Volumes

### What was implemented:

1. **PersistentVolume** (`songs-pv`):
   - Storage class: `manual`
   - Storage: 1Gi
   - HostPath: `/mnt/data/songs-service`

2. **PersistentVolumeClaim** (`songs-pvc`):
   - Bound to songs-pv
   - Referenced in Songs deployment

### Demo Steps:

```bash
# 1. View PersistentVolumes and Claims
kubectl get pv
kubectl get pvc -n k8s-program

# 2. Test persistence - Create a test file in the volume
SONG_POD=$(kubectl get pods -n k8s-program -l app=songs-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n k8s-program $SONG_POD -- sh -c 'echo "Test persistence - $(date)" > /var/lib/postgresql/data/test-file.txt'

# 3. Verify file exists
kubectl exec -n k8s-program $SONG_POD -- cat /var/lib/postgresql/data/test-file.txt

# 4. Delete the pod to test persistence
kubectl delete pod -n k8s-program $SONG_POD

# 5. Wait for ReplicaSet to create new pod
kubectl get pods -n k8s-program -l app=songs-service -w

# 6. Verify file still exists in new pod
NEW_POD=$(kubectl get pods -n k8s-program -l app=songs-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n k8s-program $NEW_POD -- cat /var/lib/postgresql/data/test-file.txt
# File content should match - data persisted!
```

## Sub-task 4: StatefulSets

### What was implemented:

1. **Database StatefulSets**:
   - `resources-db`: PostgreSQL for Resources Service
   - `songs-db`: PostgreSQL for Songs Service
   - Storage class: `standard` (default in Minikube with hostpath provisioner)
   - Automatic PV provisioning via volumeClaimTemplates

2. **Database Services** (ClusterIP):
   - `resources-db`: Internal only access
   - `songs-db`: Internal only access

### Demo Steps:

```bash
# 1. View StatefulSets
kubectl get statefulsets -n k8s-program
# Shows resources-db and songs-db with 1/1 ready

# 2. View automatically provisioned PVCs
kubectl get pvc -n k8s-program
# Shows postgres-storage-resources-db-0 and postgres-storage-songs-db-0
# Storage class: standard (automatic provisioning)

# 3. View automatically created PersistentVolumes
kubectl get pv
# Shows dynamically provisioned volumes with minikube-hostpath provisioner

# 4. Test database StatefulSet persistence
kubectl exec -n k8s-program songs-db-0 -- sh -c 'echo "DB Test - $(date)" > /var/lib/postgresql/data/pgdata/db-test.txt'
kubectl exec -n k8s-program songs-db-0 -- cat /var/lib/postgresql/data/pgdata/db-test.txt

# 5. Delete StatefulSet pod
kubectl delete pod -n k8s-program songs-db-0

# 6. Wait for StatefulSet to recreate pod with same name
kubectl get pods -n k8s-program -l app=songs-db -w

# 7. Verify data persists (StatefulSet maintains same PVC binding)
kubectl exec -n k8s-program songs-db-0 -- cat /var/lib/postgresql/data/pgdata/db-test.txt
# File still exists!

# 8. Verify ClusterIP services (no external access)
kubectl get svc -n k8s-program
# Database services show "ClusterIP: None" (headless) or specific IP
# No NodePort - only accessible within cluster

# 9. Test port-forward for database access (temporary)
kubectl port-forward -n k8s-program pod/songs-db-0 5433:5432
# Now you can connect to PostgreSQL at localhost:5433
# Press Ctrl+C to stop port-forwarding
```

## Testing the Application

### Access services via Minikube

```bash
# Get service URLs
minikube service -n k8s-program resources-service --url
minikube service -n k8s-program songs-service --url

# Or open in browser automatically
minikube service -n k8s-program resources-service
minikube service -n k8s-program songs-service
```

### API Endpoints:

**Songs Service** (port 8081):
- `GET /songs/{id}` - Get song by ID
- `POST /songs` - Create new song (JSON body required)

**Resources Service** (port 8080):
- `POST /resources` - Upload audio file (audio/mpeg)
- `GET /resources/{id}` - Download audio file
- `DELETE /resources?id=1,2,3` - Delete resources by IDs

### Test with curl:

```bash
# Get the service URL
SONGS_URL=$(minikube service -n k8s-program songs-service --url)
RESOURCES_URL=$(minikube service -n k8s-program resources-service --url)

# Test songs service (create song)
curl -X POST $SONGS_URL/songs \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Song","artist":"Test Artist","album":"Test Album","length":"3:45","resourceId":"1"}'

# Test songs service (get song)
curl $SONGS_URL/songs/1
```

## Key Concepts Demonstrated

### 1. **Why we see Pods and ReplicaSets with `kubectl get all`:**
- **Deployments** create **ReplicaSets**
- **ReplicaSets** create and manage **Pods**
- ReplicaSets ensure the desired number of pod replicas are running
- During updates, new ReplicaSets are created for rolling updates

### 2. **PersistentVolume vs PersistentVolumeClaim:**
- **PV**: Actual storage resource (manual creation or dynamic provisioning)
- **PVC**: Request for storage by pods
- Pods reference PVCs, not PVs directly

### 3. **StatefulSet vs Deployment:**
- **StatefulSet**: Maintains pod identity, stable network IDs, ordered scaling
- **Deployment**: Stateless, pods are interchangeable
- StatefulSets perfect for databases (persistent identity + storage)

### 4. **Storage Classes:**
- `manual`: Requires pre-created PersistentVolumes
- `standard` (Minikube default): Automatic provisioning via hostpath provisioner
- volumeClaimTemplates in StatefulSets create unique PVCs per pod

### 5. **Service Types:**
- **NodePort**: Exposes service externally on each node's IP
- **ClusterIP**: Only accessible within cluster (default, used for databases)
- Headless services (`clusterIP: None`) used with StatefulSets for direct pod access

## Files Structure

```
deployment/
├── deployment.yml      # Namespace, ConfigMap, Services, Deployments
├── databases.yml       # StatefulSets and Services for databases
└── persistence.yml     # PersistentVolumes and PersistentVolumeClaims
```

## Cleanup

```bash
# Delete all resources in namespace
kubectl delete namespace k8s-program

# Or delete specific resources
kubectl delete -f deployment/deployment.yml
kubectl delete -f deployment/databases.yml
kubectl delete -f deployment/persistence.yml
```

## Summary

✅ **Sub-task 2**: Deployed 2 microservices with NodePort access and 2 replicas each  
✅ **Sub-task 3**: Implemented persistent storage with manual PV/PVC for Songs service  
✅ **Sub-task 4**: Created database StatefulSets with automatic storage provisioning and ClusterIP services  

All services are running, data persists across pod restarts, and the system demonstrates proper Kubernetes resource management.
