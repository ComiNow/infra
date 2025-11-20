# ComiNow - Infraestructura Kubernetes

Este repositorio contiene toda la configuración de infraestructura para desplegar ComiNow en AWS EKS con GitOps usando ArgoCD.

## Estructura del Proyecto

```
infra/
├── k8s/                              # Manifiestos Kubernetes
│   ├── namespace.yaml                # Namespace 'cominow'
│   ├── secrets.yaml                  # Secrets (DB, JWT, APIs)
│   ├── nats-deployment.yaml          # NATS message broker
│   ├── auth-deployment.yaml          # Auth microservice
│   ├── product-deployment.yaml       # Product microservice
│   ├── order-deployment.yaml         # Order microservice
│   ├── payment-deployment.yaml       # Payment microservice
│   ├── customization-deployment.yaml # Customization microservice
│   ├── client-gateway-deployment.yaml# API Gateway
│   ├── ingress.yaml                  # Ingress ALB (opcional)
│   ├── argocd-application.yaml       # ArgoCD config
│   └── README.md                     # Documentación de manifiestos
├── .env                              # Variables consolidadas (NO SUBIR A GIT)
└── docker-compose.prod.yml           # Docker Compose producción
```

## Arquitectura

```
Internet
    ↓
AWS ALB (LoadBalancer)
    ↓
Client Gateway (3 replicas) - Puerto 3000
    ↓
NATS Message Broker - Puerto 4222
    ↓
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│    Auth     │   Product   │    Order    │   Payment   │Customization│
│ Port 3001   │  Port 3002  │  Port 3003  │  Port 3004  │  Port 3005  │
│ 2 replicas  │  2 replicas │  2 replicas │  2 replicas │  2 replicas │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
      │              │              │              │              │
      ↓              ↓              ↓              ↓              ↓
  MongoDB      PostgreSQL     PostgreSQL     PostgreSQL     PostgreSQL
  (Atlas)      (AWS RDS)      (AWS RDS)      (AWS RDS)      (AWS RDS)
```

## Despliegue Rápido

### Opción 1:

```powershell
# Despliegue completo
.\deploy.ps1 -Component all

# Desplegar solo componentes específicos
.\deploy.ps1 -Component secrets
.\deploy.ps1 -Component nats
.\deploy.ps1 -Component microservices
.\deploy.ps1 -Component gateway
```

### Opción 2:

```powershell
# 1. Namespace y Secrets
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets.yaml

# 2. NATS (primero, todos dependen de él)
kubectl apply -f k8s/nats-deployment.yaml

# 3. Microservicios
kubectl apply -f k8s/auth-deployment.yaml
kubectl apply -f k8s/product-deployment.yaml
kubectl apply -f k8s/order-deployment.yaml
kubectl apply -f k8s/payment-deployment.yaml
kubectl apply -f k8s/customization-deployment.yaml

# 4. Gateway
kubectl apply -f k8s/client-gateway-deployment.yaml

```

## Prerequisitos

### Herramientas Necesarias

