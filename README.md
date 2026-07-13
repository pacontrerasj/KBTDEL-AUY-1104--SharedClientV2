# API de ejemplo — AUY1104 (Express + Docker)

API académica mínima en **Node.js** y **Express**, pensada para practicar contenedores y pruebas con `curl`. Responde siempre en **JSON**.

## Requisitos

- Node.js 20+ (ejecución local)
- Docker (ejecución en contenedor)

## Ejecución local

```bash
npm install
npm start
```

Por defecto escucha en el puerto **3000**: `http://localhost:3000`.

## Docker

Construir la imagen (desde esta carpeta):

```bash
docker build -t auy1104-api-ejemplo .
```

Ejecutar el contenedor:

```bash
docker run --rm -p 3000:3000 -ti auy1104-api-ejemplo
```

Si el puerto **3000** de tu equipo ya está ocupado, usa otro puerto en el host (el primero del mapeo) y deja **3000** como puerto del contenedor:

```bash
docker run --rm -p 8080:3000 -ti auy1104-api-ejemplo
```

En ese caso las URLs de los ejemplos serían `http://localhost:8080/...`.

## Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/health` | Estado del servicio |
| `GET` | `/api/saludo` | Saludo en JSON; query opcional `nombre` |
| `POST` | `/api/echo` | Devuelve en JSON el cuerpo enviado |

Cualquier otra ruta responde **404** con JSON: `{ "error": "Ruta no encontrada" }`.

## Ejemplos con `curl`

Sustituye `localhost:3000` por `localhost:8080` (u otro) si mapeaste el contenedor distinto, por ejemplo `-p 8080:3000`.

### `GET /health`

```bash
curl -s http://localhost:3000/health
```

### `GET /api/saludo`

Sin parámetros (usa el nombre por defecto `estudiante`):

```bash
curl -s http://localhost:3000/api/saludo
```

Con query `nombre`:

```bash
curl -s "http://localhost:3000/api/saludo?nombre=Duoc"
```

### `POST /api/echo`

Envía JSON en el cuerpo; la API responde con estado **201** y el objeto recibido en `recibido`.

```bash
curl -s -X POST http://localhost:3000/api/echo \
  -H "Content-Type: application/json" \
  -d '{"curso":"AUY1104","modulo":"Docker"}'
```

### Ruta inexistente (404)

```bash
curl -s http://localhost:3000/api/no-existe
```

## Estructura del proyecto

```
Docker de Ejemplo/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
├── README.md
└── src/
    └── index.js
```

## Variables de entorno

| Variable | Valor por defecto | Uso |
|----------|-------------------|-----|
| `PORT` | `3000` | Puerto donde escucha la app dentro del contenedor o en local |

---

## Estrategia de Despliegue: Canary (v2 — EFT)

### Manifiestos

| Archivo | Propósito |
|---------|-----------|
| `k8s/deployment-stable.yaml` | 3 réplicas, etiqueta `version: stable`, con probes |
| `k8s/deployment-canary.yaml` | 1 réplica, etiqueta `version: canary`, con probes |
| `k8s/service.yaml` | Service tipo NodePort (30090) que selecciona `app: demo-api` |

### ¿Cómo se distribuye el 25% del tráfico al Canary?

El Service `demo-api-service` usa únicamente `app: demo-api` como selector, **sin filtrar por `version`**. Esto significa que el Service ve 4 Pods en total:

- 3 Pods con `app: demo-api, version: stable`
- 1 Pod con `app: demo-api, version: canary`

Kubernetes aplica **Round-Robin nativo** sobre los 4 endpoints. Por lo tanto, de cada 4 conexiones entrantes, 3 llegan a stable y 1 al canary → **~25% del tráfico valida la nueva versión en producción**. Si el canary falla, el impacto se limita a ese 25%; si responde bien, se puede promover a stable escalando el canary y luego actualizando stable.

### Pipeline — Health Gate y Rollback Automático

```
Build & Push Docker → Deploy Canary en K3s → Health Gate (5 intentos curl /health)
                                                    │
                                           ┌────────┴────────┐
                                           ✓                 ✗
                                      Promover a      Rollback automático:
                                      stable (futuro)  kubectl scale canary --replicas=0
                                                       Todo el tráfico vuelve a stable.
```

### Justificación Técnica — Reducción de MTTR (Mean Time To Recover)

| Aspecto | Beneficio |
|---------|-----------|
| **MTTR** | Si el Health Gate falla, el rollback se ejecuta en segundos (vía SSH + `kubectl scale`), evitando intervención manual que tomaría minutos. |
| **Impacto Controlado** | El canary recibe solo ~25% del tráfico. Una mala versión afecta a 1 de cada 4 usuarios, no al 100%. |
| **Resiliencia** | El stable de 3 réplicas permanece intacto durante toda la validación. El `livenessProbe` y `readinessProbe` aseguran que Pods defectuosos no reciban tráfico. |
| **Prueba de Fuego (Docente)** | Si el docente inyecta un error (ej. endpoint quebrado, CrashLoopBackOff), el Health Gate detecta el código HTTP no-200, el pipeline falla y el canary se escala a 0 automáticamente — **sin afectar la versión estable**. |
| **Costo de Fracaso** | Cero downtime para stable. La remediación es inmediata y no requiere `kubectl rollout undo` porque el canary nunca llegó a reemplazar a stable. |
