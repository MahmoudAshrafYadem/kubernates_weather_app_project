# 🌤️ Kubernetes Weather App

A fully containerized, microservices-based weather application deployed on Kubernetes. This project demonstrates best practices for orchestrating a multi-service application using Kubernetes primitives including Deployments, StatefulSets, Services, Ingress, Secrets, Jobs, and Persistent Volume Claims.

---

## 📋 Table of Contents

- [Architecture Overview](#-architecture-overview)
- [Microservices](#-microservices)
- [Prerequisites](#-prerequisites)
- [Project Structure](#-project-structure)
- [Quick Start](#-quick-start)
- [Configuration](#-configuration)
- [Detailed Component Breakdown](#-detailed-component-breakdown)
  - [UI Service (Frontend)](#ui-service-frontend)
  - [Auth Service (Authentication)](#auth-service-authentication)
  - [Weather Service (API Backend)](#weather-service-api-backend)
  - [MySQL Database](#mysql-database)
- [Ingress & TLS](#-ingress--tls)
- [Secrets Management](#-secrets-management)
- [Scaling](#-scaling)
- [Cleanup](#-cleanup)
- [License](#-license)

---

## 🏗️ Architecture Overview

The application follows a **microservices architecture** composed of four distinct services, each responsible for a specific domain. All services are containerized and orchestrated within a Kubernetes cluster, communicating internally via Kubernetes Service discovery.

```
                        ┌──────────────┐
                        │    Client    │
                        └──────┬───────┘
                               │ HTTPS
                        ┌──────▼───────┐
                        │   Ingress    │
                        │  (nginx TLS) │
                        └──────┬───────┘
                               │
                   ┌───────────▼───────────┐
                   │   UI Service (:3000)  │
                   │   (2 Replicas)        │
                   └──┬───────────────┬────┘
                      │               │
             ┌────────▼───┐    ┌──────▼─────────┐
             │ Auth Svc   │    │ Weather Svc    │
             │ (:8080)    │    │ (:5000)        │
             │ (1 Replica)│    │ (2 Replicas)   │
             └─────┬──────┘    └──────┬─────────┘
                   │                  │
                   │           ┌──────▼─────┐
                   │           │ OpenWeather │
                   │           │ API (ext.)  │
                   │           └────────────┘
                   │
              ┌────▼──────┐
              │   MySQL   │
              │ (:3306)   │
              │ Stateful  │
              │  Set      │
              └───────────┘
```

---

## ⚙️ Microservices

| Service | Image | Port | Replicas | Description |
|---|---|---|---|---|
| **UI** | `afakharany/weatherapp-ui:lab` | 3000 | 2 | Frontend web interface for weather data display |
| **Auth** | `afakharany/weatherapp-auth:lab` | 8080 | 1 | User authentication and session management |
| **Weather** | `afakharany/weatherapp-weather:lab` | 5000 | 2 | Weather data retrieval from external API |
| **MySQL** | `mysql:5.7` | 3306 | 1 | Persistent relational database for auth data |

---

## 📦 Prerequisites

Before deploying this project, ensure you have the following tools and resources available:

- **Kubernetes Cluster** (v1.21+) — Minikube, Kind, GKE, EKS, AKS, or any conformant cluster
- **kubectl** — Command-line tool for interacting with your cluster
- **NGINX Ingress Controller** — Must be installed in your cluster for the Ingress resource to function
- **StorageClass** named `standard` — Required for the MySQL Persistent Volume Claim (adjust `storageClassName` in `mysql/statefulset.yaml` if your cluster uses a different name)
- **Helm** (optional) — Useful for managing secrets and advanced deployments
- **OpenWeather API Key** — Required for the Weather service to fetch real-time weather data (get one at [openweathermap.org](https://openweathermap.org/api))

---

## 📁 Project Structure

```
kubernates_weather_app_project/
├── ui/
│   ├── deployment.yaml        # UI frontend deployment (2 replicas)
│   ├── service.yaml           # ClusterIP service exposing port 3000
│   ├── ingress.yaml           # NGINX Ingress with TLS termination
│   ├── tls.crt                # TLS certificate for weatherapp.local
│   └── tls.key                # TLS private key for weatherapp.local
├── auth/
│   ├── deployment.yaml        # Authentication service deployment (1 replica)
│   └── service.yaml           # ClusterIP service exposing port 8080
├── weather/
│   ├── deployment.yaml        # Weather API service deployment (2 replicas)
│   ├── service.yaml           # ClusterIP service exposing port 5000
│   └── secret.yaml            # OpenWeather API key (base64 encoded)
└── mysql/
    ├── statefulset.yaml       # MySQL StatefulSet with persistent storage
    ├── headless-service.yaml  # Headless service for stable network identity
    └── init-job.yaml          # Initialization job for database & user setup
```

---

## 🚀 Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/MahmoudAshrafYadem/kubernates_weather_app_project.git
cd kubernates_weather_app_project
```

### 2. Create the Required Secret

Before deploying any services, you must create the `mysql-secret` that stores sensitive credentials. This secret is referenced by the Auth service, MySQL StatefulSet, and the MySQL init Job.

```bash
kubectl create secret generic mysql-secret \
  --from-literal=root-password='your_root_password' \
  --from-literal=auth-password='your_auth_user_password' \
  --from-literal=secret-key='your_jwt_secret_key'
```

> ⚠️ **Important:** Replace the placeholder values above with strong, unique passwords. In production, consider using a secrets management tool such as [HashiCorp Vault](https://www.vaultproject.io/), [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets), or [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/).

### 3. Update the Weather API Key (Optional)

The Weather service's API key is stored in `weather/secret.yaml`. To use your own OpenWeather API key, encode it in base64 and update the value:

```bash
# Generate a base64-encoded API key
echo -n 'your_openweather_api_key' | base64

# Then update weather/secret.yaml with the output
```

### 4. Deploy All Services

Deploy the services in the following order to ensure proper dependency management (database first, then backend services, then frontend):

```bash
# Step 1: Deploy MySQL database
kubectl apply -f mysql/

# Step 2: Wait for the MySQL init job to complete and the pod to be ready
kubectl wait --for=condition=complete job/mysql-init-job --timeout=120s
kubectl wait --for=condition=ready pod -l app=mysql --timeout=120s

# Step 3: Deploy backend services (Auth + Weather)
kubectl apply -f auth/
kubectl apply -f weather/

# Step 4: Deploy the UI frontend and Ingress
kubectl apply -f ui/
```

Alternatively, you can deploy everything at once if you're confident the MySQL StatefulSet will be ready before the Auth service starts:

```bash
kubectl apply -f mysql/
kubectl apply -f auth/
kubectl apply -f weather/
kubectl apply -f ui/
```

### 5. Verify the Deployment

```bash
# Check all pods are running
kubectl get pods

# Check services
kubectl get svc

# Check ingress
kubectl get ingress
```

### 6. Access the Application

Add the following entry to your local `/etc/hosts` file (or the equivalent on your operating system) to resolve the Ingress host:

```
127.0.0.1  weatherapp.local
```

Then open your browser and navigate to:

```
https://weatherapp.local
```

> **Note:** Since the TLS certificate is self-signed, your browser will display a security warning. This is expected — click "Advanced" and proceed to the site. For production use, replace the self-signed certificates with certificates issued by a trusted Certificate Authority (e.g., using [Let's Encrypt](https://letsencrypt.org/) with [cert-manager](https://cert-manager.io/)).

---

## 🔧 Configuration

### Environment Variables

#### UI Service

| Variable | Default Value | Description |
|---|---|---|
| `AUTH_HOST` | `weatherapp-auth` | Hostname of the Auth service |
| `AUTH_PORT` | `8080` | Port of the Auth service |
| `WEATHER_HOST` | `weatherapp-weather` | Hostname of the Weather service |
| `WEATHER_PORT` | `5000` | Port of the Weather service |

#### Auth Service

| Variable | Default Value | Description |
|---|---|---|
| `DB_HOST` | `mysql` | MySQL host address |
| `DB_USER` | `authuser` | MySQL username for the auth service |
| `DB_PASSWORD` | *(from secret)* | Password for the auth MySQL user |
| `DB_NAME` | `weatherapp` | MySQL database name |
| `DB_PORT` | `3306` | MySQL port |
| `SECRET_KEY` | *(from secret)* | JWT secret key for token signing |

#### Weather Service

| Variable | Default Value | Description |
|---|---|---|
| `APIKEY` | *(from secret)* | OpenWeather API key |

#### MySQL

| Variable | Source | Description |
|---|---|---|
| `MYSQL_ROOT_PASSWORD` | *(from secret)* | Root password for MySQL |

---

## 🔍 Detailed Component Breakdown

### UI Service (Frontend)

The UI service is the entry point for end users, served through the NGINX Ingress controller. It runs 2 replicas for high availability and includes both **liveness** and **readiness probes** hitting the `/health` endpoint on port 3000, ensuring traffic is only routed to healthy instances.

- **Deployment:** `ui/deployment.yaml`
- **Service:** `ui/service.yaml` (ClusterIP on port 3000)
- **Ingress:** `ui/ingress.yaml` (NGINX class, TLS on `weatherapp.local`)

### Auth Service (Authentication)

The Auth service handles user authentication, session management, and JWT token generation. It connects to the MySQL database using credentials sourced from Kubernetes Secrets, ensuring sensitive data is never hardcoded in the deployment manifest. The service runs a single replica and is exposed internally via a ClusterIP service on port 8080.

- **Deployment:** `auth/deployment.yaml`
- **Service:** `auth/service.yaml` (ClusterIP on port 8080)

### Weather Service (API Backend)

The Weather service fetches real-time weather data from the [OpenWeather API](https://openweathermap.org/api). It runs 2 replicas and uses an API key stored as a Kubernetes Secret for secure credential management. Both liveness and readiness probes are configured on the root path (`/`) to ensure the service is operational before receiving traffic.

- **Deployment:** `weather/deployment.yaml`
- **Service:** `weather/service.yaml` (ClusterIP on port 5000)
- **Secret:** `weather/secret.yaml` (base64-encoded API key)

### MySQL Database

MySQL is deployed as a **StatefulSet** rather than a standard Deployment. This Kubernetes primitive provides:

- **Stable network identity** via a Headless Service (each pod gets a predictable DNS name like `mysql-0.mysql`)
- **Ordered, graceful deployment and scaling** of pods
- **Persistent storage** via a `volumeClaimTemplate` that provisions a 10Gi Persistent Volume per pod, ensuring data survives pod restarts and rescheduling

A separate **Initialization Job** (`mysql/init-job.yaml`) runs once to create the `weatherapp` database, the `authuser` MySQL user, and grant the necessary permissions. This job ensures the database schema is ready before the Auth service attempts to connect.

- **StatefulSet:** `mysql/statefulset.yaml`
- **Headless Service:** `mysql/headless-service.yaml` (port 3306, `clusterIP: None`)
- **Init Job:** `mysql/init-job.yaml`

---

## 🔐 Ingress & TLS

The application is exposed externally through an NGINX Ingress resource configured with TLS termination. The Ingress routes all HTTP/HTTPS traffic for the host `weatherapp.local` to the UI service on port 3000.

- **TLS certificates** are provided as files (`ui/tls.crt` and `ui/tls.key`) and stored in a Kubernetes TLS Secret named `weatherapp-ui-tls`.
- The `ingressClassName: nginx` annotation ensures the NGINX Ingress Controller handles this resource.

For production deployments, it is strongly recommended to:
1. Use a domain name backed by a real DNS provider instead of `weatherapp.local`
2. Replace self-signed certificates with Let's Encrypt certificates managed by [cert-manager](https://cert-manager.io/docs/)
3. Enable additional security annotations on the Ingress (e.g., rate limiting, CORS headers, HSTS)

---

## 🔑 Secrets Management

This project uses Kubernetes native `Secret` objects to manage sensitive information:

| Secret Name | Keys | Used By |
|---|---|---|
| `mysql-secret` | `root-password`, `auth-password`, `secret-key` | Auth service, MySQL StatefulSet, MySQL init Job |
| `weather` | `apikey` | Weather service |

**Best practices for secrets in production:**
- Never commit raw secrets to version control (this repository includes an API key in `weather/secret.yaml` for convenience — **rotate it immediately** before any production use)
- Use tools like [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets), [SOPS](https://github.com/mozilla/sops), or external secret managers to encrypt secrets at rest
- Enable RBAC policies to restrict which pods and users can access specific secrets
- Consider using [HashiCorp Vault](https://www.vaultproject.io/) or cloud-native solutions (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault) for enterprise-grade secret management

---

## 📈 Scaling

The application is configured with the following default replica counts, but you can scale any service up or down based on demand:

```bash
# Scale the UI service to 5 replicas
kubectl scale deployment release-name-weatherapp-ui --replicas=5

# Scale the Weather service to 3 replicas
kubectl scale deployment weatherapp-weather --replicas=3

# Scale the Auth service to 2 replicas
kubectl scale deployment weatherapp-auth --replicas=2
```

For automatic scaling based on CPU/memory usage, you can add a `HorizontalPodAutoscaler` (HPA):

```bash
kubectl autoscale deployment weatherapp-weather --cpu-percent=70 --min=2 --max=10
```

> **Note:** The MySQL StatefulSet uses a `ReadWriteOnce` access mode PVC. Scaling MySQL to multiple replicas requires additional configuration for primary-replica replication and is not supported out of the box with the current manifests.

---

## 🧹 Cleanup

To remove all deployed resources from your cluster:

```bash
kubectl delete -f ui/
kubectl delete -f weather/
kubectl delete -f auth/
kubectl delete -f mysql/

# Also delete the manually created secret
kubectl delete secret mysql-secret
```

If you want to remove everything in one command:

```bash
kubectl delete secret mysql-secret 2>/dev/null
kubectl delete -f ui/ -f weather/ -f auth/ -f mysql/
```

> ⚠️ **Warning:** Deleting the MySQL StatefulSet will also delete the associated Persistent Volume Claim and its data. Make sure to back up any important data before running the cleanup commands.

---

## 📝 License

This project is licensed under the [MIT License](LICENSE). Feel free to use, modify, and distribute it as needed.
