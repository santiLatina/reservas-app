# Arquitectura y Decisiones de Diseño — reservas-app (Variante B)

Este documento fundamenta las decisiones técnicas tomadas durante el despliegue de la arquitectura de tres capas (**React + Vite**, **Node.js 20 + Express** y **MySQL 8.4**) sobre Docker Compose y Kubernetes.

---

## 1. Preguntas obligatorias del curso

### 1.1. Configuración del frontend en tiempo de ejecución (`API_HOST`) vs. tiempo de construcción (`VITE_API_URL`)

La SPA de React ([`frontend/src/api.js`](./frontend/src/api.js)) realiza siempre peticiones relativas al mismo origen (`/api/...`) sin conocer la URL ni el puerto del backend. Quien resuelve ese destino es **Nginx** ([`frontend/nginx/templates/default.conf.template`](./frontend/nginx/templates/default.conf.template)), que reemplaza `API_HOST` y `API_PORT` mediante `envsubst` **al arrancar el contenedor**, restringido por `NGINX_ENVSUBST_FILTER=^API_` para no sobrescribir variables internas de Nginx como `$uri` o `$host`.

* **Si se hubiera configurado en tiempo de construcción (`VITE_API_URL`):** Vite incrusta las variables `VITE_*` de forma literal dentro del bundle JavaScript (`dist/`) durante `npm run build`. Para pasar de desarrollo a producción (o de Docker Compose a Kubernetes) habría que **recompilar la aplicación y reconstruir una imagen Docker distinta para cada entorno**, rompiendo la inmutabilidad del artefacto: la misma imagen que pasó pruebas no sería la que llega a producción.

### 1.2. Ausencia de CORS en esta arquitectura

Esta arquitectura no necesita cabeceras **CORS** (*Cross-Origin Resource Sharing*) en Express porque el navegador nunca se comunica directamente con `reservas-api`: tanto el HTML/JS estático (`/`) como las peticiones a la API (`/api/*`) comparten exactamente el **mismo origen** (esquema, host y puerto: `http://localhost:3000` en Compose o `http://reservas.local` en Kubernetes). Nginx actúa como *reverse proxy* interno reenviando `/api/` hacia `http://${API_HOST}:${API_PORT}/` (eliminando el prefijo `/api` gracias a la barra final de `proxy_pass`).

* **Qué habría que agregar si la SPA hablara directamente con la API:** al diferir el puerto o el host (por ejemplo, frontend en `:3000` y API en `:3001`), el navegador aplicaría la política del mismo origen (*Same-Origin Policy*), dispararía solicitudes *preflight* `OPTIONS` y bloquearía las respuestas. Habría que instalar y configurar un middleware CORS en [`backend/server.js`](./backend/server.js) declarando explícitamente `Access-Control-Allow-Origin` para el origen del frontend, los métodos permitidos (`GET`, `POST`, `DELETE`, `OPTIONS`) y cabeceras como `Content-Type`.

### 1.3. Por qué existe un único autoescalador (HPA) entre los doce objetos

El sistema declara un único `HorizontalPodAutoscaler` (`reservas-api-hpa`), que calcula el porcentaje de utilización de CPU respecto de la suma de `resources.requests.cpu` declarada en el Pod (`100m` por réplica) y escala únicamente `reservas-api` (de `2` a `6` réplicas cuando supera el `50%`). La decisión responde a la naturaleza de cada capa:

* **`reservas-api` (Stateless y sensible a CPU):** no almacena estado local entre solicitudes; procesa validaciones, serialización JSON y consultas SQL con `JOIN`. Escalar horizontalmente sus Pods distribuye el tráfico automáticamente detrás del `Service` `ClusterIP` `reservas-api`.
* **`reservas-db` (Stateful — qué ocurriría si se le asignara un HPA):**
  * *Con el PVC `ReadWriteOnce` actual:* los Pods adicionales no podrían montar simultáneamente el mismo volumen si caen en distintos nodos (quedarían bloqueados en `ContainerCreating` por conflicto *Multi-Attach*), y si coincidieran en el mismo nodo múltiples procesos `mysqld` escribiendo sobre el mismo directorio `/var/lib/mysql` corromperían los archivos InnoDB.
  * *Sin volumen compartido (como en un Deployment efímero):* Kubernetes aceptaría el escalado sin error aparente y levantaría varias instancias independientes de MySQL detrás del mismo `Service`, devolviendo datos distintos e inconsistentes en cada petición (*split-brain* silencioso).
