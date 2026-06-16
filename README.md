# Innovatech Chile — Frontend

Aplicación web para la gestión de despachos de Innovatech Chile.
Desarrollada con **React + Vite + TailwindCSS**, contenerizada con Docker
y desplegada en un clúster **Amazon EKS** con balanceo de carga automático.

---

## Tecnologías utilizadas

- React 18 + Vite
- TailwindCSS
- Nginx (servidor de producción)
- Docker + Docker Compose
- GitHub Actions (CI/CD)
- Amazon EKS (Kubernetes)
- Amazon ECR (registro de imágenes)

---

## Estructura del repositorio
Innovatech_Frontend/
├── front_despacho/ # Aplicación React
│ ├── src/ # Código fuente (componentes, vistas)
│ ├── Dockerfile # Imagen Docker multi-stage (build + nginx)
│ ├── nginx.conf # Configuración de Nginx
│ └── vite.config.js # Configuración de Vite
├── docker-compose.yml # Levantamiento local
├── .env.example # Variables de entorno requeridas
└── .github/workflows/ # Pipeline CI/CD

text

---

## Variables de entorno

Crear un archivo `.env` en la raíz basado en `.env.example`:

```env
VITE_API_URL=http://<IP-O-DNS-BACKEND>:8081
```

> En producción esta variable se inyecta como `build-arg` en el pipeline de GitHub Actions.

---

## Levantar localmente

```bash
git clone https://github.com/MatiDroid21/Innovatech_Frontend.git
cd Innovatech_Frontend
cp .env.example .env
# Editar .env con la URL del backend
docker compose up -d --build
```

La aplicación quedará disponible en `http://localhost:80`.

---

## Pipeline CI/CD (GitHub Actions)

El pipeline se activa automáticamente con cada push a la rama `deploy`.

**Job 1 — Build & Push:**
1. Construye la imagen Docker del frontend
2. Inyecta `VITE_API_URL` como build-arg
3. Sube la imagen a Docker Hub como `usuario/innovatech-frontend:latest`

**Job 2 — Deploy (depende del Job 1):**
1. Se conecta por SSH a la EC2
2. Hace `git pull` y actualiza el `.env`
3. Ejecuta `docker compose up -d` con la nueva imagen

**Secrets requeridos en GitHub:**

| Secret | Descripción |
|---|---|
| `DOCKER_USERNAME` | Usuario Docker Hub |
| `DOCKER_PASSWORD` | Token Docker Hub |
| `EC2_HOST` | IP pública EC2 |
| `EC2_USER` | Usuario SSH (`ec2-user`) |
| `EC2_SSH_KEY` | Contenido del archivo `.pem` |
| `VITE_API_URL` | URL del backend |

---

## Despliegue en EKS

El frontend corre en Amazon EKS como un `Deployment` con 2 réplicas.
El acceso público se realiza a través de un **Network Load Balancer (NLB)**
creado automáticamente por Kubernetes al declarar `type: LoadBalancer`
en el `frontend-service`.

```bash
# Ver el servicio y URL del balanceador
kubectl get services

# Ver pods en ejecución
kubectl get pods
```

URL pública del balanceador:
acafd3cfd8ad84468b8295effd5e5caf-2ca44e4ca21e036b.elb.us-east-1.amazonaws.com
