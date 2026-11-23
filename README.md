# reservas-app — Sistema de Reservas de Salas (Variante B)

Aplicación de tres capas para la gestión de reservas de salas por franja horaria (**React + Vite · Node.js 20 + Express · MySQL 8.4**), contenerizada con **Docker / Docker Compose** y desplegable en **Kubernetes** mediante **12 manifiestos declarativos** y su **Chart de Helm** equivalente.

## Arquitectura

```text
navegador ──► reservas-frontend ──────► reservas-api ──────► reservas-db
              React + Vite + Nginx      Node 20 + Express    MySQL 8.4
              :8080 (Compose :3000)     :3000 (interno)      :3306 (interno)
```

| Capa | Tecnología / Imagen | Puerto interno | Exposición |
| --- | --- | --- | --- |
| `reservas-frontend` | React + Vite servido por `nginxinc/nginx-unprivileged:1.27-alpine` | `8080` | Compose: `3000` / K8s: `NodePort 30301` + `Ingress (reservas.local)` |
| `reservas-api` | Node.js 20 Alpine + Express | `3000` | Solo red interna (`ClusterIP` en K8s) |
| `reservas-db` | `mysql:8.4` | `3306` | Solo red interna (`ClusterIP` en K8s) |

El frontend es el único punto de entrada: Nginx sirve los archivos estáticos de la SPA y actúa como *reverse proxy* reenviando `/api/*` hacia `http://${API_HOST}:${API_PORT}/` (resuelto en tiempo de ejecución mediante `envsubst`).

> **Nota sobre `NGINX_ENVSUBST_FILTER=^API_`:** esta variable declarada en [`frontend/Dockerfile`](./frontend/Dockerfile) es obligatoria. Sin ese filtro, el script de arranque de Nginx sustituiría también variables internas propias de Nginx (`$uri`, `$host`, `$proxy_add_x_forwarded_for`), dejándolas vacías y generando una configuración inválida en silencio.

---

## Estructura del repositorio

```text
.
├── .env.example          # Contrato de variables de entorno de referencia para Docker Compose
├── .gitignore            # Exclusión de node_modules/, .env, dist/ y logs
├── docker-compose.yml    # Orquestación local con red propia, volumen, healthchecks y cap_drop: ALL
├── backend/              # API Node.js 20 + Express + MySQL (Dockerfile multi-etapa, usuario node)
├── frontend/             # SPA React + Vite + Nginx unprivileged (Dockerfile multi-etapa, usuario nginx)
├── k8s/                  # 12 manifiestos individuales de Kubernetes
├── reservas-app/         # Chart de Helm parametrizado (Chart.yaml, values.yaml y 12 templates)
├── informe/              # Capturas de pantalla y evidencias enlazadas desde INFORME.md
├── ARQUITECTURA.md       # Respuestas técnicas, concurrencia, dificultades MySQL y producción
└── INFORME.md            # Demostración de funcionamiento (5 evidencias obligatorias + persistencia)
```

> **Advertencia sobre credenciales:** los archivos `k8s/03-secret-db.yaml`, `reservas-app/values.yaml` y `.env.example` contienen **valores de laboratorio** (`reservas_user` / `reservas_pass` / `reservas_root_pass`) exclusivamente para facilitar la reproducción y evaluación del curso. No incluyen credenciales reales; en [`ARQUITECTURA.md`](./ARQUITECTURA.md) se detalla cómo gestionar secretos en un entorno de producción real.

---

## Inventario de objetos de Kubernetes (12 objetos exactos)

Tanto la carpeta [`k8s/`](./k8s) como el Chart [`reservas-app/`](./reservas-app) definen exactamente los **12 objetos** del proyecto:

