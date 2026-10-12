# Impactive Infrastructure

Infrastructure and deployment configuration for the Impactive platform, managed with Docker Compose.

This repository contains the Docker Compose configuration, Nginx reverse-proxy configuration, monitoring components, and environment-variable templates required to configure and operate the platform's infrastructure.

## Table of Contents

- Overview
- Repository Structure
- Prerequisites
- Getting Started
- Environment Configuration
- Docker Compose Commands
- Services and Infrastructure
- Monitoring and Logging
- Nginx Configuration
- Backup and Storage Configuration
- Troubleshooting
- Security Best Practices
- Maintenance

## Overview

The repository provides a centralized configuration for running the Impactive infrastructure using Docker Compose.

The configuration includes components for:

- Application routing and reverse proxying with Nginx.
- MongoDB database connectivity and administration.
- PostgreSQL database configuration.
- RabbitMQ messaging and questionnaire processing.
- Keycloak-based authentication and authorization.
- SMTP integration for email delivery.
- Grafana-based monitoring.
- Log collection and aggregation using Grafana Alloy and Loki.
- Backup configuration using Cloudflare R2-compatible object storage.

The actual services, networks, volumes, ports, and deployment profiles are defined in `compose.yaml`.

## Repository Structure

```
impactive-infrastructure/
├── compose.yaml
├── .dockerignore
├── .env.example
├── .gitignore
├── README.md
├── .github/
│   └── workflows/
│       └── validate.yml
├── monitoring/
│   ├── alloy/
│   │   └── config.alloy
│   ├── geoip/
│   ├── grafana/
│   │   └── provisioning/
│   │       └── datasources/
│   └── loki/
│       └── config.yaml
└── nginx/
    ├── Dockerfile
    ├── nginx.conf
    └── maintenance.html
```

### Key files and directories

| Path | Purpose |
| --- | --- |
| `compose.yaml` | Defines the containerized infrastructure and service configuration. |
| `.env.example` | Template for the environment variables used by the Compose configuration. |
| `.dockerignore` | Excludes selected files from Docker build contexts. |
| `.gitignore` | Specifies files that should not be tracked by Git. |
| `nginx/Dockerfile` | Defines the Nginx container image build. |
| `nginx/nginx.conf` | Contains reverse-proxy, routing, and related Nginx settings. |
| `nginx/maintenance.html` | Static maintenance page served when configured. |
| `monitoring/alloy/config.alloy` | Configures Grafana Alloy log collection and processing. |
| `monitoring/loki/config.yaml` | Configures Loki log aggregation. |
| `monitoring/grafana/provisioning/datasources/` | Directory reserved for Grafana datasource provisioning. |
| `monitoring/geoip/` | Directory reserved for GeoIP-related assets or configuration. |
| `.github/workflows/validate.yml` | Intended location for GitHub Actions validation. |

The monitoring directories and GitHub Actions workflow should be checked for completeness before relying on them in production.

## Prerequisites

Install the following tools on the host machine:

- Docker Engine.
- Docker Compose v2, available through the `docker compose` command.
- Git.
- Access to the container images and application services referenced by `compose.yaml`.
- Valid configuration values for the services being deployed.

Verify the installation:

```
docker --version
docker compose version
git --version
```

For production deployment, also ensure that DNS records, firewall rules, persistent storage, TLS certificates, and external service credentials are configured appropriately.

## Getting Started

### 1\. Clone the repository

If you have not already cloned the repository:

```
git clone <repository-url>
cd impactive-infrastructure
```

Replace `<repository-url>` with the actual Git repository URL.

If the repository is already available locally, navigate to its directory:

```
cd /path/to/impactive-infrastructure
```

### 2\. Create the environment file

The Compose configuration references numerous environment variables. The repository includes `.env.example` as the configuration template.

Create the local environment file:

```
cp .env.example .env
```

Edit it using your preferred editor:

```
nano .env
```

Populate the required values using the appropriate development, staging, or production configuration.

