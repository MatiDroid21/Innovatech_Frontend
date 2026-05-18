# Innovatech Frontend — React + Vite

Interfaz web de la plataforma Innovatech Chile, desarrollada en **React + Vite**.  
Desplegada en una instancia **EC2 pública** en AWS, contenerizada con Docker y con pipeline CI/CD automatizado mediante **GitHub Actions**.

---

## Arquitectura
EC2 Frontend (subred pública — accesible desde Internet)
└── Contenedor: front-despacho → Puerto 80 (Nginx)
↓ comunica con →
EC2 Backend (subred privada)

El frontend consume los endpoints del backend a través de la IP interna de la VPC. Solo el frontend es accesible desde Internet.

---

## Contenedorización

### Requisitos previos
- Docker >= 24.x
- Docker Compose >= 2.x

### Variables de entorno

Copia `.env.example` a `.env` y completa los valores:

```bash
cp .env.example .env
```

| Variable | Descripción |
|---|---|
| `VITE_API_URL` | URL base del backend (ej: `http://<IP-BACKEND>:8080`) |

> Esta variable se embebe en el build. Si cambia la IP del backend, se debe reconstruir la imagen.

### Levantar el contenedor

```bash
docker compose up -d
```

### Verificar que el contenedor está corriendo

```bash
docker compose ps
docker compose logs -f
```

### Detener el contenedor

```bash
docker compose down
```

---

## Pipeline CI/CD — GitHub Actions

El pipeline se activa automáticamente con cada `push` a la rama **`deploy`**.

### Flujo del pipeline
push → rama deploy
↓

Build de imagen Docker (multi-stage: Node → Nginx)
↓

Push de imagen a Docker Hub / ECR
↓

SSH a EC2 → docker compose pull → docker compose up -d

### GitHub Secrets requeridos

Configurar en **Settings → Secrets and variables → Actions**:

| Secret | Descripción |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub |
| `DOCKERHUB_TOKEN` | Token de acceso Docker Hub |
| `EC2_HOST` | IP pública de la instancia EC2 del frontend |
| `EC2_USER` | Usuario SSH de la EC2 (ej: `ec2-user`) |
| `EC2_SSH_KEY` | Clave privada SSH (contenido del `.pem`) |
| `VITE_API_URL` | URL del backend para el build |

### Activar el pipeline

```bash
git checkout deploy
git merge main   # o la rama con tus cambios
git push origin deploy
```

---

## Estructura del repositorio
Innovatech_Frontend/
├── .github/
│ └── workflows/ # Pipelines GitHub Actions
├── front_despacho/
│ ├── Dockerfile # Multi-stage: Node (build) → Nginx (serve)
│ ├── nginx.conf # Configuración Nginx (proxy al backend)
│ └── src/
├── docker-compose.yml # Configuración del contenedor
├── .env.example # Variables de entorno requeridas
└── README.md

---

## Acceso

Una vez desplegado, el frontend es accesible desde:
http://<IP-PUBLICA-EC2-FRONTEND>

---

##  Integrantes

- Keiton Chaves
- Sergio Soto
- Matías Chávez

**Asignatura:** Introducción a Herramientas DevOps — ISY1101  
**Institución:** Duoc UC