| # | Archivo | Tipo (`kind`) | Nombre | Función |
| :---: | --- | --- | --- | --- |
| 1 | `01-cm-api.yaml` | `ConfigMap` | `reservas-api-config` | Configuración no sensible de la API (`DB_HOST`, `DB_PORT`, `DB_NAME`, `PORT`, `SEED_DEMO`; sin `APP_VERSION`). |
| 2 | `02-cm-frontend.yaml` | `ConfigMap` | `reservas-frontend-config` | Configuración del proxy inverso Nginx (`API_HOST`, `API_PORT`). |
| 3 | `03-secret-db.yaml` | `Secret` | `reservas-db-secret` | Credenciales de laboratorio de MySQL consumidas por la base y renombradas por la API. |
| 4 | `04-pvc-db.yaml` | `PersistentVolumeClaim` | `reservas-db-pvc` | Almacenamiento persistente (`1Gi`, `ReadWriteOnce`) para el directorio de datos de MySQL. |
| 5 | `05-svc-db.yaml` | `Service` (`ClusterIP`) | `reservas-db` | Nombre DNS interno estable en puerto `3306` para MySQL. |
| 6 | `06-svc-api.yaml` | `Service` (`ClusterIP`) | `reservas-api` | Balanceo y nombre DNS interno en puerto `3000` para las réplicas de la API. |
| 7 | `07-svc-frontend.yaml` | `Service` (`NodePort`) | `reservas-frontend` | Exposición del frontend en puerto `8080` (`nodePort: 30301`). |
| 8 | `08-deploy-db.yaml` | `Deployment` | `reservas-db` | 1 réplica de `mysql:8.4` montando `reservas-db-pvc` con `subPath: mysql`. |
| 9 | `09-deploy-api.yaml` | `Deployment` | `reservas-api` | 2 réplicas de `slsantiago/reservas-api:v1`, probes `/health` y `resources.requests/limits`. |
| 10 | `10-deploy-frontend.yaml` | `Deployment` | `reservas-frontend` | 2 réplicas de `slsantiago/reservas-frontend:v1` y probes `/healthz`. |
| 11 | `11-ingress.yaml` | `Ingress` | `reservas-app-ingress` | Host `reservas.local` con una sola regla `/` hacia `reservas-frontend:8080`. |
| 12 | `12-hpa-api.yaml` | `HorizontalPodAutoscaler` | `reservas-api-hpa` | Escalado automático de `reservas-api` (`2` a `6` réplicas, objetivo `50%` CPU). |

- **Por qué no se incluye `reservas-api-nodeport`:** fue un Service temporal de depuración de la sexta entrega y no forma parte de la arquitectura final (la API solo es accesible desde dentro del clúster).
- **Por qué el Ingress no define una regla `/api`:** el Nginx del contenedor `reservas-frontend` ya realiza ese proxy inverso; duplicarlo en el Ingress generaría dos fuentes de verdad para el enrutamiento y rompería la paridad con Docker Compose.

---

## Camino 1: Ejecución local con Docker Compose

1. Copiar el archivo de variables de entorno de referencia:
   ```bash
   cp .env.example .env
   ```
2. Validar la configuración y levantar las tres capas:
   ```bash
   docker compose config --quiet
   docker compose up --build -d
   ```
3. Verificar que los tres contenedores alcancen el estado `healthy` y respondan correctamente:
   ```bash
   docker compose ps
   curl -s http://localhost:3000/api/health
   curl -s http://localhost:3000/api/reservas
   ```
4. Abrir la aplicación en el navegador: **http://localhost:3000**

Para detener los contenedores conservando los datos en el volumen `reservas-db-data`:
```bash
docker compose down
```
*(Agregar `-v` solo si se desea eliminar también el volumen persistente de MySQL).*

---

## Construcción y publicación de imágenes (Docker Hub)

Las imágenes publicadas para el clúster corresponden al repositorio `slsantiago`:

```bash
# Construir API v1 y v2 (para probar upgrade/rollback en Kubernetes y Helm)
docker build --build-arg APP_VERSION=v1 -t slsantiago/reservas-api:v1 ./backend
docker build --build-arg APP_VERSION=v2 -t slsantiago/reservas-api:v2 ./backend

# Construir Frontend v1
docker build -t slsantiago/reservas-frontend:v1 ./frontend

# Autenticarse y publicar en Docker Hub
docker login
docker push slsantiago/reservas-api:v1
docker push slsantiago/reservas-api:v2
docker push slsantiago/reservas-frontend:v1
```

---

## Camino 2: Despliegue en Kubernetes con Helm (y manifiestos `k8s/`)

### Requisitos previos
- Clúster de Kubernetes (clúster de la cátedra vía VPN o clúster local como Minikube/k3s) con **NGINX Ingress Controller**, **StorageClass** por defecto y **Metrics Server** operativos.
- Entrada en `/etc/hosts` apuntando `reservas.local` a la `EXTERNAL-IP` del Ingress Controller (o a `minikube ip` en entornos locales):
  ```bash
  # En el clúster de la cátedra:
  INGRESS_IP=$(kubectl get svc -n nginx-ingress nginx-ingress-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo "${INGRESS_IP} reservas.local" | sudo tee -a /etc/hosts
  ```

### Despliegue con Helm (`reservas-app/`)
```bash
# 1. Validar que el chart renderiza exactamente los 12 objetos
helm template reservas ./reservas-app -n reservas-app | grep -c "^kind:"

# 2. Instalar la release
helm install reservas ./reservas-app -n reservas-app --create-namespace
kubectl get all,cm,secret,pvc,ingress,hpa -n reservas-app

# 3. Actualizar la API a v2 (sin modificar el tag del frontend)
helm upgrade reservas ./reservas-app -n reservas-app --set api.image.tag=v2
kubectl rollout status deployment/reservas-api -n reservas-app

# 4. Consultar historial y revertir a la revisión 1
helm history reservas -n reservas-app
helm rollback reservas 1 -n reservas-app
helm history reservas -n reservas-app

# 5. Desinstalar
helm uninstall reservas -n reservas-app
```

### Alternativa: Manifiestos planos (`k8s/`)
*(Si se prueban los manifiestos planos antes de instalar Helm, eliminarlos con `kubectl delete -f k8s/` antes de ejecutar `helm install` para evitar colisión de nombres).*

```bash
kubectl create namespace reservas-app
kubectl apply -f k8s/
kubectl get all,cm,secret,pvc,ingress,hpa -n reservas-app
```

Una vez desplegado, acceder en **http://reservas.local** (o vía `NodePort` en `http://<IP-del-nodo>:30301`).

---

## Endpoints principales de la API

A través del proxy de Nginx (prefijo `/api`):

| Método | Ruta | Descripción |
| --- | --- | --- |
| `GET` | `/api/health` | Estado de la API, conexión a MySQL y `APP_VERSION` |
| `GET` | `/api/franjas` | Bloques horarios disponibles (`1` a `6`, de `08:00` a `20:00`) |
| `GET` | `/api/salas` | Listado de salas |
| `POST` | `/api/salas` | Crear una sala (`nombre`, `capacidad`, `ubicacion`) |
| `DELETE` | `/api/salas/:id` | Eliminar una sala (devuelve `409` si tiene reservas) |
| `GET` | `/api/reservas` | Listar reservas (filtros opcionales: `?fecha=YYYY-MM-DD&sala_id=N`) |
| `POST` | `/api/reservas` | Crear una reserva (devuelve `409` si hay solapamiento por restricción `UNIQUE`) |
| `DELETE` | `/api/reservas/:id` | Cancelar una reserva |

---

## Documentación de entrega

- [**ARQUITECTURA.md**](./ARQUITECTURA.md): Respuestas técnicas del curso, garantía de concurrencia en Variante B, dificultades resueltas con MySQL y consideraciones para producción.
- [**INFORME.md**](./INFORME.md): Evidencias de funcionamiento con comandos y capturas (Ingress, HPA, Helm `history`, Helm `template`, concurrencia y persistencia PVC).