**Important:** Do not commit `.env` to version control. It may contain database credentials, authentication secrets, SMTP passwords, and object-storage access keys.

### 3\. Validate the configuration

Run:

```
docker compose -f compose.yaml config --quiet
```

This checks whether Docker Compose can parse and validate the configuration.

If Compose reports that variables are unset or that `.env` is missing, complete the environment setup before continuing.

To inspect the resolved configuration, use:

```
docker compose -f compose.yaml config
```

The resolved output may contain interpolated credentials. Do not publish it or include it in logs, issue reports, or commits without removing sensitive values.

### 4\. Start the infrastructure

After configuring the environment:

```
docker compose -f compose.yaml up -d
```

To build images where the Compose configuration defines a build context:

```
docker compose -f compose.yaml up -d --build
```

Check the running containers:

```
docker compose -f compose.yaml ps
```

Inspect the service logs:

```
docker compose -f compose.yaml logs --tail=100
```

The actual startup sequence and available services depend on the definitions in `compose.yaml`.

### 5\. Verify service health

Check the container status:

```
docker compose -f compose.yaml ps -a
```

Follow logs for a specific service:

```
docker compose -f compose.yaml logs -f <service-name>
```

Replace `<service-name>` with a service defined in `compose.yaml`.

A running container does not necessarily mean its application is ready. Check application logs, health checks, and connectivity to dependent services where applicable.

## Environment Configuration

Docker Compose reads environment variables from the shell and, by default, from a project `.env` file when interpolating the Compose configuration. Services may also use explicit `env_file` declarations.

The `.env.example` file is the reference for the variables expected by this repository.

The following table groups variables observed in the Compose configuration. Exact requirements and defaults must be verified against the relevant service definitions and application documentation.

### MongoDB

| Variable | Purpose |
| --- | --- |
| `MONGO_HOST` | MongoDB hostname. |
| `MONGO_PORT` | MongoDB port. |
| `MONGO_INITDB_DATABASE` | Database used during MongoDB initialization. |
| `MONGO_INITDB_ROOT_USERNAME` | Initial MongoDB root username. |
| `MONGO_INITDB_ROOT_PASSWORD` | Initial MongoDB root password. |
| `MONGO_ADMIN_USER` | Administrative database username. |
| `MONGO_ADMIN_PASSWORD` | Administrative database password. |
| `MONGO_DB_ADMIN` | Administrative database name. |
| `MONGO_DB` | Application database name. |
| `MONGO_ADMIN_USER_DEV` | Development-specific administrative username. |
| `MONGO_ADMIN_PASSWORD_DEV` | Development-specific administrative password. |
| `MONGO_DB_ADMIN_DEV` | Development-specific administrative database name. |

MongoDB initialization variables are generally applied when initializing a new database directory. Changing them later does not automatically change credentials in an existing database.

### PostgreSQL

| Variable | Purpose |
| --- | --- |
| `POSTGRES_USER` | PostgreSQL database username. |
| `POSTGRES_PASSWORD` | PostgreSQL database password. |

Additional database names, ports, and connection settings should be documented if they are introduced or required by the actual service configuration.

### RabbitMQ

| Variable | Purpose |
| --- | --- |
| `RABBITMQ_HOST` | RabbitMQ hostname. |
| `RABBITMQ_PORT` | RabbitMQ port. |
| `RABBITMQ_DEFAULT_USER` | Default RabbitMQ username. |
| `RABBITMQ_DEFAULT_PASS` | Default RabbitMQ password. |
| `QUESTIONNAIRE_QUEUE` | Queue name used for questionnaire processing. |

Ensure that application services use the same broker address, credentials, and queue configuration.

### Keycloak and authentication

