# DevOps Deployment Home Task

## Overview

This project provisions and deploys a monitored infrastructure stack to a remote VPS. Ansible is used for server configuration and deployment, Docker Compose runs the application and monitoring services, and Kubernetes manifests provide a basic local Kubernetes deployment.

The remote test server is a VirtualBox VM at `192.168.0.30`. Ansible runs from the development laptop and connects to the VM over SSH using a key. The same laptop also runs the self-hosted GitHub Actions runner used by the deployment job.

## Accessing the VPS

The VM is on a private home network behind Carrier-Grade NAT, so it does not have a publicly reachable IP address and normal port forwarding is not available. SSH access from outside the local network is provided through a Cloudflare Tunnel.

To connect through the tunnel:

1. Install `cloudflared` for your operating system.
2. Start the local TCP proxy:

```bash
cloudflared access tcp --hostname tasks-creates-pentium-secretariat.trycloudflare.com --url localhost:2222
```

3. In a second terminal, connect with the provided private key:

```bash
ssh -i id_vps_interview -p 2222 user@localhost
```

SSH password login is disabled on the VM, so the SSH private key is required. The key is provided separately and is not committed to this repository. The Cloudflare `trycloudflare.com` quick tunnel is intended for temporary access and may need to be restarted if it expires.

## Architecture

```text
Developer laptop
	|
	| Ansible over SSH
	v
VirtualBox VPS (192.168.0.30)
	|
	+-- Nginx HTTP :80 / HTTPS :443
	+-- PostgreSQL :5432
	+-- Prometheus :9090
	+-- Grafana :3000
	+-- Alertmanager :9093
	+-- Node Exporter
	+-- Blackbox Exporter :9115
	+-- Loki :3100
	+-- Fluent Bit
	+-- PostgreSQL backup service

Minikube cluster on the VirtualBox VPS
	+-- Nginx Deployment
	+-- NodePort Service
	+-- TLS Ingress
	+-- HorizontalPodAutoscaler
```

## What Was Created

### Ansible

`ansible/playbook.yml` performs the following actions on the VPS:

- Updates operating system packages.
- Installs Docker, Docker Compose, OpenSSL and required dependencies.
- Enables and starts Docker.
- Creates deployment directories under `/opt/vps-infrastructure`.
- Copies the project files to the VPS.
- Creates a protected Compose `.env` file from credentials supplied by the controller environment.
- Generates a self-signed SSL certificate when one does not already exist.
- Disables SSH password authentication.
- Starts or recreates the Docker Compose stack.

The playbook can be rerun safely. Existing certificates are not regenerated, and credentials are not stored in the repository.

### Docker Compose

`project/docker-compose.yml` defines the complete stack:

- Nginx with separate HTTP and HTTPS pages.
- PostgreSQL 15 with automatic database initialization.
- A PostgreSQL backup container with daily backups and seven-day retention.
- Prometheus, Node Exporter and Blackbox Exporter for metrics and availability checks.
- Alertmanager for alert routing.
- Grafana for dashboards.
- Loki for log storage and Fluent Bit for Docker log collection.
- Named volumes for Grafana, Prometheus, Loki and Fluent Bit state.

The HTTP server restricts access to configured local/client networks. HTTPS uses the generated certificate and key.

### Database

The initialization script creates the `active_network_equipment` table with:

- `id` as the primary key
- `device_type`
- `serial_number`
- `ip_address`
- `install_date`
- `maintenance_date`

It also inserts 30 mock network equipment records.

### Kubernetes

The `kubernetes/` directory contains:

- `deployment.yaml`: two Nginx replicas with resource requests/limits and a restricted security context.
- `service.yaml`: NodePort service for HTTP and HTTPS.
- `webserver-config.yaml`: Nginx configuration and web content ConfigMap.
- `ingress-tls.yaml`: TLS Ingress for `vps-interview.local`.
- `hpa.yaml`: CPU-based autoscaling from two to five replicas.

## Local Compose Usage

Create a local environment file:

```bash
cp project/.env.example project/.env
```

Edit `project/.env` and replace the example values. Generate a local certificate if required:

```bash
mkdir -p project/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
	-keyout project/ssl/nginx.key \
	-out project/ssl/nginx.crt \
	-subj "/CN=localhost"
```

Start the stack:

```bash
cd project
docker compose up -d
docker compose ps
```

Useful endpoints:

- HTTP: `http://192.168.0.30/`
- HTTPS: `https://192.168.0.30/` (self-signed certificate)
- Grafana: `http://192.168.0.30:3000`
- Prometheus: `http://192.168.0.30:9090`
- Alertmanager: `http://192.168.0.30:9093`
- Loki: `http://192.168.0.30:3100`

The endpoints above are for the Docker Compose stack deployed on the VirtualBox VPS and are available only while that VM is running. If the repository is also run locally on the laptop with Docker Compose, the local endpoints use `localhost` instead of `192.168.0.30`.

Do not use `docker compose down -v` on the VPS because it deletes persistent volumes and dashboards.

## Ansible Deployment

From the `ansible/` directory, provide credentials in the local controller environment:

```bash
export POSTGRES_USER=admin
export POSTGRES_DB=network_db
read -r -s POSTGRES_PASSWORD; export POSTGRES_PASSWORD
read -r -s GRAFANA_ADMIN_PASSWORD; export GRAFANA_ADMIN_PASSWORD
```

Run the playbook:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

The SSH private key is referenced by `inventory.ini` and must never be committed or copied into the repository.

## Kubernetes Deployment

Run these commands on the VirtualBox VPS. If Minikube is not already running, start it first:

```bash
minikube status
minikube start
```

