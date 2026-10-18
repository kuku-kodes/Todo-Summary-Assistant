# Docker Deliverable — Design and Usage

## 1. Objective

Containerize both parts of TodoSummaryAssistant:

- **Backend:** Spring Boot / Java 21 / Maven
- **Frontend:** React application served by Nginx in the production image

The Dockerfiles should build reproducible images, keep build tools out of runtime images, run as a non-root user where supported, and obtain environment-specific configuration at runtime.

## 2. Required files

```text
Backend/todo-summary-assistant/
├── Dockerfile
└── .dockerignore

Frontend/todo/
├── Dockerfile
└── .dockerignore
```

Keep each Dockerfile and `.dockerignore` beside the application it builds. Build contexts must match those paths.

## 3. Backend Docker design

The backend Dockerfile should use a multi-stage build:

1. **Build stage:** use a Java 21 JDK/Maven image to resolve dependencies, compile the source, run the configured tests, and package the application.
2. **Runtime stage:** copy only the packaged JAR into a smaller Java 21 runtime image.
3. **Least privilege:** create/use a dedicated non-root user and run the Java process as that user.
4. **Configuration:** inject database credentials and integration secrets through environment variables or a secret manager; do not copy populated `.env` files into the image.
5. **Runtime port:** expose/document port `8080` unless the application configuration overrides it.
6. **Failure visibility:** run the application as the container's main process so Docker can observe its exit status and logs.

Example build command, run from the repository root:

```bash
docker build \
  -f Backend/todo-summary-assistant/Dockerfile \
  -t todo-summary-backend:local \
  Backend/todo-summary-assistant
```

Example run command (provide the required values securely in your own shell/environment; do not paste secrets into documentation):

```bash
docker run --rm --name todo-summary-backend \
  -p 8080:8080 \
  --env-file /path/to/untracked-backend.env \
  todo-summary-backend:local
```

The environment file must exist locally, contain the exact variable names expected by the application, and remain outside version control.

## 4. Frontend Docker design

The frontend Dockerfile should also use multiple stages:

1. **Build stage:** use a Node.js image to install dependencies and create the optimized static production build.
2. **Runtime stage:** copy only the generated static assets into an Nginx image.
3. **Non-root runtime:** configure Nginx to run as a non-root user, including writable cache, PID, and temporary paths as needed.
4. **Small runtime image:** do not ship Node.js, source maps, development dependencies, or build tooling unless explicitly required.
5. **Networking:** serve the frontend on container port `80`; map a host or load-balancer port as required by the deployment.

Example build command:

```bash
docker build \
  -f Frontend/todo/Dockerfile \
  -t todo-summary-frontend:local \
  Frontend/todo
```

Example run command:

```bash
docker run --rm --name todo-summary-frontend -p 3000:80 todo-summary-frontend:local
```

If the frontend needs a backend API URL at build time, document and pass the appropriate build argument supported by its Dockerfile. React build-time variables are generally embedded into static assets, so they must never contain secrets.

## 5. `.dockerignore` design

Each application should have its own `.dockerignore` to reduce build context size and prevent accidental inclusion of local files.

Typical exclusions include:

```text
.git
.gitignore
node_modules
target
dist
build
coverage
.env
.env.*
!.env.example
*.log
.DS_Store
```

Adjust exclusions per application: for example, do not exclude `target` if a Dockerfile intentionally expects a prebuilt JAR, and do not exclude a required frontend build configuration. The recommended multi-stage design builds artifacts inside Docker, so local build output should generally be excluded.

## 6. Image validation

Build both images:

```bash
docker build -f Backend/todo-summary-assistant/Dockerfile \
  -t todo-summary-backend:local Backend/todo-summary-assistant

docker build -f Frontend/todo/Dockerfile \
  -t todo-summary-frontend:local Frontend/todo
```

Inspect the images and running containers:

```bash
docker image ls
docker run --rm todo-summary-backend:local id
docker run --rm todo-summary-frontend:local id
docker compose ps
```

The `id` commands are illustrative and may need an image-specific command override; validate the configured runtime user through image metadata or by inspecting the Dockerfile. Confirm that the actual application starts and that the configured health endpoints respond successfully.

## 7. Production considerations

- Do not bake database credentials, API keys, or webhook URLs into image layers.
- Use immutable commit-SHA tags in CI/CD; a `local` tag is for development only.
- Keep the database on private Amazon RDS for production.
- Use health checks and post-deployment validation.
- Configure Nginx SPA fallback only if the frontend uses client-side routing.
- Review image size and vulnerability scan results before release.

## 8. Assumptions

- Java 21 is the backend runtime.
- The backend packages a runnable Spring Boot JAR using Maven.
- The frontend package scripts produce static assets suitable for Nginx. Confirm the output directory (`dist` or `build`) from the frontend tooling.
- Port `8080` is used by the backend and port `80` by the Nginx frontend container unless the application configuration differs.
- Exact build arguments and environment variable names must match the Dockerfiles and application configuration committed in the repository.