| Variable | Purpose |
| --- | --- |
| `KEYCLOAK_URL` | Keycloak server URL. |
| `KEYCLOAK_ADMIN_REALM` | Administrative realm. |
| `KEYCLOAK_USERS_REALM` | User authentication realm. |
| `KEYCLOAK_ADMIN_CLIENT_ID` | Administrative client identifier. |
| `KEYCLOAK_ADMIN_SECRET` | Administrative client secret. |
| `KEYCLOAK_ADMIN_ISSUER` | Administrative token issuer. |
| `KEYCLOAK_ADMIN_ISSUER_BACKEND_USER` | Backend user issuer configuration. |
| `KEYCLOAK_USERS_CLIENT_ID` | User-facing client identifier. |
| `KEYCLOAK_USERS_SECRET` | User-facing client secret. |
| `KEYCLOAK_CLIENT_ID` | Default or application client identifier. |
| `KEYCLOAK_CLIENT_SECRET` | Default or application client secret. |
| `KEYCLOAK_ISSUER` | Authentication issuer URL. |
| `KEYCLOAK_CLIENT_ID_USER` | User application client identifier. |
| `KEYCLOAK_CLIENT_SECRET_USER` | User application client secret. |
| `KEYCLOAK_ISSUER_USER` | User authentication issuer URL. |
| `KC_BOOTSTRAP_ADMIN_USERNAME` | Initial Keycloak administrator username. |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | Initial Keycloak administrator password. |

Keep client IDs, realms, and issuer URLs consistent with the Keycloak realm and client configuration.

Never use production administrator credentials in local development or expose client secrets to frontend code.

### Frontend and session configuration

| Variable | Purpose |
| --- | --- |
| `FRONT_END_HOST` | User-facing frontend host. |
| `FRONT_END_ADMIN_HOST` | Administrative frontend host. |
| `NEXTAUTH_URL` | Base URL for the default authentication application. |
| `NEXTAUTH_SECRET` | Secret used to protect authentication sessions. |
| `NEXTAUTH_URL_USER` | Base URL for the user-facing authentication application. |
| `NEXTAUTH_SECRET_USER` | Secret used for user-facing authentication sessions. |
| `AUTH_TRUST_HOST` | Configures whether the authentication application trusts the incoming host. |

Use the externally accessible URLs appropriate to the environment. Authentication callback URLs, proxy headers, and trusted-host settings must agree with the reverse-proxy configuration.

### SMTP

| Variable | Purpose |
| --- | --- |
| `SMTP_HOST` | SMTP server hostname. |
| `SMTP_PORT` | SMTP server port. |
| `SMTP_USER` | SMTP authentication username. |
| `SMTP_PASSWORD` | SMTP authentication password. |

Use the correct SMTP security settings for the selected mail provider. Do not assume that a port number alone enables TLS or STARTTLS.

### Grafana

| Variable | Purpose |
| --- | --- |
| `GRAFANA_ADMIN_USER` | Grafana administrator username. |
| `GRAFANA_ADMIN_PASSWORD` | Grafana administrator password. |

Use a unique, strong password and restrict access to the Grafana interface.

### Backup and object storage

| Variable | Purpose |
| --- | --- |
| `BACKUP_ENCRYPTION_KEY` | Key used by the backup process for encryption. |
| `R2_ACCOUNT_ID` | Cloudflare account identifier. |
| `R2_ENDPOINT` | R2-compatible object-storage endpoint. |
| `R2_BUCKET` | Bucket used for backups or stored objects. |
| `R2_ACCESS_KEY_ID` | Object-storage access key ID. |
| `R2_SECRET_ACCESS_KEY` | Object-storage secret access key. |

The exact backup implementation and encryption behavior must be verified against the scripts or services that consume these variables.

Protect encryption keys independently from the encrypted backups. Losing an encryption key may make the corresponding backups unrecoverable.

## Docker Compose Commands

Run these commands from the repository root.

### Start services

```
docker compose -f compose.yaml up -d
```

### Stop services

```
docker compose -f compose.yaml stop
```

### Stop and remove containers and networks

```
docker compose -f compose.yaml down
```

This does not normally remove named volumes. Persistent data may remain after containers are removed.

### Rebuild services

```
docker compose -f compose.yaml up -d --build
```

### List services and container status

```
docker compose -f compose.yaml ps
```

### Follow logs

```
docker compose -f compose.yaml logs -f
```

