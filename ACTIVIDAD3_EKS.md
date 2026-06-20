# Actividad 3 - Despliegue seguro en Kubernetes (EKS)

## 1) Requisito previo

- Cluster EKS activo con 2 nodos worker.
- kubectl configurado contra el cluster.
- AWS CLI autenticado.

Comandos base:

```bash
aws eks update-kubeconfig --region <REGION> --name <NOMBRE_CLUSTER>
kubectl get nodes -o wide
```

Evidencia criterio 1:

```bash
kubectl get nodes
```

Debe mostrar 2 nodos en estado Ready.

## 2) Imagenes de contenedor (frontend y backend)

Si necesitas regenerar imagenes y publicarlas en Docker Hub:

```bash
# Desde la raiz del repo FinTech-App-Unir-main-NGuaillas

docker build -t nathyguaillas/fintech-backend:v2 ./backend
docker push nathyguaillas/fintech-backend:v2

docker build -t nathyguaillas/fintech-frontend:v2 ./frontend
docker push nathyguaillas/fintech-frontend:v2
```

Si publicas tags nuevos (por ejemplo v2), actualiza las imagenes en:

- kubernetes/04-backend.yaml
- kubernetes/05-frontend.yaml

## 3) Despliegue Kubernetes completo

Aplicar todos los manifiestos:

```bash
kubectl apply -k kubernetes
```

Archivos incluidos:

- 01-namespace.yaml: namespace fintech

- 02-security-config.yaml: Secret para credenciales de base de datos
- 03-database.yaml: Service DB + PV estatico + PVC + StatefulSet
- 04-backend.yaml: Deployment + Service ClusterIP
- 05-frontend.yaml: Deployment + Service LoadBalancer

## 4) Verificacion de cargas de trabajo (criterio 2)

```bash
kubectl get deployments -n fintech
kubectl get pods -n fintech -o wide
kubectl describe deployment backend -n fintech
kubectl describe deployment frontend -n fintech
kubectl get statefulset -n fintech
kubectl get pvc -n fintech
kubectl get pv
```

Resultado esperado:

- frontend y backend con replicas disponibles.
- StatefulSet db en Running.
- PVC Bound.
- PV estatico db-static-pv en estado Bound.

## 5) Verificacion de servicios (criterio 3)

```bash
kubectl get svc -n fintech
```

Tipos esperados:

- frontend: LoadBalancer
- backend: ClusterIP
- db: ClusterIP

Obtener URL de acceso:

```bash
kubectl get svc frontend -n fintech
```

Acceso desde navegador:

- usar el DNS en EXTERNAL-IP del servicio frontend

Validaciones:

- Frontend puede listar/crear usuarios.
- Frontend puede crear transacciones.
- Backend responde healthcheck.

Healthcheck backend:

```bash
kubectl port-forward svc/backend 3001:3001 -n fintech
curl http://localhost:3001/api/health
```

## 6) Seguridad (Secret)

```bash
kubectl get secrets -n fintech
kubectl describe secret fintech-db-secret -n fintech
```

Puntos de seguridad aplicados:

- Credenciales DB en Secret.
- Backend y DB con servicios internos ClusterIP.

## 7) Persistencia (PV y PVC)

La persistencia se define con un PV estatico y un PVC pre-creado.
Este enfoque evita depender de permisos IAM adicionales para el driver EBS CSI en AWS Academy.

Evidencias:

```bash
kubectl get pvc -n fintech
kubectl get pv
kubectl describe pvc db-data-pvc -n fintech
```

Prueba de persistencia sugerida:

1. Crear un usuario desde la UI.
2. Reiniciar pod de la DB:

```bash
kubectl delete pod db-0 -n fintech
```

3. Verificar que los datos siguen disponibles.

## 8) Checklist de capturas para el informe final

- EKS cluster en estado Active (consola AWS).
- kubectl get nodes con 2 workers Ready.
- kubectl get deployments.
- kubectl get pods.
- kubectl describe deployment backend.
- kubectl get svc.
- kubectl get secrets.
- kubectl get pv y kubectl get pvc.
- Aplicacion abierta en navegador funcionando.

## 9) Estructura recomendada para el documento final

### Introduccion

- Objetivo: desplegar aplicacion financiera segura en EKS.
- Entorno: EKS, Docker Hub, kubectl, AWS CLI.

### Arquitectura

Internet
  |
AWS Load Balancer (Service tipo LoadBalancer)
  |
Frontend
  |
Backend (ClusterIP)
  |
PostgreSQL (StatefulSet + PVC)

EKS
- Worker Node 1
- Worker Node 2

### Desarrollo

- Creacion del cluster.
- Configuracion de nodos.
- Despliegue de frontend, backend y base de datos.
- Configuracion de servicios.
- Seguridad con Secret.
- Persistencia con PV y PVC estaticos.

### Resultados

- Evidencias de comandos kubectl.
- Capturas de aplicacion en ejecucion.

### Conclusiones

- Kubernetes mejora disponibilidad y escalabilidad.
- Uso de Secret mejora el manejo de credenciales sensibles.

## 10) Explicacion de cada YAML

### 01-namespace.yaml

- Crea el namespace fintech.
- Sirve para aislar los recursos de esta aplicacion del resto del cluster.

### 02-security-config.yaml

- Crea un Secret llamado fintech-db-secret.
- Guarda usuario y contrasena de PostgreSQL para no dejarlos en texto plano en los Deployments.

### 03-database.yaml

- Crea el Service db de tipo ClusterIP para acceso interno en el cluster.
- Crea un PersistentVolume estatico llamado db-static-pv.
- Crea un PersistentVolumeClaim llamado db-data-pvc que se enlaza al PV estatico.
- Crea el StatefulSet db (1 replica) con imagen postgres:15-alpine.
- Monta el PVC db-data-pvc en /var/lib/postgresql/data para persistencia.

### 04-backend.yaml

- Crea el Deployment backend (1 replica) con imagen nathyguaillas/fintech-backend:v2.
- Inyecta variables de entorno para conectar con la base de datos.
- Crea el Service backend de tipo ClusterIP para consumo interno desde frontend/nginx.

### 05-frontend.yaml

- Crea el Deployment frontend (1 replica) con imagen nathyguaillas/fintech-frontend:v2.
- Crea el Service frontend de tipo LoadBalancer para acceso desde Internet por DNS del balanceador de AWS.

### kustomization.yaml

- Agrupa todos los YAML anteriores para aplicar todo en un solo comando.
- Permite desplegar con kubectl apply -k kubernetes.

## 11) Puertos y grupos de seguridad en EKS

Para este despliegue minimo:

- El Service tipo LoadBalancer crea el balanceador en AWS automaticamente.
- El acceso de usuarios va por el DNS que aparece en EXTERNAL-IP del servicio frontend.
- Backend (3001) y DB (5432) no se abren a internet porque van por servicios ClusterIP internos.

Reglas practicas recomendadas:

- Security Group del Load Balancer: permitir 80/TCP desde el origen esperado (por ejemplo 0.0.0.0/0 si es publico).
- Admin/Bastion SG: solo SSH/RDP segun uses, y salida HTTPS para operar kubectl/aws cli.

## 12) Pods esperados

Con los manifiestos actuales:

- frontend: 1 pod.
- backend: 1 pod.
- db (StatefulSet): 1 pod.

Total de la aplicacion: 3 pods (sin contar pods de sistema de Kubernetes en kube-system).
