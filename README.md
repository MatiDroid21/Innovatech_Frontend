# Innovatech Chile — Frontend

Aplicación web para la gestión de despachos de Innovatech Chile.
Desarrollada con **React + Vite + TailwindCSS**, contenerizada con Docker
y desplegada en un clúster **Amazon EKS** con balanceo de carga automático..

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
1. Configura las credenciales AWS en el runner.
2. Se autentica en Amazon ECR.
3. Construye la imagen Docker del frontend inyectando `VITE_API_URL` como build-arg.
4. Publica la imagen en Amazon ECR.

**Job 2 — Deploy en EKS (depende del Job 1):**
1. Configura las credenciales AWS en el runner.
2. Actualiza el contexto de `kubectl` apuntando al clúster EKS.
3. Ejecuta `rollout restart` en el deployment del frontend.
4. Verifica el estado del despliegue con `rollout status`.

**Secrets requeridos en GitHub:**

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial de acceso AWS |
| `AWS_SECRET_ACCESS_KEY` | Clave secreta AWS |
| `AWS_SESSION_TOKEN` | Token de sesión temporal AWS Academy |
| `AWS_REGION` | Región del clúster (us-east-1) |
| `ECR_REGISTRY` | URL del registro de imágenes ECR |
| `EKS_CLUSTER_NAME` | Nombre del clúster EKS destino |
| `VITE_API_URL` | URL del backend para el build de React |

---

## Despliegue en EKS

El frontend corre en Amazon EKS como un `Deployment` con 1 réplica activa,
escalando hasta 4 mediante **Horizontal Pod Autoscaler (HPA)** según demanda de CPU
(umbral 50%).

El acceso público se realiza a través de un **Network Load Balancer (NLB)**
creado automáticamente por Kubernetes al declarar `type: LoadBalancer`
en el `frontend-service`.

```bash
# Ver el servicio y URL del balanceador
kubectl get services

# Ver pods en ejecución
kubectl get pods

# Ver HPA
kubectl get hpa
```

> Considerar esto: La URL del NLB cambia cada vez que se recrea el servicio en AWS Academy.
> Actualizar `VITE_API_URL` y los secrets de GitHub Actions cuando la sesión renueve credenciales.


---


## Arquitectura general
Internet
│
▼
[Network Load Balancer - NLB]
│
▼
[frontend-service - LoadBalancer]
│
▼
[Pod Frontend - React/Nginx]
│
├──► ventas-service (ClusterIP) ──► [Pod Ventas - Spring Boot]
│ │
└──► despachos-service (ClusterIP) ──► [Pod Despachos - Spring Boot]
│
[mysql-service]
│
[Pod MySQL 8]

text

El frontend es el único componente expuesto públicamente vía NLB.
Los microservicios backend y MySQL son internos al clúster (ClusterIP),
accesibles solo mediante el DNS interno de Kubernetes.

---

## Problemas encontrados y soluciones

### Error 504 Gateway Timeout en Nginx
**Causa:** El archivo `nginx.conf` tenía IPs hardcodeadas (`10.0.2.10`)
correspondientes al entorno local de desarrollo, inválidas dentro del clúster EKS.

**Solución:** Se reemplazaron las IPs estáticas por los nombres de servicio
internos de Kubernetes (`ventas-service` y `despachos-service`), que son
resueltos automáticamente por el DNS interno del clúster.

```nginx
# Antes (incorrecto en EKS)
proxy_pass http://10.0.2.10:8080;

# Después (correcto)
proxy_pass http://ventas-service;
proxy_pass http://despachos-service;
```

---

## Justificación del umbral HPA (50% CPU)

Se eligió el **50% de uso de CPU** como umbral de escalado porque representa
un punto de equilibrio entre rendimiento y costo:

- Permite absorber picos de carga antes de que afecten la experiencia del usuario.
- Evita escalar prematuramente ante uso normal.
- Con instancias `t3.medium` (2 vCPU), el 50% equivale a **1 vCPU disponible
  como margen** antes de que Kubernetes agregue una nueva réplica.

Configuración aplicada:
- Mínimo: 1 réplica
- Máximo: 4 réplicas
- Target CPU: 50%