For a single service:

```
docker compose -f compose.yaml logs -f <service-name>
```

### Restart a service

```
docker compose -f compose.yaml restart <service-name>
```

### Validate Compose configuration

```
docker compose -f compose.yaml config --quiet
```

### List defined services

```
docker compose -f compose.yaml config --services
```

### Pull configured images

```
docker compose -f compose.yaml pull
```

### Remove containers and networks

```
docker compose -f compose.yaml down
```

**Warning:** Avoid `docker compose down -v` unless you explicitly intend to delete the associated named and anonymous volumes. Doing so may permanently remove database and other persistent data.

## Services and Infrastructure

### Reverse proxy

Nginx is configured under `nginx/` and provides the entry point for HTTP routing and related web traffic.

Its configuration is defined in `nginx/nginx.conf`. The repository also contains a custom `maintenance.html` page.

Review the Nginx configuration before deployment to verify:

- Upstream service names and ports.
- Hostname-based routing.
- TLS termination and certificate paths.
- Forwarded headers and client IP handling.
- Request size and timeout limits.
- Access restrictions for administrative endpoints.
- Maintenance-page behavior.

### Databases

The Compose environment references MongoDB and PostgreSQL configuration.

Database availability, initialization, and persistence depend on the service definitions and volume mappings in `compose.yaml`.

Before deploying a new environment, verify that database storage is persistent and that backups can be restored.

### Authentication

Keycloak-related variables support authentication and authorization integration.

Ensure that the configured realm, clients, secrets, issuer URLs, and callback URLs match the deployed Keycloak configuration.

### Messaging

RabbitMQ-related settings configure the message broker and questionnaire queue integration.

Verify that the relevant application services can reach the broker and that queue permissions and names are consistent.

### Email delivery

SMTP settings provide the connection details for outgoing email.

Test email delivery in a non-production environment before relying on password resets, notifications, or other email-dependent functionality.

## Monitoring and Logging

The repository contains configuration for Grafana Alloy and Loki, alongside Grafana datasource provisioning directories.

### Grafana Alloy

Configuration file:

```
monitoring/alloy/config.alloy
```

Alloy can collect, process, and forward telemetry according to its configured pipelines.

Check the configuration to confirm which containers, log sources, labels, and destinations are enabled.

### Loki

Configuration file:

```
monitoring/loki/config.yaml
```

Loki provides log aggregation and querying for configured log sources.

Review retention, storage paths, authentication, and network exposure before production deployment.

### Grafana

Grafana is configured through the Compose environment and has a datasource provisioning directory:

```
monitoring/grafana/provisioning/datasources/
```

Add valid datasource provisioning files if automatic datasource setup is required.

A directory existing in the repository does not, by itself, guarantee that Grafana datasources or dashboards are provisioned.

### Monitoring verification

List the available services:

```
docker compose -f compose.yaml config --services
```

Check container status:

```
docker compose -f compose.yaml ps
```

Inspect logs for the relevant monitoring service:

```
docker compose -f compose.yaml logs --tail=200 <service-name>
```

Confirm that telemetry is reaching the configured backend and that dashboards can query the expected datasource.

## Nginx Configuration

The Nginx configuration is maintained in:

```
nginx/nginx.conf
nginx/Dockerfile
nginx/maintenance.html
```

After modifying the configuration, validate it inside the running Nginx container, using the actual service name from Compose:

```
docker compose -f compose.yaml exec <nginx-service> nginx -t
```

If the configuration is valid, reload Nginx:

```
docker compose -f compose.yaml exec <nginx-service> nginx -s reload
```

If the container does not support these commands or is not running, rebuild or restart the relevant service as appropriate.

When changing routing rules, verify that upstream service names resolve on the correct Docker network and that the forwarded host and protocol headers are correct.

## Backup and Storage Configuration

The environment template includes variables for backup encryption and Cloudflare R2-compatible object storage.

Before enabling backups:

