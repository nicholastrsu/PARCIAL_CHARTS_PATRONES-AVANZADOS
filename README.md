# Pedido App – Helm Chart + ArgoCD (Frontend + Backend + PostgreSQL)

Repositorio para desplegar **Pedido App** en Kubernetes usando **Helm** (manual) o **ArgoCD** (GitOps).  
Incluye **frontend**, **backend (FastAPI)**, **PostgreSQL (Bitnami)**, **Ingress (nginx)** y **HPA** para el backend.

---

## Contenido del chart

- **Chart**: `pedido-app/`
  - `Chart.yaml`  
    - `dependencies`: `bitnami/postgresql` `15.5.22`
  - `values.yaml`  

    - Imágenes:  
      - Backend: `dani008/fastapi-backend:1.0.0`  
      - Frontend: `dani008/frontend:1.0.0`

    - Servicios:  
      - Backend: `ClusterIP:8080`  
      - Frontend: `ClusterIP:80`

    - Ingress: `enabled: true`, `host: pedido.local`, `ingressClassName: nginx`

    - PostgreSQL: `enabled: true`  
      - `auth.username: patrones_1parcial`  
      - `auth.password: parcial1`  
      - `auth.postgresPassword: parcial1`  
      - `auth.database: patrones`

  - Plantillas:
    - `deployment.yaml` (backend + frontend)
    - `service.yaml` (svc backend + frontend)
    - `ingress.yaml` (host `pedido.local`, paths `/` → frontend, `/api` → backend)
    - `configmap-backend.yaml` (DB_NAME/DB_HOST/DB_PORT/DATABASE_URL)
    - `secrets.yaml` (Secret `postgres` con `POSTGRES_USER` y `POSTGRES_PASSWORD`)
    - `hpa_backend.yaml` (HPA backend: 2–5 réplicas, 70% CPU)

- **ArgoCD Application**: `application.yaml`  
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: pedido-app-dev
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://github.com/nicholastrsu/PARCIAL_CHARTS_PATRONES-AVANZADOS.git
      targetRevision: HEAD
      path: pedido-app
      helm:
        valueFiles:
          - values.yaml
          - values-dev.yaml
    destination:
      server: https://kubernetes.default.svc
      namespace: pedido-dev
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
        - CreateNamespace=true

## Instalacion manual (Helm)

 1) Clonar y entrar al chart
git clone https://github.com/nicholastrsu/PARCIAL_CHARTS_PATRONES-AVANZADOS.git
cd PARCIAL_CHARTS_PATRONES-AVANZADOS/pedido-app

 2) Preparar dependencias (Bitnami PostgreSQL)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency update

 3) Instalar en el namespace 'pedido-dev'
helm install pedido-app . -n pedido-dev --create-namespace

 4) Agregar DNS local del Ingress
reemplaza <IP_INGRESS> por la IP de tu controlador (minikube ip o LB)
Ejemplo minikube: echo "$(minikube ip) pedido.local" | sudo tee -a /etc/hosts
Ejemplo genérico:
echo "<IP_INGRESS> pedido.local" | sudo tee -a /etc/hosts

## Instalación con ArgoCD

Aplicar la Application
kubectl apply -f application.yaml -n argocd

## Comandos utiles:

Ver estado general:
kubectl get pods,svc,ingress,hpa -n pedido-dev

Ver logs del backend / frontend:
kubectl logs -n pedido-dev deploy/backend -f
kubectl logs -n pedido-dev deploy/frontend -f

## Preparar dependencias (Bitnami PostgreSQL)

helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency update

## Instalar en el namespace pedido-dev
helm install pedido-app . -n pedido-dev --create-namespace

## Configurar DNS para el ingress
kubectl get svc -n ingress-nginx

## Otros comandos utiles
kubectl get events -n pedido-dev
kubectl describe ingress pedido-ingress -n pedido-dev

# Estado general
kubectl get pods,svc,ingress,hpa -n pedido-dev

# Logs del backend
kubectl logs -n pedido-dev deploy/backend -f

# Logs del frontend
kubectl logs -n pedido-dev deploy/frontend -f



## Diagrama

<img width="1024" height="1536" alt="image" src="https://github.com/user-attachments/assets/9e7221f1-7743-42fd-96a3-d9a2699e3384" />

# Ingress

- Es la puerta de entrada a la aplicación desde internet.

- Se encarga de recibir las solicitudes HTTP/HTTPS y enrutar el tráfico hacia los servicios adecuados dentro del clúster de Kubernetes, en este caso hacia el Backend.

# Backend

- Es la capa que procesa la lógica de negocio de tu aplicación.

- Recibe las solicitudes del Ingress y las pasa al FastAPI, que actúa como la API principal.

# FastAPI

- Es el framework web que sirve como API REST para tu aplicación.

- Se encarga de manejar las rutas, controladores y la interacción con la base de datos.

- Está conectado a PostgreSQL, que es donde se almacenan los datos de la aplicación.

# PostgreSQL

- Base de datos relacional que almacena la información de la aplicación (usuarios, pedidos, datos, etc.).

- FastAPI interactúa con PostgreSQL para crear, leer, actualizar y eliminar datos (CRUD).

# HPA (Horizontal Pod Autoscaler)

- Es un recurso de Kubernetes que ajusta automáticamente la cantidad de pods de FastAPI según la carga de trabajo.

- Garantiza que la aplicación pueda escalar horizontalmente si hay más tráfico y reducir pods si hay menos tráfico, optimizando recursos.

  ## Variables de entorno y configuración

La aplicación utiliza **ConfigMap** y **Secret** para manejar la configuración y las credenciales de la base de datos de manera segura.

### 1. ConfigMap (`backend-config`)
Contiene información de configuración no sensible:

- `DB_NAME` → nombre de la base de datos.
- `DB_HOST` → host o servicio de la base de datos en Kubernetes.
- `DB_PORT` → puerto de conexión (por defecto `5432`).

### 2. Secret (`postgres`)
Contiene credenciales sensibles en base64:

- `DB_USER` → usuario de la base de datos.
- `DB_PASSWORD` → contraseña del usuario.

### 3. DATABASE_URL
La variable de entorno `DATABASE_URL` se construye combinando los valores del ConfigMap y del Secret:

```text
postgresql://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)


+-----------------+       +-----------------+
|   ConfigMap     |       |      Secret     |
| DB_NAME         |       | DB_USER         |
| DB_HOST         |       | DB_PASSWORD     |
| DB_PORT         |       +-----------------+
+--------+--------+                |
         |                         |
         |                         |
         |                         v
         +---------------------> DATABASE_URL
                                 |
                                 v
                           +----------------+
                           |   Backend App  |
                           +----------------+





