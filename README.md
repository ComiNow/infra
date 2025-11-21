# ComiNow - Infraestructura Kubernetes

Infraestructura para desplegar ComiNow en AWS EKS con GitOps usando ArgoCD.

## URLs de Producción

- **API Gateway (CloudFront)**: https://d1rubx5h2u6ukl.cloudfront.net
- **Payment Webhook (CloudFront)**: https://d1fnxwfvihz944.cloudfront.net
- **Client Gateway (LoadBalancer)**: http://k8s-cominow-clientga-487d3ec347-410239c1ef728475.elb.us-east-2.amazonaws.com
- **Payment Webhook (LoadBalancer)**: http://k8s-cominow-paymentm-a14a871b54-df85504235616a59.elb.us-east-2.amazonaws.com
- **ArgoCD UI**: https://a998dbdf32e4843f397535fad6fdecb4-1662659382.us-east-2.elb.amazonaws.com
  - Usuario: `admin`
  - Password: `zFquhoaw67OczPya`
- **Frontend Cliente**: https://d3gwsdg49ynx4o.cloudfront.net
- **Frontend Empleado**: https://d2wez1qp46w24p.cloudfront.net

## Información del Cluster

- **Nombre**: cominow-cluster
- **Región**: us-east-2
- **Nodes**: 3x t3.medium
- **Namespace**: cominow

## Arquitectura

```
Internet
    ↓
AWS NLB (LoadBalancer)
    ↓
Client Gateway (3 replicas)
    ↓
NATS Message Broker
    ↓
┌──────────┬──────────┬──────────┬──────────┬──────────────┐
│   Auth   │ Product  │  Order   │ Payment  │Customization │
│2 replicas│2 replicas│2 replicas│2 replicas│  2 replicas  │
└──────────┴──────────┴──────────┴──────────┴──────────────┘
     │          │          │          │            │
     ↓          ↓          ↓          ↓            ↓
  MongoDB   PostgreSQL PostgreSQL PostgreSQL  PostgreSQL
  (Atlas)   (AWS RDS)  (AWS RDS)  (AWS RDS)   (AWS RDS)
```

## Estructura del Proyecto

```
infra/
├── k8s/
│   ├── namespace.yaml
│   ├── secrets.yaml
│   ├── nats-deployment.yaml
│   ├── auth-deployment.yaml
│   ├── product-deployment.yaml
│   ├── order-deployment.yaml
│   ├── payment-deployment.yaml
│   ├── customization-deployment.yaml
│   ├── client-gateway-deployment.yaml
│   ├── ingress.yaml
│   └── argocd-application.yaml
└── K8s.README.md
```

## CI/CD Automático

El pipeline está completamente automatizado:

1. Push a cualquier microservicio en GitHub
2. GitHub Actions construye imagen con tag SHA del commit
3. Sube imagen a Amazon ECR
4. Actualiza el deployment YAML en repo infra
5. ArgoCD detecta el cambio (máximo 3 minutos)
6. Despliega automáticamente los nuevos pods

**No se requiere intervención manual para deployments.**

## Comandos Esenciales

### Ver Estado del Cluster

```powershell
# Ver todos los pods
kubectl get pods -n cominow

# Ver todos los recursos
kubectl get all -n cominow

# Ver estado de ArgoCD
kubectl get application cominow -n argocd
```

### Obtener URL del LoadBalancer

```powershell
kubectl get svc client-gateway -n cominow
```

### Ver Logs

```powershell
# Logs de un servicio específico
kubectl logs -n cominow -l app=client-gateway --tail=50

# Seguir logs en tiempo real
kubectl logs -n cominow -l app=client-gateway -f
```

### Verificar Deployments

```powershell
# Ver estado de un deployment
kubectl get deployment client-gateway -n cominow

# Ver historial de rollouts
kubectl rollout history deployment/client-gateway -n cominow

# Ver progreso de un rollout
kubectl rollout status deployment/client-gateway -n cominow
```

### Rollback

```powershell
# Rollback a versión anterior
kubectl rollout undo deployment/client-gateway -n cominow

# Rollback a versión específica
kubectl rollout undo deployment/client-gateway -n cominow --to-revision=2
```

### Escalar Servicios

```powershell
# Escalar réplicas
kubectl scale deployment client-gateway -n cominow --replicas=5

# Ver estado del escalado
kubectl get deployment client-gateway -n cominow
```

### Debug

```powershell
# Describir un pod
kubectl describe pod <pod-name> -n cominow

# Ejecutar comando en un pod
kubectl exec -it <pod-name> -n cominow -- sh

# Ver eventos recientes
kubectl get events -n cominow --sort-by='.lastTimestamp'
```

## Conectar al Cluster

```powershell
# Configurar kubectl
aws eks update-kubeconfig --region us-east-2 --name cominow-cluster

# Verificar conexión
kubectl get nodes
```

## ArgoCD

ArgoCD está configurado para sincronización automática. Puedes monitorear los deployments desde la UI o con:

```powershell
# Ver aplicaciones
kubectl get applications -n argocd

# Forzar sincronización manual
kubectl patch application cominow -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# Ver estado detallado
kubectl describe application cominow -n argocd
```

## Repositorios

- **Infra**: https://github.com/ComiNow/infra
- **Microservicios**:
  - Auth: https://github.com/ComiNow/auth-microservice
  - Client Gateway: https://github.com/ComiNow/client-gateway
  - Product: https://github.com/ComiNow/product-microservice
  - Order: https://github.com/ComiNow/order-microservice
  - Payment: https://github.com/ComiNow/payment-microservice
  - Customization: https://github.com/ComiNow/customization-microservice

## Notas Importantes

- Los deployments usan tags SHA del commit para versionado
- El LoadBalancer URL es permanente y no cambia con los deployments
- ArgoCD sincroniza cambios cada 3 minutos automáticamente
- Todos los secrets están almacenados en Kubernetes Secrets
- Las bases de datos están en servicios externos (MongoDB Atlas y AWS RDS)