1. Configure the object-storage endpoint, bucket, and credentials.
2. Configure the backup encryption key using the documented format expected by the backup implementation.
3. Restrict storage credentials to the minimum permissions required.
4. Verify that backups are uploaded successfully.
5. Test restoration into an isolated environment.
6. Establish retention and recovery procedures appropriate to the application.

The presence of backup-related variables does not establish that a backup job is scheduled or operational. Verify the relevant service, script, or external scheduler.

Do not delete database volumes or existing backups during routine maintenance without a verified recovery plan.

## Troubleshooting

### Docker Compose reports that `.env` is missing

Example:

```
env file .../.env not found
```

Create the local environment file:

```
cp .env.example .env
nano .env
```

Fill in the required values and validate again:

```
docker compose -f compose.yaml config --quiet
```

If the Compose file contains an explicit `env_file` reference, ensure that the referenced file exists at the expected path.

### Docker Compose warns that variables are not set

Example:

```
The "MONGO_HOST" variable is not set.
Defaulting to a blank string.
```

These warnings indicate that Compose could not resolve the referenced variables from its interpolation environment.

Check the variable names in `.env.example` against the references in `compose.yaml`. Also check for spelling differences, missing assignments, and shell variables that override values from `.env`.

Do not silence warnings by assigning empty strings to required configuration values.

### Configuration validation fails

Run:

```
docker compose -f compose.yaml config
```

Inspect the error and correct invalid YAML, unresolved file paths, or other configuration issues.

The command may display sensitive interpolated values. Review its output before sharing it.

### A container exits unexpectedly

Check:

```
docker compose -f compose.yaml ps -a
docker compose -f compose.yaml logs --tail=200 <service-name>
```

Look for missing configuration, failed health checks, invalid credentials, unavailable dependencies, or storage permission errors.

### Services cannot communicate

Verify the service names, exposed ports, Docker networks, and internal connection URLs in `compose.yaml`.

Containers on the same Docker network should generally connect using the target service name and its container listening port, rather than a host-published port.

### Authentication fails

Check the Keycloak URL, realm, client ID, client secret, issuer, and frontend callback URL.

Also verify that the reverse proxy preserves the required host and protocol information and that Keycloak is configured for the deployment's external URL.

### Email delivery fails

Verify the SMTP hostname, port, credentials, encryption requirements, and provider restrictions.

Inspect application logs for connection failures and authentication errors.

### Monitoring data is missing

Check the Alloy configuration, Loki configuration, Grafana datasource provisioning, and container logs.

Confirm that the configured destinations are reachable and that the log pipeline is enabled.

## Security Best Practices

- Never commit `.env`, passwords, API keys, or encryption keys.
- Use unique, strong credentials for each environment.
- Rotate credentials when exposure is suspected.
- Restrict database, message broker, Grafana, and administrative endpoints to trusted networks.
- Use HTTPS for externally accessible services.
- Validate proxy headers and trusted-host settings.
- Apply least-privilege access policies to databases and object storage.
- Keep container images and host software updated.
- Configure persistent storage and tested backups for important data.
- Review log contents to avoid collecting passwords, access tokens, or other sensitive information.
- Avoid publishing resolved Compose configuration or container environment dumps.
- Verify restore procedures before relying on backups for disaster recovery.

## Maintenance

Before changing infrastructure configuration:

1. Review the intended changes.
2. Back up important persistent data when appropriate.
3. Validate the Compose configuration.
4. Test changes in a non-production environment.
5. Deploy using the relevant Compose commands.
6. Inspect service logs and health checks.
7. Verify application routing, authentication, messaging, database access, and monitoring.
8. Document any environment-specific requirements.

Validate the configuration with:

```
docker compose -f compose.yaml config --quiet
```

Review the configured services with:

```
docker compose -f compose.yaml config --services
```

---

## License

Add the project's applicable license and copyright information here.

## Support

For infrastructure issues, consult the repository configuration and service logs first. Include relevant error messages and deployment context when reporting an issue, but remove credentials, tokens, private endpoints, and other sensitive information.

## Apply it to your repository

