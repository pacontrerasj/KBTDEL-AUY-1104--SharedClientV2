# demo-api — AUY1104 (Express + Docker + K3s)

API académica construida con **Node.js** y **Express** como microservicio de ejemplo para el módulo AUY1104. Expone endpoints JSON para salud, saludos, echo y suma. Se despliega en un clúster **K3s sobre AWS EC2** mediante GitHub Actions, usando una estrategia **Canary** con Health Gate y rollback automático.

---

## Stack Tecnológico

| Capa | Tecnología |
|------|-----------|
| Runtime | Node.js 20+ (Alpine) |
| Framework | Express 4.21 |
| Contenedor | Docker + Docker Hub (`blackrosse/demo-api`) |
| Orquestador | Kubernetes (K3s en EC2) |
| CI/CD | GitHub Actions (workflows reutilizables) |
| Ingress | HAProxy académico compartido (puerto 30088) |

---

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/health` | Estado del servicio |
| `GET` | `/api/saludo` | Saludo en JSON; query opcional `?nombre=` |
| `POST` | `/api/echo` | Devuelve el cuerpo enviado en JSON |
| `GET` | `/api/suma?a=&b=` | Suma por query params |
| `POST` | `/api/suma` | Suma por body JSON |

Cualquier otra ruta responde **404** con `{ "error": "Ruta no encontrada" }`.

---

## Ejecución Local

```bash
npm install
npm start
# Escucha en http://localhost:3000
```

### Docker

```bash
docker build -t demo-api .
docker run --rm -p 3000:3000 demo-api
```

---

## Variables de Entorno

| Variable | Valor por defecto | Uso |
|----------|-------------------|-----|
| `PORT` | `3000` | Puerto de escucha dentro del contenedor |
| `LOG_LEVEL` | — | Nivel de logging (inyectado vía pipeline) |

---

## Estructura del Proyecto

```
├── .github/workflows/
│   ├── client.yaml        # Pipeline de deploy (tag v*.*.* o manual)
│   └── tests.yaml          # Tests en push a cualquier rama
├── k8s/
│   ├── deployment-stable.yaml   # 3 réplicas, versión estable
│   ├── deployment-canary.yaml   # 1 réplica, versión canary
│   └── service.yaml             # NodePort 30090, selector app:demo-api
├── src/
│   ├── index.js
│   └── lib/ejemplo.js
├── tests/
├── Dockerfile
├── package.json
└── README.md
```

---

## CI/CD: Pipeline de Deploy Canary

### Disparadores

- **Automático:** Push de tags `v*.*.*`
- **Manual:** `workflow_dispatch` desde la UI de GitHub Actions

### Flujo del Pipeline

```
[Tag v*.*.*] ──► client.yaml
                    │
                    ▼
    deploy-api.yaml (SharedWorkflowsV2)
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   Build & Push  Deploy Canary  Health Gate
   (Docker Hub)  (SSH a K3s)    (5 curls /health)
                                    │
                              ┌─────┴─────┐
                              ✓           ✗
                           Éxito      Rollback:
                                      scale canary ──replicas=0
```

### Secretos y Variables Requeridas en GitHub

| Nombre | Tipo | Descripción |
|--------|------|-------------|
| `DOCKER_USERNAME` | Secret | Usuario de Docker Hub (`blackrosse`) |
| `DOCKER_PASSWORD` | Secret | Token de acceso de Docker Hub |
| `EA2_SSH_PRIVATE_KEY` | Secret | Clave privada SSH para EC2 |
| `K3S_SERVER_PUBLIC_IP` | Variable | IP pública del servidor K3s |
| `K8S_NAMESPACE` | Variable | Namespace de K8s (default: `default`) |
| `API_ENV_VARS` | Variable | Env vars extra en formato `CLAVE=VALOR,CLAVE2=VALOR2` |

---

## Estrategia Canary

### Manifiestos

| Archivo | Propósito |
|---------|-----------|
| `k8s/deployment-stable.yaml` | 3 réplicas, `version: stable`, probes en `/health` |
| `k8s/deployment-canary.yaml` | 1 réplica, `version: canary`, probes en `/health` |
| `k8s/service.yaml` | NodePort 30090, selector `app: demo-api` (sin `version`) |

### Distribución de Tráfico (25%)

El Service selecciona únicamente por `app: demo-api`, sin filtrar por `version`. Con 4 Pods totales (3 stable + 1 canary), Kubernetes aplica Round-Robin nativo: de cada 4 conexiones, 3 van a stable y 1 al canary (~25%). Esto limita el impacto de una versión defectuosa al 25% de los usuarios.

### Health Gate

El paso 5 del pipeline ejecuta hasta 5 intentos de `curl -f http://<IP>:30090/health` con 5 segundos de espera entre cada uno. Si todos fallan, el pipeline se marca como fallido.

### Rollback Automático

Si el pipeline falla (ya sea en el deploy del canary o en el health check), el paso 6 ejecuta:

```
sudo k3s kubectl scale deployment/demo-api-canary --replicas=0
```

Esto elimina los Pods canary al instante. El Service solo ve los 3 Pods estables, restaurando el 100% del tráfico a la versión anterior. La remediación toma segundos, sin intervención manual.

### Probes de Resiliencia

Ambos deployments incluyen:

- **readinessProbe:** `HTTP GET /health` cada 10s, con 5s de retardo inicial. Si falla, el Pod no recibe tráfico del Service.
- **livenessProbe:** `HTTP GET /health` cada 20s, con 15s de retardo inicial. Si falla, K8s reinicia el Pod automáticamente.

Esto mitiga escenarios de `CrashLoopBackOff` y errores de configuración: un Pod que no responde no recibe tráfico y es reiniciado.

---

## Justificación Técnica (MTTR)

| Aspecto | Beneficio |
|---------|-----------|
| **Tiempo de recuperación** | Rollback en segundos vía `kubectl scale`, sin intervención manual. |
| **Impacto controlado** | Solo el 25% del tráfico toca el canary. Una mala versión no afecta a todos los usuarios. |
| **Aislamiento** | El deployment estable nunca se modifica durante la validación del canary. |
| **Prueba de Fuego** | Si el docente inyecta un error, el Health Gate lo detecta, el pipeline falla y el canary se escala a 0 automáticamente. La versión estable sigue sirviendo tráfico sin interrupción. |
| **Costo de fallo** | Cero downtime para el stable. La remediación es inmediata. |

---

## Pruebas

El workflow `tests.yaml` corre en push a cualquier rama excepto `main`:

```bash
npm test
```

Actualmente ejecuta un placeholder. Para pruebas reales se usa Jest + Supertest (declarados en `devDependencies`).

---

## Referencias

- Repositorio de Workflows Compartidos: `pacontrerasj/KBTDEL-AUY-1104--SharedWorkflowsV2`
- Docker Hub: `blackrosse/demo-api`
- Clúster K3s: AWS EC2 provisionado vía `asanchezo-duoc/AUY1104-SharedWorkflows`