* **`reservas-frontend` (Contenido estático y proxy ligero):** Nginx sirve archivos estáticos compilados y reenvía tráfico HTTP con un consumo mínimo de CPU y memoria. Dos réplicas fijas garantizan redundancia y tolerancia a fallos de un Pod sin requerir escalado dinámico por CPU.

---

## 2. Pregunta propia de la Variante B: qué deja de estar garantizado al pasar de una réplica de la API a tres

Al pasar `reservas-api` de una réplica a tres, existen tres procesos Node.js independientes atendiendo solicitudes concurrentes sobre la misma base de datos.

* **Qué deja de estar garantizado en la capa de aplicación:** la consulta previa a la inserción (`SELECT id FROM reservas WHERE sala_id = ? AND fecha = ? AND franja = ?` en [`backend/server.js`](./backend/server.js)) **no sostiene la invariante** de unicidad. Entre el `SELECT` y el `INSERT` existe una ventana de tiempo (agravada por el `await` asíncrono y por haber tres procesos paralelos) durante la cual múltiples peticiones leen simultáneamente que la sala está «libre» y avanzan a insertar la misma reserva (*race condition*).
* **Qué impide efectivamente la doble reserva:** la restricción de unicidad definida a nivel de esquema en el motor MySQL (`CONSTRAINT sin_solapamiento UNIQUE (sala_id, fecha, franja)` en [`backend/db.js`](./backend/db.js)). El motor serializa las inserciones conflictivas mediante bloqueos a nivel de índice único: exactamente una transacción confirma el `INSERT` (`201 Created`) y las restantes fallan atómicamente con el código `ER_DUP_ENTRY`, que `traducirError()` en `backend/server.js` convierte en `409 Conflict`.
* **Evidencia:** la verificación empírica con 3 réplicas de la API, 30 peticiones `POST` simultáneas (`1` respuesta `201`, `29` respuestas `409`) y el recuento `SELECT COUNT(*) = 1` en MySQL se encuentra documentada en [`INFORME.md`](./INFORME.md) (sección 5).

---

## 3. Dificultades propias de MySQL encontradas y resueltas durante el curso

### 3.1. Inicialización en dos tiempos, healthcheck con `-h 127.0.0.1`, escape `$$` y `start_period`

* **Cómo se manifestó:** al orquestar las tres capas en Docker Compose, el contenedor `reservas-api` registraba errores de conexión (`ECONNREFUSED`) o el contenedor `reservas-db` quedaba marcado como `unhealthy` durante el primer arranque, aun teniendo `depends_on` con `condition: service_healthy`.
* **Cómo se diagnosticó:** al inspeccionar `docker logs reservas-db` y `docker inspect reservas-db`, se comprobaron tres factores propios de MySQL 8.4:
  1. La imagen inicializa el directorio de datos levantando primero un servidor **temporal** que solo escucha por socket Unix (`port: 0`, sin red); si el healthcheck usa `-h localhost`, `mysqladmin ping` conecta por socket y reporta `healthy` antes de que exista el usuario y el servidor TCP definitivo (`port: 3306`).
  2. La primera inicialización del `datadir` demora decenas de segundos, por lo que sin período de gracia los reintentos se agotan prematuramente.
  3. Escribir `$MYSQL_PASSWORD` (con un solo `$`) en `docker-compose.yml` provoca que Compose interpole la variable antes de crear el contenedor (dejando la contraseña vacía o expuesta en texto plano en `docker inspect`).
