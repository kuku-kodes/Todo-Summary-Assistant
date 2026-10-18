# Failure and rollback runbook

## 1. A faulty version is deployed

Images are tagged with the immutable Git commit SHA. Identify the last healthy SHA from the GitHub Actions run history and deploy those two exact ECR tags. Re-run backend `/actuator/health` and frontend `/` checks, then inspect application logs. Keep the previous image tags in ECR until the new release has passed its observation window. For a production-grade enhancement, make the workflow retain previous tags and automate a rollback command when post-deploy checks fail.

## 2. The application crashes after deployment

Docker uses `restart: unless-stopped` to restart a stopped container, but this alone does not guarantee recovery from a bad release. Check `docker ps -a`, `docker logs backend`, `docker logs frontend`, Prometheus `up` metrics and the Grafana dashboard. The CI/CD post-deployment health checks fail the run if the backend or frontend endpoint does not respond. Roll back to the previous known-good SHA if the health check fails.

## 3. CI/CD is unavailable

Use the last known-good immutable ECR tags and the approved operator procedure from an authenticated administrative session on the EC2 host. Pull the selected SHA and recreate only the application containers, preserving `/opt/todo-assistant/backend.env`, `todo-app-network`, monitoring containers, and persistent monitoring volumes. Record the operator, reason, image SHA, health-check results and time. Never paste credentials into shell history or tickets.

## 4. Secrets are leaked

Immediately revoke/rotate the exposed DB password, Cohere key, Slack webhook and/or AWS credentials. Remove the compromised value from the source and CI secret store, audit CloudTrail and provider logs, review Git history and access, update the EC2 environment file through a secure channel, restart the affected service, and verify health. Treat a value as compromised even after deleting it from the latest commit.

## 5. EC2 fails

Provision or recover an instance from the documented configuration, attach the correct instance profile and security groups, install Docker/AWS CLI/SSM agent, restore the protected backend environment file from the approved secret store, recreate `todo-app-network`, and deploy the last healthy ECR SHA. Persistent user data remains in RDS; monitoring history survives only if its volumes are backed up or retained separately.

## 6. RDS is unavailable

Todo CRUD and summaries that depend on the database will fail or time out. Confirm RDS status, connectivity, security group ingress from the EC2 security group, subnet routes, DNS and DB connection limits. Restore from the latest automated backup or snapshot if corruption/loss occurred. Multi-AZ can reduce recovery time for supported failures but does not replace backups. Validate the restored database before switching application traffic and document data-loss window / recovery time.

## Deployment safety notes

- Runtime secrets must be outside source control.
- Verify the selected image SHA and health endpoints after each manual rollback.
- Avoid deleting monitoring volumes or changing the monitoring stack during application recovery.
- Prometheus alert rules evaluate conditions but do not send notifications unless an Alertmanager receiver or Grafana contact point is separately configured.
