# CI/CD Deliverable — GitHub Actions Pipeline

## 1. Objective

Automate validation, container image creation, image publication to Amazon ECR, and deployment to the EC2 application host. The pipeline should be repeatable, fail on errors, avoid committing secrets, and check application health after deployment.

Pipeline definition:

```text
.github/workflows/ci-cd.yml
```

## 2. Trigger strategy

The workflow should run for pushes to the deployment branch and pull requests. A recommended safety rule is to run build/test validation on pull requests but publish and deploy only from trusted pushes to the designated deployment branch. Do not deploy untrusted pull-request code with production credentials.

## 3. Pipeline stages

### Stage 1 — Checkout

Use `actions/checkout` to retrieve the exact commit that triggered the workflow. The commit SHA becomes the immutable image tag, allowing the deployed version to be identified.

### Stage 2 — Backend validation

Set up the required Java 21 environment and Maven cache, then run the backend build and tests from `Backend/todo-summary-assistant`.

Example command:

```bash
./mvnw --batch-mode clean verify
```

Use `mvn` instead if the repository does not contain the Maven Wrapper. The workflow should fail if this command fails. If the current tests need external services, configure test dependencies properly rather than silently treating a failing test as successful.

### Stage 3 — Frontend validation

Set up the Node.js version supported by the frontend `package.json`, install dependencies with the repository's lockfile, and run the production build.

Example commands:

```bash
npm ci
npm run build
```

Run from `Frontend/todo`. Add lint or test commands only when the project defines them.

### Stage 4 — AWS authentication

Authenticate to AWS using GitHub Actions OIDC and an IAM role, rather than long-lived AWS access keys. The GitHub role should trust only the intended repository and branch/environment and have only the permissions required to push to the two ECR repositories and invoke the deployment mechanism.

Typical configuration values include:

- AWS region: `ap-south-1`
- Backend ECR repository: `todo-summary-backend`
- Frontend ECR repository: `todo-summary-frontend`

These are configuration identifiers, not credentials. Keep account-specific values in repository/environment variables where practical.

### Stage 5 — ECR login and image build

Log in to Amazon ECR, build both images from their respective Dockerfiles, and tag them with the full Git commit SHA. Push both images to ECR.

Conceptual tag format:

```text
<account-id>.dkr.ecr.<region>.amazonaws.com/todo-summary-backend:<commit-sha>
<account-id>.dkr.ecr.<region>.amazonaws.com/todo-summary-frontend:<commit-sha>
```

Immutable SHA tags improve traceability and make rollback to a known image possible. A mutable `latest` tag may be used as a convenience, but it should not be the only release identifier.

### Stage 6 — Deploy to EC2 through Systems Manager

Use AWS Systems Manager Run Command to execute a deployment script on the target EC2 instance. This avoids requiring inbound SSH access for CI/CD, provided the instance is managed by SSM and the instance role and network configuration are correct.

The deployment script should:

1. Validate required runtime configuration exists on the host.
2. Authenticate Docker to ECR using the instance's IAM role.
3. Pull the two images for the exact commit SHA.
4. Replace the backend and frontend containers in a controlled manner.
5. Attach containers to the intended Docker network.
6. Pass backend runtime configuration from a protected host-side environment file or secret store.
7. Wait for startup and run health checks.
8. Return a non-zero exit code on failure so the workflow reports a failed deployment.

Do not pass passwords, tokens, or full secret values as workflow command-line arguments or print them in logs.

### Stage 7 — Post-deployment health check

Check the backend health endpoint from an appropriate network location, for example:

```bash
curl --fail --silent --show-error http://127.0.0.1:8080/actuator/health
```

Use the actual host port mapping and network path from the deployment configuration. If the endpoint is not exposed on the EC2 host, run the check from within the application Docker network or through the deployment script. Also verify the frontend responds on its configured port. Fail the job if checks do not pass.

### Stage 8 — Failure handling and rollback

The deployment should stop on errors (`set -euo pipefail` in Bash scripts where appropriate), retain the last known-good image tags, and document how to redeploy a prior SHA. If health checks fail after replacement, the script should either automatically restore the previous containers or fail clearly and provide a documented rollback procedure. Do not claim automatic rollback unless it is implemented and tested.

## 4. Required GitHub configuration

Configure repository or environment settings for the deployment:

- GitHub OIDC permission: `id-token: write`
- Repository read permission: `contents: read`
- AWS role ARN / deployment target identifiers as appropriate
- Region and ECR repository names
- EC2 instance ID or SSM target identifiers
- Any non-secret deployment variables required by the workflow

Avoid static AWS access keys if OIDC is configured. Database credentials and API keys belong on the host or in AWS Secrets Manager / SSM Parameter Store, not in workflow YAML. Never commit a populated `.env` file.

## 5. Security and reliability controls

- Deploy only from trusted branches or protected environments.
- Restrict the GitHub OIDC role trust policy to the intended repository and ref.
- Grant ECR and SSM permissions with least privilege.
- Use commit-SHA image tags.
- Fail the workflow if build, push, deployment, or health-check commands fail.
- Avoid printing secrets in shell output.
- Keep runtime secrets outside the repository and Docker image.
- Keep a known-good image tag available for rollback.

## 6. How to inspect a run

1. Open the repository's **Actions** tab.
2. Select the workflow run associated with the commit.
3. Inspect backend validation, frontend validation, AWS authentication, image push, deployment, and health-check stages.
4. Confirm each required stage completed successfully and the deployed image SHA matches the triggering commit.
5. If deployment fails, inspect the SSM command output and EC2 container logs without sharing secret values.

## 7. Assumptions and implementation notes

- GitHub Actions is the CI/CD tool.
- AWS region is `ap-south-1`.
- Images are stored in Amazon ECR.
- EC2 deployment uses AWS Systems Manager.
- The application is a Spring Boot backend and React frontend, with MySQL hosted on Amazon RDS.
- Exact workflow job names, branch filters, IAM permissions, health-check URLs, and rollback behavior must match the actual `.github/workflows/ci-cd.yml` and deployment script in the repository.
