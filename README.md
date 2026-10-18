# Todo Summary Assistant — DevOps Deployment

A full-stack Todo Summary Assistant application deployed on AWS using Docker, Amazon ECR, Amazon EC2, Amazon RDS for MySQL, GitHub Actions, Prometheus, and Grafana.

This repository demonstrates containerization, automated CI/CD, cloud deployment, application health checks, infrastructure security, and monitoring.

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Local Setup and Execution](#local-setup-and-execution)
- [Environment Configuration](#environment-configuration)
- [Docker Design and Containerization](#docker-design-and-containerization)
- [CI/CD Pipeline](#cicd-pipeline)
- [AWS Deployment and Security](#aws-deployment-and-security)
- [Monitoring and Alerting](#monitoring-and-alerting)
- [Assumptions and Design Decisions](#assumptions-and-design-decisions)
- [Troubleshooting](#troubleshooting)

## Project Overview

The project contains a React frontend, a Spring Boot REST API, and a MySQL database. The application is containerized and deployed to an EC2 instance, while the database is hosted separately on Amazon RDS.

GitHub Actions automates application validation, Docker image creation, publishing to Amazon ECR, and deployment to EC2 using AWS Systems Manager (SSM).

Prometheus collects application and host metrics, while Grafana provides dashboards for application performance and infrastructure health.

### Key DevOps objectives

- Containerize the frontend and backend using Docker.
- Build optimized images using multi-stage Dockerfiles.
- Automate build, test, image publishing, and deployment through GitHub Actions.
- Use Amazon ECR as the container image registry.
- Host the application on Amazon EC2 and the database on Amazon RDS for MySQL.
- Externalize application configuration and keep credentials out of source control.
- Monitor application and infrastructure health using Prometheus and Grafana.
- Document deployment failures, rollback procedures, and operational practices.

## Architecture

```text
                   Developer
                       |
                       v
                GitHub Repository
                       |
                       v
               GitHub Actions CI/CD
                       |
                Build and Validate
                       |
                Build Docker Images
                       |
                       v
                 Amazon ECR
              /              \
             v                v
      Backend Image      Frontend Image
             \                /
              \              /
               v            v
                  Amazon EC2
             +--------------------+
             | Docker Containers  |
             |                    |
             | React + Nginx      |
             | Spring Boot API    |
             +--------------------+
                       |
                       v
                Amazon RDS MySQL
                 Private Network

    Prometheus <--- Application Metrics
        |
        +--------- Node Exporter
        |
        v
      Grafana Dashboard and Alerts
```

The diagram is a logical overview. Refer to `aws/architecture-diagram.png` for the deployment architecture and network boundaries.

## Technology Stack

| Component | Technology |
|---|---|
| Frontend | React.js, Node.js build environment, Nginx |
| Backend | Java 21, Spring Boot, Maven |
| Database | Amazon RDS for MySQL |
| Containerization | Docker |
| Container Registry | Amazon ECR |
| Cloud Hosting | Amazon EC2 |
| CI/CD | GitHub Actions |
| Deployment | AWS Systems Manager (SSM) |
| Monitoring | Prometheus, Node Exporter, Grafana |
| Metrics Endpoint | Spring Boot Actuator with Prometheus metrics |
| Source Control | Git and GitHub |

## Prerequisites

Install the following tools before running the application locally:

- Git
- Java 21
- Maven, or the project's Maven Wrapper if provided
- Node.js and npm
- Docker Engine and Docker Compose, if running containers locally

You will also need access to a MySQL database and any external API credentials required by the application's configuration.

## Local Setup and Execution

### 1. Clone the repository

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd TodoSummaryAssistant
```

### 2. Start the backend

Navigate to the backend directory:

```bash
cd Backend/todo-summary-assistant
```

Configure the database connection and any required external service credentials using environment variables or a local, untracked environment file.

Build the backend:

```bash
mvn clean package
```

If the project includes a Maven Wrapper, you can use `./mvnw clean package` instead.

Run the application:

```bash
mvn spring-boot:run
```

The backend is configured to listen on port `8080` in the deployment environment. Verify the actual local port and health endpoint against the application configuration.

If Spring Boot Actuator is enabled, check:

```bash
curl -i http://localhost:8080/actuator/health
```

A healthy application should return HTTP 200 with a status of `UP`.

### 3. Start the frontend

Open a second terminal and navigate to the frontend directory:

```bash
cd Frontend/todo
```

Install dependencies:

```bash
npm install
```

Inspect `package.json` for the available scripts, then start the development server using the appropriate script. For example, if the project defines a `dev` script:

```bash
npm run dev
```

If it defines a `start` script instead, use:

```bash
npm start
```

Configure the frontend's API base URL to point to the locally running backend. Use the environment-variable name expected by the frontend build configuration.

### 4. Configure MySQL

The backend requires a reachable MySQL database. You can use a local MySQL instance for development or a suitably secured development database.

Configure the database URL, database name, username, and password through the application's supported environment variables or configuration mechanism.

Do not use production database credentials for local development.

### 5. Run with Docker

Build the backend image from the repository root:

```bash
docker build \
  -t todo-summary-backend:local \
  -f Backend/todo-summary-assistant/Dockerfile \
  Backend/todo-summary-assistant
```

Build the frontend image:

```bash
docker build \
  -t todo-summary-frontend:local \
  -f Frontend/todo/Dockerfile \
  Frontend/todo
```

Run the containers with the environment configuration and network settings required by the application. Ensure that the backend can resolve and reach the configured MySQL host.

If a root-level `docker-compose.yml` is included and configured for local development, it can be used instead:

```bash
docker compose up --build -d
docker compose ps
docker compose logs --tail=100
```

Use the ports and service names defined in the Compose file. Do not assume that the local development setup and the production EC2 setup use identical ports or environment files.

To stop a local Compose environment:

```bash
docker compose down
```

This command should only be run against the intended local Compose project, not against the production deployment.

## Environment Configuration

Application configuration must be supplied externally instead of being committed to the repository.

Create a local environment file based on `.env.example`, if provided, and fill in the required values locally.

Typical configuration categories include:

| Configuration | Purpose |
|---|---|
| Database URL/host | Identifies the MySQL database |
| Database name | Selects the application database |
| Database username | Authenticates the backend to MySQL |
| Database password | Authenticates the backend to MySQL |
| External API key | Authenticates any required external AI/API integration |
| Webhook URL | Configures any required notification integration |

Use the exact variable names supported by the application and deployment script. The names above describe configuration categories, not a guarantee of the application's literal variable names.

**Security requirements:**

- Never commit `.env` files, database passwords, API keys, webhook URLs, private keys, or access tokens.
- Keep production values in GitHub Actions secrets or an appropriate AWS secret-management service.
- Keep production environment files on EC2 restricted to the deployment user and authorized administrators.
- Commit only placeholder values in `.env.example`.

## Docker Design and Containerization

The project uses separate Docker images for the frontend and backend.

### Backend Dockerfile

The backend Dockerfile should use a multi-stage build:

1. **Build stage:** Uses a Java/Maven build environment to resolve dependencies, compile the application, and package the executable JAR.
2. **Runtime stage:** Uses a smaller Java runtime image and copies only the packaged application into the final image.

The runtime process should run as a non-root user. Database configuration and other application settings are provided at runtime through environment variables rather than embedded in the image.

### Frontend Dockerfile

The frontend Dockerfile should use a multi-stage build:

1. **Build stage:** Uses Node.js and npm to install dependencies and generate optimized production assets.
2. **Runtime stage:** Uses Nginx to serve the generated static assets without including the Node.js build environment in the final image.

The production frontend listens on the port configured by Nginx and the deployment. API routing must be configured to reach the backend correctly.

### `.dockerignore` files

Separate `.dockerignore` files should be maintained in the backend and frontend build contexts.

They should exclude unnecessary content such as:

- `node_modules/`
- Build output and temporary files that do not belong in the image
- `.git/`
- Local environment files containing secrets
- Logs, caches, and IDE-specific files

The exact exclusions should preserve all files needed for a successful build.

### Benefits of this design

- **Smaller images:** Build tools and intermediate files are excluded from runtime images.
- **Reduced attack surface:** The production application runs as a non-root user and does not need the full build toolchain.
- **Repeatability:** The same Docker build process can be used by developers and CI/CD.
- **Configuration separation:** Environment-specific values are supplied at runtime.
- **Independent deployment:** Frontend and backend images can be built, published, and updated separately.

## CI/CD Pipeline

The GitHub Actions workflow is located at:

```text
.github/workflows/ci-cd.yml
```

The pipeline automates the delivery process from a Git event to deployment on EC2.

### Pipeline stages

| Stage | Responsibility |
|---|---|
| 1. Trigger | Starts the workflow on the configured Git push or pull request events. |
| 2. Checkout | Retrieves the application source from the repository. |
| 3. Build and test | Builds the backend and frontend and executes the configured tests. Where automated tests are absent, the pipeline should explicitly document the validation performed. |
| 4. AWS authentication | Obtains AWS credentials through the configured GitHub OIDC role, avoiding long-lived AWS access keys in the repository. |
| 5. Image build | Builds the backend and frontend Docker images. |
| 6. Image tagging | Tags images with the Git commit SHA so a specific build can be identified and redeployed. |
| 7. Publish | Authenticates to Amazon ECR and pushes both images to their respective repositories. |
| 8. Deploy | Uses AWS Systems Manager to execute the deployment process on the EC2 instance. |
| 9. Health verification | Checks the deployed application's health and fails the workflow if the configured verification fails. |

### Image tagging

The workflow uses the commit SHA to identify each build:

```text
<backend-ecr-repository>:<commit-sha>
<frontend-ecr-repository>:<commit-sha>
```

This makes it easier to identify which version was deployed and to select a previous image for rollback.

### AWS authentication and secrets

The pipeline should use GitHub Actions OIDC with the configured IAM role, `Github-Action-role`, and only the permissions required to publish images and invoke the deployment mechanism.

The EC2 instance uses its attached IAM role to access required AWS services. Database credentials and other application secrets are not stored in the repository or embedded in Docker images.

Configure the workflow's required GitHub variables, repository names, role identifiers, region, and deployment settings according to the actual workflow file.

### Deployment and failure handling

The deployment process should:

- Pull the images identified by the current commit SHA.
- Confirm that required production configuration exists before replacing running containers.
- Attach the containers to the intended Docker network.
- Start the backend with its protected environment configuration.
- Start the frontend on its configured web port.
- Perform a post-deployment health check.
- Return a failure status if deployment or health verification fails.

A failed health check should trigger investigation and, where the deployment script implements it, rollback to the previous known-good image. Review `FAILURE_AND_ROLLBACK.md` for the recovery procedure.

## AWS Deployment and Security

### EC2

The EC2 instance hosts the application containers and the monitoring stack. Docker is used to run the frontend, backend, Prometheus, Grafana, and Node Exporter.

The instance profile supplies AWS permissions to the instance. Access should be restricted to authorized administrators and the required application traffic.

### Amazon RDS for MySQL

The application database is hosted on Amazon RDS rather than inside an application container.

The RDS instance should remain private and accept MySQL connections only from the EC2 security group. Database credentials are supplied to the backend securely at runtime.

### Amazon ECR

Amazon ECR stores the backend and frontend images. The CI/CD workflow publishes commit-specific tags so that deployments can reference an identifiable application version.

### Security controls

- Restrict inbound EC2 ports to those required by the application and authorized administration.
- Restrict SSH access to approved source IP addresses if SSH is enabled.
- Keep the RDS instance private and restrict its inbound rules to the application security group.
- Apply least-privilege IAM permissions to the GitHub deployment role and EC2 instance role.
- Avoid long-lived AWS access keys in GitHub workflow files.
- Keep secrets out of source control, Docker build contexts, and image layers.
- Bind monitoring interfaces to localhost or protect them behind an approved access mechanism rather than exposing them publicly.

See `aws/aws-setup.md` for the deployment-specific setup and security-group documentation.

## Monitoring and Alerting

Prometheus and Grafana run as containers on the EC2 instance.

### Prometheus

The configuration file is maintained under the monitoring directory. Prometheus scrapes:

- **Backend metrics:** Spring Boot Actuator Prometheus endpoint at `/actuator/prometheus`.
- **Host metrics:** Node Exporter metrics for EC2 system monitoring.
- **Prometheus self-metrics:** Health and operational information about the monitoring server itself.

The scrape interval and target definitions are controlled by `prometheus.yml`.

### Grafana

Grafana uses Prometheus as its data source and provides an operations dashboard covering the following categories:

- Backend request rate.
- HTTP error rate.
- Request latency.
- EC2 CPU utilization.
- Available disk space.
- JVM memory usage.
- Backend and exporter availability.
- Container restart metrics, where a suitable container-metrics source is configured.

Some panels require the corresponding metrics to be present. For example, container restart panels require a source that exposes container-level metrics; Node Exporter alone does not provide complete container restart information.

### Alert rules

The configured Prometheus rules include:

- **TodoBackendDown:** Detects when the backend scrape target is unavailable.
- **TodoBackendHigh5xxRate:** Detects a sustained high proportion of HTTP 5xx responses.
- **EC2LowDiskSpace:** Detects low available disk space on the monitored root filesystem.

These rules are evaluated by Prometheus. To deliver notifications to email, Slack, or another destination, configure Alertmanager or another supported notification integration with an appropriate receiver.

Configuration files and operating procedures are maintained under `monitoring/` and in `MONITORING_AND_OPERATIONS.md`.

## Assumptions and Design Decisions

1. The existing application business logic remains unchanged except for changes needed to support environment-based configuration, health checks, and metrics.
2. The backend uses Java 21 and Spring Boot with Maven. The frontend is a React application managed through npm.
3. MySQL is hosted on Amazon RDS and is not deployed as a production application container.
4. EC2 is responsible for hosting the application and monitoring containers.
5. GitHub Actions is the selected CI/CD platform, Amazon ECR is the image registry, and AWS Systems Manager is the deployment mechanism.
6. Commit-SHA image tags are used to identify deployments and support recovery to a previous image version.
7. Application credentials and external service secrets are provided outside the source code.
8. The deployment assumes that the EC2 instance has Docker installed, the required IAM permissions, access to ECR, and network connectivity to RDS.
9. Automated tests are executed when available and configured. A successful build alone must not be represented as proof that all application tests passed.
10. Prometheus alert rules evaluate conditions but do not deliver external notifications unless a notification receiver is configured.
11. Monitoring dashboards can only display metrics exposed by the configured scrape targets. Missing metrics must be investigated rather than treated as healthy results.
12. AWS resource availability, security-group rules, IAM permissions, secret configuration, and billing depend on the actual deployed environment and must be verified independently.

## Troubleshooting

### Backend fails to start

Check the container logs:

```bash
docker logs --tail=100 backend
```

Verify the database endpoint, credentials, environment variables, network connectivity, and application configuration.

### Frontend cannot reach the backend

Check the frontend API URL, Nginx configuration, backend container status, Docker network connectivity, and browser console errors.

### GitHub Actions deployment fails

Review the failed workflow stage. Verify the OIDC role trust relationship, IAM permissions, ECR repository names, image tags, SSM permissions, EC2 managed-instance connectivity, and deployment logs.

### Prometheus target is down

Inspect the Prometheus target status and verify that the target name resolves on the monitoring network. Check the backend Actuator endpoint and Node Exporter health.

### Grafana dashboard shows no data

Confirm that the Prometheus data source is healthy, the relevant scrape targets are up, the dashboard queries match the available metric names, and sufficient metrics have been collected for the selected time range.

### Disk space is low

Inspect Docker images, unused containers, logs, and persistent monitoring data before removing anything. Avoid deleting running application containers or persistent volumes as a first response.

## Operational Documentation

Refer to these files for additional deployment and recovery details:

- `aws/architecture-diagram.png` — deployment architecture.
- `aws/aws-setup.md` — AWS networking, IAM, and security configuration.
- `monitoring/prometheus.yml` — Prometheus scrape configuration.
- `monitoring/alert-rules.yml` — Prometheus alert definitions.
- `monitoring/grafana-dashboard.json` — Grafana dashboard definition, if stored under this name.
- `FAILURE_AND_ROLLBACK.md` — deployment failure and recovery procedures.
- `MONITORING_AND_OPERATIONS.md` — metrics, logs, alerts, and operational practices.

---

**Note:** Verify all paths and filenames against the final repository before submission. Documentation describes the intended design; actual configuration files and successful pipeline runs are the source of truth for implementation status.
