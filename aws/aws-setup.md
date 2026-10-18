# AWS setup and security notes

## Deployed topology

- Region: `ap-south-1` (Mumbai).
- Application host: EC2 instance `todo-assistant-server` (`i-016f6c8860d0dcedc`).
- Database: Amazon RDS for MySQL, database `todo_db`, intended to remain private.
- Container registry: ECR repositories `todo-summary-backend` and `todo-summary-frontend`.
- EC2 instance profile: `todo-ec2-role`; GitHub Actions federation role: `Github-Action-role`.
- Application containers share the Docker network `todo-app-network`. Backend port 8080 is bound to loopback on the host; frontend is published on port 80. Prometheus and Grafana are bound to `127.0.0.1` on ports 9090 and 3000 respectively.

## Network/security group checklist

1. Place RDS in private subnets and set **Publicly accessible = No**.
2. RDS inbound: TCP 3306 from the EC2 application's security group only, not `0.0.0.0/0`.
3. EC2 inbound: TCP 80 from the expected reviewer/client range; TCP 22 only from an administrator IP if SSH is required. Prefer Systems Manager Session Manager over public SSH where configured.
4. Do not open TCP 3000, 9090, or 9100 to the internet. Use an SSM port-forwarding session or an SSH tunnel for private administration.
5. Attach the EC2 instance role for ECR image pulls and Systems Manager. Do not put AWS access keys on the instance.
6. GitHub Actions should use GitHub OIDC to assume a role restricted to the repository and `main`/`master` branch. Grant only ECR push and SSM deployment permissions needed by the workflow.
7. Store runtime values in `/opt/todo-assistant/backend.env` with owner root and mode `600`; never commit this file. Store GitHub deployment settings in GitHub Actions secrets/variables.

## Runtime environment

`backend.env` must contain `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`, `COHERE_API_KEY`, and `SLACK_WEBHOOK_URL`. Use the real RDS endpoint and rotate any values previously committed or shared. A placeholder `.env.example` is provided; it contains no real credentials.

## Reproducibility and assumptions

- The application is deployed as separate backend and frontend containers; MySQL remains on RDS.
- The workflow expects an EC2 instance managed by Systems Manager, AWS CLI available on that instance, Docker installed, a pre-created `todo-app-network`, and `/opt/todo-assistant/backend.env` provisioned securely before the first deployment.
- GitHub repository settings must define secret `AWS_ROLE_ARN`, secret `EC2_INSTANCE_ID`, and variable `AWS_REGION` (optional; defaults to `ap-south-1`). ECR repositories must already exist.
- Verify the real security group rules and RDS subnet/public-access settings in the AWS console before submission; those values cannot be proven from this repository alone.
