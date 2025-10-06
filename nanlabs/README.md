# Cloud Infrastructure Challenge — LocalStack Edition

Este proyecto implementa una infraestructura **serverless simulada** utilizando **Terraform**, **LocalStack** y **Docker**, debido a que es una prueba para probar de manera local sin infraestructura en la nube, (AWS)
---

##  Arquitectura General

La arquitectura simula un entorno AWS con los siguientes componentes:

- **API Gateway** → Endpoint HTTP (`GET /info`)
- **Lambda Function** → Invocada por el API Gateway, se conecta a un backend HTTP
- **Backend (fake-service)** → Aplicación Docker que responde en `/version`
- **CloudWatch Logs** → Registra la salida de la Lambda
- **VPC simulada** → Incluye subredes públicas/privadas y grupos de seguridad lógicos
- **PostgreSQL** → Contenedor Docker que actúa como sustituto del RDS (solo si se desea probar la conexión a DB)

>  En un entorno AWS real, el RDS o EC2 en subred privada sustituirían el backend Docker.

---

##  Tecnologías Utilizadas

| Componente | Descripción |
|-------------|--------------|
| **Terraform** | Infraestructura como código (IaC) para desplegar módulos. |
| **LocalStack** | Emulador de servicios AWS local (Lambda, API Gateway, CloudWatch, etc.). |
| **Docker Compose** | Orquestador local para levantar LocalStack, backend y base de datos. |
| **Python (Lambda)** | Código de la función Lambda empaquetado con dependencias. |

---

## 🐳 Estructura del Proyecto

```bash
nanlabs/
├── modules/
│   ├── api_gateway/
│   ├── lambda/
│   │   ├── build/lambda.zip
│   │   └── src/app.py
│   ├── logs/
│   ├── rds/
│   └── vpc/
├── docker-compose.yml        # Orquesta LocalStack + backend + PostgreSQL
├── main.tf                   # Infraestructura raíz
├── providers.tf              # Configuración del provider AWS + Docker
├── variables.tf
├── outputs.tf                # Outputs globales
├── envs/
│   └── localstack.tfvars     # Configuración de entorno local
└── README.md
```

---

##  Instrucciones de Despliegue (LocalStack)

### 1 Levantar la infraestructura base con Docker Compose
```bash
docker-compose up -d
```

Esto levanta:
- `localstack-main` (servicios AWS emulados)
- `ls-api-dev-backend` (backend fake-service)
- `pg-local` (PostgreSQL opcional)

### 2️ Inicializar Terraform
```bash
terraform init -upgrade
```

### 3️ Aplicar la infraestructura en modo LocalStack
```bash
terraform apply -var-file=envs/localstack.tfvars -auto-approve
```
> ⚙️ El archivo `localstack.tfvars` define:
> ```hcl
> use_localstack = true
> region = "us-east-1"
> ```

### 4️ Ver los recursos creados
```bash
terraform output
```

---

##  Validación del Entorno

###  Probar el backend
```bash
curl http://localhost:5001/version
```

### 🔸 Invocar la Lambda
```bash
aws --endpoint-url=http://localhost:4566 lambda invoke   --function-name "ls-api-dev-fn" out.json --log-type Tail
cat out.json
```

### Invocar la API Gateway
```bash
curl "$(terraform output -raw api_url)"
```

---

## Comandos útiles

| Acción | Comando |
|--------|----------|
| Listar Lambdas | `aws --endpoint-url=http://localhost:4566 lambda list-functions` |
| Listar APIs | `aws --endpoint-url=http://localhost:4566 apigateway get-rest-apis` |
| Listar Logs | `aws --endpoint-url=http://localhost:4566 logs describe-log-groups` |
| Destruir recursos | `terraform destroy -var-file=envs/localstack.tfvars -auto-approve` |

---

##  docker-compose.yml explicado

###  Descripción general

El archivo `docker-compose.yml` orquesta tres contenedores principales:
1. **LocalStack** → emula los servicios de AWS.
2. **Fake Backend** → simula un servicio interno accesible desde la Lambda.
3. **PostgreSQL** → emula una base de datos RDS para pruebas locales.

###  Contenido típico del docker-compose.yml

```yaml
version: "3.8"

services:
  localstack:
    container_name: localstack-main
    image: localstack/localstack
    ports:
      - "4566:4566"
      - "4510-4559:4510-4559"
    environment:
      - SERVICES=lambda,logs,apigateway,cloudwatch,iam,s3
      - DEBUG=1
      - LAMBDA_EXECUTOR=docker-reuse
      - LAMBDA_DOCKER_NETWORK=nanlabs
      - AWS_ACCESS_KEY_ID=test
      - AWS_SECRET_ACCESS_KEY=test
      - AWS_DEFAULT_REGION=us-east-1
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./.localstack:/var/lib/localstack"
    networks:
      - nanlabs

  backend:
    container_name: ls-api-dev-backend
    image: ghcr.io/nicholasjackson/fake-service:v0.26.2
    environment:
      - LISTEN_ADDR=0.0.0.0:9090
      - NAME=local-backend
    ports:
      - "5001:9090"
    networks:
      - nanlabs

  pg-local:
    container_name: pg-local
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
      POSTGRES_DB: testdb
    ports:
      - "5432:5432"
    networks:
      - nanlabs

networks:
  nanlabs:
    driver: bridge
```

###  Explicación por servicio

| Servicio | Rol | Puertos | Notas |
|-----------|------|----------|-------|
| **localstack-main** | Emula AWS (Lambda, API Gateway, CloudWatch, etc.) | 4566 | Core del entorno LocalStack |
| **ls-api-dev-backend** | Servicio HTTP local que responde `/version` y `/health` | 5001 → 9090 | Actúa como backend privado |
| **pg-local** | Base de datos PostgreSQL para pruebas | 5432 | Emula un RDS PostgreSQL real |
| **Red Docker (`nanlabs`)** | Red compartida | — | Permite comunicación entre LocalStack, Lambda y backend |

## Cómo iniciarlo y detenerlo

**Iniciar entorno completo**
```bash
docker-compose up -d
```

**Verificar estado**
```bash
docker ps
```

**Apagar y limpiar**
```bash
docker-compose down -v
```

---


- **Autor:** Juan Andres Ceballos Beltran 
- **Infraestructura:** Terraform + LocalStack
