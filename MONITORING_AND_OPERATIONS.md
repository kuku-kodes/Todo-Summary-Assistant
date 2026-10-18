# Monitoring and operations

## Metrics and why

- **Request rate:** `http_server_requests_seconds_count` rate indicates traffic and sudden demand changes.
- **HTTP 5xx rate:** fraction of server errors identifies backend failures; alert only after a sustained threshold to reduce noise.
- **Latency:** p95 from `http_server_requests_seconds_bucket` highlights slow requests that averages can hide.
- **Host CPU, memory and disk:** Node Exporter host metrics indicate resource pressure; low disk is particularly important because Docker logs and image layers can fill the root filesystem.
- **JVM memory:** heap used versus max helps identify memory pressure and potential leaks.
- **Container restarts and uptime:** cAdvisor's `container_start_time_seconds` and Prometheus `up` identify restart churn and missing targets. cAdvisor must be running and scraped for container panels to have data.

## Critical logs

1. Backend container logs: startup failures, stack traces, DB connection errors, HTTP errors and Cohere/Slack integration failures. Avoid logging credentials or sensitive request data.
2. Frontend/Nginx logs: access and upstream errors, particularly 502/504 responses.
3. GitHub Actions deployment logs: test/build results, image SHA, SSM command status and post-deployment health checks.
4. EC2 system, Docker daemon and SSM agent logs for host-level failures.
5. RDS events and MySQL error/slow-query logs where enabled.

## Alert policy

- `TodoBackendDown`: critical when the backend scrape target is unavailable for 1 minute.
- `TodoBackendHigh5xxRate`: warning when 5xx responses exceed 5% for 5 minutes.
- `EC2HighCPU`: warning when CPU remains above 85% for 10 minutes.
- `EC2LowDiskSpace`: warning when root filesystem free space is below 15% for 5 minutes.

Do not page on every single 4xx response, a brief CPU spike, or an alert whose target is intentionally stopped during maintenance. Group related alerts and tune thresholds against observed baseline traffic. Prometheus rules only mark alerts as pending/firing; notifications require Alertmanager or Grafana contact-point configuration.

## Early detection / routine checks

- Confirm Prometheus targets are `UP` and alert rules load after changing config.
- Confirm Grafana's Prometheus data source is healthy and dashboard panels return data over a recent time range.
- Verify `/actuator/health` and `/actuator/prometheus` from the Docker network after each release.
- Inspect the last CI/CD run and immutable image SHA; fail deployment if health checks do not pass.
- Review CPU, memory, disk, latency and 5xx trends routinely and before releases.
- Test rollback and database restore procedures periodically; monitoring dashboards do not replace backups.

## Known limits

The dashboard includes a container-restart panel, but it requires cAdvisor metrics. If cAdvisor is not running or its scrape target is down, that panel will show no data. This repository contains Prometheus rule definitions; notification delivery is not configured by default. The EC2 and RDS security group configuration must be verified in the AWS account.