If `minikube status` shows that the cluster is already running, continue with the next steps without running `minikube start` again.

Create the TLS Secret locally. The certificate and private key are intentionally excluded from Git:

```bash
kubectl create secret tls nginx-tls-secret \
	--cert=kubernetes/tls.crt \
	--key=kubernetes/tls.key \
	--dry-run=client -o yaml | kubectl apply -f -
```

Apply the manifests:

```bash
kubectl apply -f kubernetes/webserver-config.yaml
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
kubectl apply -f kubernetes/ingress-tls.yaml
kubectl apply -f kubernetes/hpa.yaml
```

For Minikube, enable the required addons first:

```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

See [`kubernetes/TLS-SECRET.md`](kubernetes/TLS-SECRET.md) for the complete TLS procedure.

## CI/CD Pipeline

The GitHub Actions workflow in `.github/workflows/ci-cd.yml` runs on pushes to `main` and pull requests.

### Validation

- YAML linting with `yamllint`.
- Ansible syntax validation and `ansible-lint`.
- Docker Compose configuration validation.
- Kubernetes schema validation with `kubeconform`.

### Security

- Trivy configuration scanning for HIGH and CRITICAL findings.
- Gitleaks scanning for accidentally committed secrets.

### Integration Test

The workflow generates a temporary TLS certificate, starts Docker Compose, checks HTTP and HTTPS responses, waits for PostgreSQL readiness, and shuts the stack down after the test.

### Deployment

The deployment job runs only after all checks pass on `main`. It uses a self-hosted Linux runner on the laptop because the VPS address is a private VirtualBox/LAN address and cannot be reached by a GitHub-hosted runner. The runner executes Ansible locally, and Ansible deploys to the VirtualBox VPS over SSH.

Configure these secrets in the GitHub `production` environment:

```text
VPS_HOST
VPS_USER
VPS_SSH_KEY
POSTGRES_USER
POSTGRES_PASSWORD
POSTGRES_DB
GRAFANA_ADMIN_PASSWORD
```

The self-hosted runner must be online and show `Idle` before deployment starts.

## Monitoring and Alerting

Prometheus collects host metrics from Node Exporter and web availability metrics from Blackbox Exporter. Alerts are defined for high CPU usage, low available memory and low disk space.

Fluent Bit reads Docker JSON logs and sends them to Loki. Grafana has Loki provisioned as a datasource, so container logs can be viewed from Grafana Explore.

## Challenges and Solutions

### Credentials were missing during Ansible deployment

The Compose file requires environment variables, while `.env` is intentionally excluded from Git. The playbook was updated to create the remote `.env` securely from controller environment variables with file mode `0600`.

### Fluent Bit entered a restart loop

The Fluent Bit Loki output did not support the `Line_Key` option in the selected version. It was replaced with the supported `Line_Format json` option.

### Blackbox probes returned HTTP 403 and HTTPS 400

The Docker network address changed dynamically and was not included in the Nginx allowlist. The allowlist was widened to the Docker private address range, and Blackbox was configured to send the `localhost` Host header and TLS server name.

### Docker network creation failed

A fixed Compose subnet overlapped with an existing Docker network on the VPS. The fixed subnet was removed so Docker can select an available private subnet.

### Grafana dashboards disappeared after redeployment

The containers were recreated with `--force-recreate` without persistent storage. Named volumes were added for Grafana and Prometheus data so dashboards and metric history survive container recreation.

### CI checks failed

The workflow initially contained an invalid Trivy action reference, unsupported Trivy flags, YAML formatting issues and a PostgreSQL startup race. These were resolved by using the Trivy CLI directly, correcting the workflow, normalizing YAML files and adding PostgreSQL readiness retries.

### GitHub runner could not reach the VPS

The VirtualBox VPS uses a private LAN address. The deployment job was moved to a self-hosted Linux runner on the laptop, which has network access to the VM.

### Ansible file synchronization required verification

When the playbook was run from the WSL-mounted repository under `/mnt/c`, file synchronization occasionally required verification after deployment. File sizes and service configuration were checked on the VM, and the playbook was rerun when a copied file did not match the local version.

### Container name conflicts during redeployment

Containers created manually during testing could conflict with Docker Compose container names. Existing containers and the Compose project state were checked before redeployment so the stack could be recreated cleanly.

### Missing Nginx `events` block

Nginx requires an `events` block even when it is empty. Adding the block fixed the initial Nginx restart loop.

## Assumptions and Limitations

- `192.168.0.30` is a local VirtualBox address and is not publicly reachable.
- The ISP connection uses Carrier-Grade NAT, so external SSH access depends on the temporary Cloudflare Tunnel described above.
- The SSL certificates are self-signed and suitable for testing, not production trust chains.
- A production deployment should use a trusted certificate such as Let's Encrypt.
- The Kubernetes TLS Secret must be created manually in the Minikube cluster on the VPS.
- The Kubernetes deployment serves a basic Nginx page and is separate from the complete Docker Compose stack.
- Kubernetes and application endpoints are unavailable when the VirtualBox VPS is powered off. GitHub repository history and workflow results remain available independently.
- Grafana and monitoring ports are currently published by Docker Compose and should be restricted with firewall rules in production.
- Strong, unique passwords should be used instead of local test values.

## Deliverables

- Ansible provisioning and deployment playbook.
- Docker Compose application and monitoring stack.
- Kubernetes Deployment, Service, Ingress and HPA manifests.
- CI/CD workflow with validation, security scanning and deployment.
- Database schema and 30 mock records.
- TLS Secret creation instructions.
- Architecture, assumptions, limitations and challenge report.