* **Cómo se resolvió:** en [`docker-compose.yml`](./docker-compose.yml) se configuró `mysqladmin ping -h 127.0.0.1 -u$$MYSQL_USER -p"$$MYSQL_PASSWORD" --silent` (forzando TCP contra el servidor definitivo y escapando `$$` para expansión dentro del contenedor) junto con `start_period: 40s` y suficientes reintentos.

### 3.2. Directorio de datos no vacío al montar el PVC en Kubernetes (`subPath: mysql`)

* **Cómo se manifestó:** al asociar el `PersistentVolumeClaim` (`reservas-db-pvc`) al `Deployment` de MySQL en Kubernetes, el Pod `reservas-db` entraba en estado `CrashLoopBackOff` sin llegar a aceptar conexiones.
* **Cómo se diagnosticó:** al ejecutar `kubectl logs deploy/reservas-db -n reservas-app`, MySQL imprimía el error fatal `--initialize specified but the data directory has files in it. Aborting.`. El motor exige que `/var/lib/mysql` esté completamente vacío en la primera inicialización, condición que se incumple cuando el volumen persistente contiene archivos previos o el directorio `lost+found` creado en la raíz por sistemas de archivos `ext4`.
* **Cómo se resolvió:** en [`k8s/08-deploy-db.yaml`](./k8s/08-deploy-db.yaml) y en la plantilla equivalente del Chart de Helm se agregó `subPath: mysql` en el `volumeMounts` de `/var/lib/mysql`, aislando los datos del motor en un subdirectorio inicialmente vacío dentro del volumen persistente.

---

## 4. Qué quedaría pendiente para un entorno de producción real

* **Por qué una base de datos como `Deployment` con un volumen no es adecuada en producción:** un `Deployment` está diseñado para cargas *stateless* con Pods intercambiables y estrategia de despliegue *RollingUpdate*. Al recrear o actualizar el Pod de MySQL, Kubernetes puede intentar iniciar el Pod nuevo antes de que el anterior libere el volumen `ReadWriteOnce`, generando bloqueos *Multi-Attach*, ventanas de indisponibilidad o riesgo de corrupción. Además, un único Pod sin replicación constituye un punto único de fallo. En un entorno serio se utiliza un servicio administrado (AWS RDS, Cloud SQL) o un operador de MySQL sobre `StatefulSet` con identidad de red estable, almacenamiento por réplica y replicación primario-réplica con conmutación automática (*failover*).
* **Gestión externa de secretos:** en este repositorio, `k8s/03-secret-db.yaml` y `reservas-app/values.yaml` contienen únicamente **credenciales de laboratorio**. Los objetos `Secret` de Kubernetes solo codifican en Base64 (decodificable trivialmente con `base64 -d`). En producción se debe activar el cifrado en reposo en `etcd`, aplicar RBAC estricto e inyectar credenciales desde un gestor externo como **HashiCorp Vault**, **External Secrets Operator** o **Sealed Secrets**.
* **Copias de respaldo (Backups) y recuperación:** incorporar un `CronJob` de Kubernetes que ejecute respaldos lógicos (`mysqldump --single-transaction`) o snapshots del volumen hacia almacenamiento de objetos externo (S3/GCS) con cifrado, política de retención y pruebas periódicas de restauración.
* **Observabilidad:** centralizar logs estructurados (Loki/ELK), métricas de aplicación, Nginx y MySQL (Prometheus + Grafana con alertas sobre latencia, errores 5xx, saturación de pool y espacio en disco) y trazas distribuidas (OpenTelemetry).
* **Integración y despliegue continuos (CI/CD):** automatizar en un pipeline (GitHub Actions / GitLab CI) la ejecución de tests, construcción de imágenes inmutables etiquetadas por versión/commit, escaneo de vulnerabilidades con **Trivy** (fallando ante CVEs críticos con corrección disponible), `helm lint` / `helm template` y despliegue declarativo vía GitOps (ArgoCD o Flux) con TLS/HTTPS gestionado por `cert-manager` en el Ingress.
