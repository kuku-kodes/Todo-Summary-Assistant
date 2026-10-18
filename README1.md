# Todo Summary Assistant — DevOps Assessment Submission

A Spring Boot + React Todo application packaged into non-root Docker containers and deployed to AWS EC2. MySQL is hosted by Amazon RDS; images are stored in Amazon ECR; GitHub Actions builds/tests, publishes immutable commit-SHA tags, and deploys via AWS Systems Manager (SSM). Prometheus and Grafana provide application and host observability.

## Architecture

See [`aws/architecture-diagram.png`](aws/architecture-diagram.png) and [`aws/aws-setup.md`](aws/aws-setup.md).

```text
GitHub push -> GitHub Actions -> tests/build -> ECR (SHA tags) -> SSM -> EC2 Docker
                                                                      |-- frontend :80
                                                                      |-- backend 127.0.0.1:8080
                                                                      |-- Prometheus :9090 (localhost only)
                                                                      |-- Grafana :3000 (localhost only)
                                                                      `-- Node Exporter / cAdvisor metrics
Backend -> private RDS MySQL :3306
```

## Local prerequisites

- Docker Engine + Docker Compose plugin
- Java 17 / Maven and Node.js 20 if running without Docker
- A MySQL database and valid Cohere/Slack integration values if using those features

## Build the application images

```bash
docker build -t todo-summary-backend:local Backend/todo-summary-assistant
docker build -t todo-summary-frontend:local Frontend/todo
```

The backend reads DB, Cohere and Slack settings from environment variables. Copy `.env.example` to an ignored local env file and fill it with local-only values. Never commit real values. For the frontend container, Nginx serves the React build and proxies `/api/` to the backend service on `todo-app-network`.

## CI/CD configuration

Workflow: [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml).

1. Pull requests and pushes run Maven tests and a React production build.
2. Pushes to `main` or `master` authenticate to AWS through GitHub OIDC, build both images, and push immutable `${GITHUB_SHA}` tags to ECR.
3. The workflow invokes AWS SSM to pull the exact tags on EC2, replace application containers, and check backend Actuator health and frontend HTTP response.

Required GitHub configuration:

- Secret `AWS_ROLE_ARN`: IAM role ARN trusted for this repository's GitHub OIDC identity.
- Secret `EC2_INSTANCE_ID`: deployment EC2 instance ID.
- Variable `AWS_REGION`: optional, defaults to `ap-south-1`.
- The role needs ECR push permissions and narrowly scoped `ssm:SendCommand`, `ssm:GetCommandInvocation` permissions. The EC2 instance profile needs ECR pull and Systems Manager permissions.
- The EC2 instance must have Docker, AWS CLI, SSM agent, the `todo-app-network` Docker network, and `/opt/todo-assistant/backend.env` provisioned securely before the first deployment. The file must include `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `COHERE_API_KEY`, and `SLACK_WEBHOOK_URL` and be readable only by root.
- ECR repositories must exist before the first run.

**Important:** Review the account-specific IAM trust/permission policies and run a workflow after setting these repository variables. This source archive cannot prove that external AWS/GitHub settings are correct.

## Monitoring

Monitoring configuration lives in [`monitoring/`](monitoring/):

- `prometheus.yml`: scrapes Prometheus, backend Actuator metrics, Node Exporter and cAdvisor.
- `alert-rules.yml`: backend unavailable, sustained high 5xx rate, sustained high CPU, low disk.
- `grafana/dashboards/todo-assistant.json`: request rate, 5xx rate, p95 latency, CPU, disk, JVM heap, container restarts, and scrape availability.
- Grafana data source and dashboard provider are provisioned from files.

The monitoring compose file is intended to run on the EC2 host after the application Docker network and backend exist:

```bash
cd monitoring
# Set GRAFANA_ADMIN_PASSWORD in an ignored .env file before starting.
docker compose up -d
```

Prometheus/Grafana ports are bound to loopback by design. Access them with SSM port forwarding or an SSH tunnel; do not open these ports publicly. Node Exporter runs in host network mode. cAdvisor is optional and requires Docker host access. The container-restart panel will be empty if cAdvisor is not running or its scrape target is down. Prometheus alert rules do not send notifications without Alertmanager or configured Grafana contact points.

## Security notes

- No real application credentials are included in this submission. Keep environment files out of Git.
- RDS should be private and allow TCP 3306 only from the EC2 security group.
- Expose only application port 80 and tightly restricted administrative access. Keep 3000, 9090 and 9100 private.
- Backend port 8080 is bound to EC2 loopback; frontend proxies API calls through the Docker network.
- The Dockerfiles use multi-stage builds and run the application process as non-root.

## Operations and rollback

See [`FAILURE_AND_ROLLBACK.md`](FAILURE_AND_ROLLBACK.md) and [`MONITORING_AND_OPERATIONS.md`](MONITORING_AND_OPERATIONS.md) for rollback, failure recovery, secret rotation, database recovery, key metrics, logs, and alert thresholds.

## Assumptions / changes

- Added Spring Boot Actuator and Micrometer Prometheus registry to expose `/actuator/health` and `/actuator/prometheus` for DevOps health checks and scraping.
- Replaced hard-coded datasource and integration settings with environment-variable placeholders.
- Changed the React API base URL to `/api/todos`; Nginx forwards `/api/` to the backend, avoiding a hard-coded browser-visible backend address.
- This repository contains the application source and reproducible configuration templates. Verify live AWS security groups/RDS settings, actual GitHub Actions execution, Grafana panel data, and alert notification delivery in the target account before claiming those external states are complete.
