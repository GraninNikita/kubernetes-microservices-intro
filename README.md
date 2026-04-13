# microservices-intro
Microservices Intro program, pre-requisite for Kubernetes educational program: https://learn.epam.com/catalog/detailsPage?id=550944b4-72c9-4c2d-93ef-545b6e569f61

# Kubernetes and podman commands

## Prerequisites

- `minikube start`

### Sub-task 3

- Show created namespaces:
`kubectl get namespace`

1. Build services images
  `podman build -t resources-service:latest -f resources-service/Dockerfile .`
  `podman build -t songs-service:latest -f songs-service/Dockerfile .`
2. Load images into minikube
  ```podman save localhost/resources-service:latest | (eval $(minikube podman-env) && podman load)
     podman save localhost/songs-service:latest | (eval $(minikube podman-env) && podman load)```.  
3. Deploy with k8s
   `kubectl apply -f deployment.yml`