From your current repository directory, replace the existing README using the following commands:

bash
````
cat > README.md <<'EOF'
# Impactive Infrastructure

Infrastructure and deployment configuration for the Impactive platform, managed with Docker Compose.

This repository contains the Docker Compose configuration, Nginx reverse-proxy configuration, monitoring components, and environment-variable templates required to configure and operate the platform's infrastructure.

## Repository Structure

- `compose.yaml` — Docker Compose infrastructure configuration.
- `.env.example` — Environment-variable template.
- `nginx/` — Nginx Dockerfile, routing configuration, and maintenance page.
- `monitoring/alloy/config.alloy` — Grafana Alloy configuration.
- `monitoring/loki/config.yaml` — Loki configuration.
- `monitoring/grafana/provisioning/datasources/` — Grafana datasource provisioning directory.
- `.github/workflows/validate.yml` — Intended location for CI validation.

## Prerequisites

- Docker Engine
- Docker Compose v2
- Valid environment configuration for the services being deployed

## Getting Started

Create your local environment file:

```bash
cp .env.example .env
nano .env
````

Populate the variables with valid values for your environment. Do not commit `.env` or expose its secrets.

Validate the Compose configuration:

bash
```
docker compose -f compose.yaml config --quiet
```

List the configured services:

bash
```
docker compose -f compose.yaml config --services
```

Start the infrastructure:

bash
```
docker compose -f compose.yaml up -d
```

Inspect service status and logs:

bash
```
docker compose -f compose.yaml ps
docker compose -f compose.yaml logs --tail=100
```

## Environment Configuration

The environment template contains settings for MongoDB, PostgreSQL, RabbitMQ, Keycloak, frontend authentication, SMTP, Grafana, and backup storage.

Ensure that all required variables are populated. Docker Compose warnings about unset variables mean that configuration values could not be resolved and may default to empty strings.

## Nginx

Nginx configuration is maintained in `nginx/nginx.conf`. The directory also contains a Dockerfile and a maintenance page.

Validate the configuration inside the Nginx container:

bash
```
docker compose -f compose.yaml exec <nginx-service> nginx -t
```

Replace `<nginx-service>` with the actual service name defined in `compose.yaml`.

## Monitoring and Logging

Monitoring configuration is maintained under `monitoring/`:

- Grafana Alloy configuration: `monitoring/alloy/config.alloy`
- Loki configuration: `monitoring/loki/config.yaml`
- Grafana datasource provisioning: `monitoring/grafana/provisioning/datasources/`

Verify the configured collection pipelines, destinations, datasources, and retention settings before production use.

## Common Commands

Start services:

bash
```
docker compose -f compose.yaml up -d
```

Rebuild services:

bash
```
docker compose -f compose.yaml up -d --build
```

Follow logs:

bash
```
docker compose -f compose.yaml logs -f
```

Restart a service:

bash
```
docker compose -f compose.yaml restart <service-name>
```

Stop services:

bash
```
docker compose -f compose.yaml stop
```

Remove containers and networks:

bash
```
docker compose -f compose.yaml down
```

Avoid `docker compose down -v` unless you intentionally want to remove persistent volumes.

## Backup and Security

The environment template includes variables for backup encryption and R2-compatible object storage. Verify the backup implementation, encryption behavior, retention policy, and restore procedure before relying on it.

Use unique credentials, restrict administrative endpoints, enable HTTPS for public services, and never commit secrets or publish resolved Compose configuration containing sensitive values.

## Troubleshooting

If `.env` is missing, create it from `.env.example`.

If variables are unset, check their names and values against the Compose configuration.

If a service fails to start, inspect:

bash
```
docker compose -f compose.yaml ps -a
docker compose -f compose.yaml logs --tail=200 <service-name>
```

If services cannot communicate, verify Docker networks, service names, ports, and connection URLs.

## Maintenance

Validate the Compose configuration before deploying changes. Test infrastructure changes in a non-production environment and verify service health, routing, authentication, database connectivity, messaging, and monitoring after deployment.