- [AWS CLI](https://aws.amazon.com/cli/) v2.x
- [kubectl](https://kubernetes.io/docs/tasks/tools/) v1.28+
- [eksctl](https://eksctl.io/) v0.167+
- [Helm](https://helm.sh/) v3.x (para ArgoCD)

### Configuración AWS

```powershell
# Configurar credenciales
aws configure
# AWS Access Key ID: [tu-access-key]
# AWS Secret Access Key: [tu-secret-key]
# Default region: us-east-2
# Default output format: json

# Verificar
aws sts get-caller-identity
```

## Crear Cluster EKS

```powershell
eksctl create cluster `
  --name cominow-cluster `
  --region us-east-2 `
  --nodegroup-name cominow-nodes `
  --node-type t3.medium `
  --nodes 3 `
  --nodes-min 2 `
  --nodes-max 4 `
  --managed

# Configurar kubectl
aws eks update-kubeconfig --region us-east-2 --name cominow-cluster

# Verificar
kubectl get nodes
```

```powershell
kubectl apply -f k8s/secrets.yaml
kubectl get secrets -n cominow
```

## Verificar Despliegue

```powershell
# Ver todos los recursos
kubectl get all -n cominow

# Ver estado de pods
kubectl get pods -n cominow -o wide

# Ver logs de un servicio
kubectl logs -n cominow -l app=client-gateway --tail=50

# Ver eventos
kubectl get events -n cominow --sort-by='.lastTimestamp'
```

## Obtener URL del API

```powershell
# Obtener URL del LoadBalancer
kubectl get svc client-gateway -n cominow

# EXTERNAL-IP mostrará el DNS del ALB
# Ejemplo: a1234567890abcd-1234567890.us-east-2.elb.amazonaws.com

# Probar
curl http://<EXTERNAL-IP>/health
```

## ArgoCD (GitOps)

### Instalar ArgoCD

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Obtener contraseña inicial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Acceder a UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abre: https://localhost:8080

- Usuario: `admin`
- Password: (obtenido arriba)

### Configurar Application

```powershell
# Actualizar argocd-application.yaml con tu repo
# Reemplazar: https://github.com/YOUR-ORG/infra.git

kubectl apply -f k8s/argocd-application.yaml
```

ArgoCD sincronizará automáticamente los cambios del repositorio.

## Actualizaciones

### Con ArgoCD (Automático)

1. Hacer cambios en el código del microservicio
2. GitHub Actions construye y pushea imagen a ECR
3. Actualizar tag en `k8s/<servicio>-deployment.yaml`
4. Commit y push al repo infra
5. ArgoCD detecta cambios y despliega automáticamente

### Manual

```powershell
# Actualizar imagen
kubectl set image deployment/client-gateway client-gateway=515966498907.dkr.ecr.us-east-2.amazonaws.com/cominow/client-gateway:nuevo-tag -n cominow

# Ver progreso
kubectl rollout status deployment/client-gateway -n cominow
```

## Rollback

```powershell
# Ver historial
kubectl rollout history deployment/client-gateway -n cominow

# Rollback a versión anterior
.\rollback.ps1 -Service client-gateway

# Rollback a versión específica
.\rollback.ps1 -Service client-gateway -Revision 3

# Rollback de todos los servicios
.\rollback.ps1 -Service all
```

## Escalado

```powershell
# Escalar manualmente
kubectl scale deployment client-gateway -n cominow --replicas=5

# Ver réplicas
kubectl get deployment client-gateway -n cominow
```

### Horizontal Pod Autoscaler (HPA)

```powershell
# Crear HPA (escalar según CPU)
kubectl autoscale deployment client-gateway -n cominow --cpu-percent=70 --min=3 --max=10

# Ver HPA
kubectl get hpa -n cominow
```

## Troubleshooting

### Pods en CrashLoopBackOff

```powershell
kubectl logs -n cominow <pod-name>
kubectl describe pod -n cominow <pod-name>
```

### Conectividad entre servicios

```powershell
# Pod temporal para debug
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n cominow -- sh

# Dentro del pod:
curl http://nats-server:4222
curl http://auth-microservice:3001/health
```

### Secrets no encontrados

```powershell
kubectl get secrets -n cominow
kubectl describe secret cominow-secrets -n cominow
```

## Eliminar Todo

```powershell
# Eliminar namespace (todos los recursos)
kubectl delete namespace cominow

# Eliminar cluster EKS
eksctl delete cluster --name cominow-cluster --region us-east-2
```

## Referencias Útiles

- [AWS EKS Docs](https://docs.aws.amazon.com/eks/)
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

## Comandos Kubernetes Útiles (Referencia Rápida)

### Gestión de Recursos

```powershell
# Obtener recursos
kubectl get pods -n cominow
kubectl get deployments -n cominow
kubectl get services -n cominow
kubectl get all -n cominow

# Describir recursos
kubectl describe pod <nombre> -n cominow
kubectl describe deployment <nombre> -n cominow

# Logs
kubectl logs <pod-name> -n cominow
kubectl logs -f <pod-name> -n cominow  # follow
kubectl logs -l app=client-gateway -n cominow --tail=100

# Eliminar recursos
kubectl delete pod <nombre> -n cominow
kubectl delete deployment <nombre> -n cominow
```

### Secrets

```powershell
# Crear secret
kubectl create secret generic mi-secret -n cominow --from-literal=key=value

# Múltiples valores
kubectl create secret generic mi-secret -n cominow `
  --from-literal=key1=value1 `
  --from-literal=key2=value2

# Ver secrets
kubectl get secrets -n cominow
kubectl get secret cominow-secrets -n cominow -o yaml

# Editar secret
kubectl edit secret cominow-secrets -n cominow
```

### Deployments y Services

```powershell
# Crear deployment
kubectl create deployment mi-app -n cominow --image=mi-imagen:tag --dry-run=client -o yaml > deployment.yml

# Crear service ClusterIP (interno)
kubectl create service clusterip mi-servicio -n cominow --tcp=8080 --dry-run=client -o yaml > service.yml

# Crear service NodePort (externo)
kubectl create service nodeport mi-servicio -n cominow --tcp=3000 --dry-run=client -o yaml > service.yml
```

### Debug y Testing

```powershell
# Port forward
kubectl port-forward -n cominow svc/client-gateway 3000:3000

# Ejecutar comando en pod
kubectl exec -it <pod-name> -n cominow -- sh

# Copiar archivos
kubectl cp <pod-name>:/path/to/file ./local-file -n cominow
kubectl cp ./local-file <pod-name>:/path/to/file -n cominow

# Ver uso de recursos
kubectl top nodes
kubectl top pods -n cominow
```
