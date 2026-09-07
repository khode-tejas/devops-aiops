# Cloud-Native Boutique Platform with AIOps

This repository contains a boutique e-commerce application and the infrastructure used to run it locally or on Amazon EKS. The implementation includes a React frontend, Node.js microservices, PostgreSQL, container builds, Terraform, Kubernetes manifests, Argo CD, Prometheus, Grafana, and an AWS Bedrock-based incident-analysis assistant.

## Architecture

```text
Browser
  |
  v
React frontend :3000
  |
  v
API gateway :3001
  |-- /api/auth     --> auth :3002
  |-- /api/products --> product-service :3003
  |-- /api/orders   --> orders :3005
  `-- /api/users    --> user-service :3006

order-service :3004 --> product-service + orders_db

auth, product-service, orders, order-service, and user-service
  |
  v
PostgreSQL :5432
  |-- auth_db
  |-- products_db
  |-- orders_db
  `-- users_db

Prometheus --> service /metrics endpoints --> Grafana

GitHub Actions --> Amazon ECR --> Git manifests --> Argo CD --> Amazon EKS
                                                               |
AWS Bedrock Agent --> Lambda action groups --> CloudWatch, Prometheus, and EKS
```

The gateway routes the application API to `auth`, `product-service`, `orders`, and `user-service`. The separate JavaScript `order-service` on port `3004` implements a smaller order flow that validates a product and writes to the `order_service.orders` table; it is built and deployed by the repository but is not registered as a gateway route in the current gateway source.

## Components

| Component | Implementation | Port | Responsibility |
|---|---|---:|---|
| Frontend | React 18, TypeScript, Vite, Material UI | 3000 | Catalog, cart, authentication, orders, and profile UI |
| Gateway | Node.js, Express, TypeScript | 3001 | Routes `/api/*` requests and exports Prometheus metrics |
| Auth | Node.js, Express, TypeScript | 3002 | Registration, login, logout, and current-user operations |
| Product service | Node.js, Express, TypeScript | 3003 | Product catalog and category queries |
| Order service | Node.js, Express, JavaScript | 3004 | Product validation and basic order creation |
| Orders | Node.js, Express, TypeScript | 3005 | Checkout, order history, and order status updates |
| User service | Node.js, Express, TypeScript | 3006 | Profiles and addresses |
| PostgreSQL | PostgreSQL 15 | 5432 | Service databases and seeded catalog data |
| Grafana | Grafana container / Helm chart | 3007 locally | Metrics dashboards |
| Prometheus | Prometheus container / Helm chart | 9090 | Metrics scraping and storage |
| AIOps assistant | Streamlit, AWS Bedrock Agent, AWS Lambda | 8501 | Incident queries across logs, metrics, and EKS health |

## Repository layout

```text
.
|-- .github/workflows/ci.yml             # Build, push, and manifest-update workflow
|-- gitops/
|   |-- argo-cd.yml                       # Argo CD Application
|   |-- kustomization.yml                 # Boutique workload entry point
|   `-- k8s/                              # Database, backend, frontend, and monitoring resources
|-- projects/
|   |-- Infrastructure/                   # Terraform root module
|   |   `-- modules/                      # VPC, EKS, ECR, Argo CD, and monitoring modules
|   |-- aiops-assistant/                  # Bedrock Agent, Lambda handlers, schemas, and Streamlit UI
|   `-- boutique-microservices/
|       |-- backend/services/             # Gateway and backend services
|       |-- database/                     # PostgreSQL initialization and seed data
|       |-- frontend/                     # React/Vite application
|       |-- grafana/                      # Local Grafana provisioning
|       |-- prometheus/                   # Local scrape configuration
|       `-- docker-compose.yml            # Local application stack
`-- docs/                                 # Supporting design and workflow notes
```

## Prerequisites

Local stack:

- Git
- Docker with Docker Compose v2
- Node.js 20 or later
- npm

AWS deployment:

- An AWS account and configured AWS CLI
- Terraform 1.5 or later
- `kubectl`
- Access to Amazon EKS, ECR, IAM, Lambda, CloudWatch, and Amazon Bedrock

The Terraform and AIOps scripts are configured for `us-east-1`.

## Run the application locally

Clone the repository and enter the application directory:

```bash
git clone https://github.com/khode-tejas/devops-aiops.git
cd devops-aiops/projects/boutique-microservices
```

Install the workspace dependencies and build the frontend and TypeScript services:

```bash
npm install
npm run build
```

The frontend build is required before Compose starts because the Compose service mounts `frontend/build` into an Nginx container.

Build and start the stack:

```bash
docker compose up -d --build
docker compose ps
```

| Endpoint | URL |
|---|---|
| Storefront | http://localhost:3000 |
| Gateway health | http://localhost:3001/health |
| Gateway metrics | http://localhost:3001/metrics |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3007 |

The local Grafana credentials defined in `docker-compose.yml` are `admin` / `admin`.

View logs or stop the stack:

```bash
docker compose logs -f
docker compose down
```

PostgreSQL, Prometheus, and Grafana use named Docker volumes, so `docker compose down` preserves their data.

## Local implementation details

- Vite builds the frontend into `frontend/build`.
- Nginx serves the static frontend and proxies `/api/` to the gateway container.
- The gateway removes each `/api/<service>` prefix before forwarding the request.
- PostgreSQL initialization creates `auth_db`, `products_db`, `orders_db`, and `users_db`, then applies the schema and seed SQL in `database/init`.
- Prometheus scrapes `/metrics` from all six backend containers every 15 seconds.
- The services expose `/health` and `/metrics` through the shared metrics setup.

## Provision AWS infrastructure

Confirm the active AWS identity:

```bash
aws sts get-caller-identity
```

Initialize and review the Terraform configuration:

```bash
cd projects/Infrastructure
terraform init
terraform validate
terraform plan
```

Apply the reviewed plan:

```bash
terraform apply
```

The checked-in `terraform.tfvars` configures:

- Region: `us-east-1`
- VPC: `10.1.0.0/16`
- Three public subnets in `us-east-1a`, `us-east-1b`, and `us-east-1c`
- EKS cluster: `eks-cluster`, Kubernetes `1.34`
- Managed node group: one desired `m7i-flex.large` on-demand node, scaling from one to two nodes
- Seven ECR repositories, one for each application image
- Amazon EBS CSI driver with an IAM role for service accounts
- Argo CD Helm chart `6.7.0`
- `kube-prometheus-stack` Helm chart `56.21.0`

Configure `kubectl` after Terraform completes:

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-cluster
kubectl get nodes
```

## Prepare the Kubernetes configuration

The manifests under `gitops/` require repository-specific values before the first deployment.

1. Replace every `<AWS_ACCOUNT_ID>` in `gitops/k8s` with the AWS account that owns the ECR repositories.
2. Change `spec.source.repoURL` in `gitops/argo-cd.yml` to the Git repository Argo CD should track.
3. Change the `order-service` Service `port` and `targetPort` in `gitops/k8s/backend/order-service.yml` from `3002` to `3004`, matching the Deployment and container.
4. Replace the plain-text database values in `gitops/secrets.yml` before using the manifests outside a local or disposable environment.

Verify that no image placeholder remains:

```bash
rg '<AWS_ACCOUNT_ID>' gitops/k8s
```

The command should produce no output.

## CI/CD and GitOps flow

`.github/workflows/ci.yml` runs manually and on pushes to `main`.

1. A matrix job builds `frontend`, `gateway`, `auth`, `product-service`, `order-service`, `orders`, and `user-service`.
2. Each image is tagged with the commit SHA and pushed to its ECR repository.
3. A dependent job updates image tags in `gitops/k8s` and pushes the manifest commit.
4. Argo CD reads `gitops/` and applies its Kustomize configuration to the `boutique` namespace.

Configure these GitHub Actions secrets:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_ACCOUNT_ID
AWS_REGION
```

Set `AWS_REGION` to `us-east-1`. The current workflow uses access keys; its manifest replacement only matches image addresses that already contain the configured account ID. The initial `<AWS_ACCOUNT_ID>` replacement must therefore be committed before the first successful manifest-update run.

Register the Argo CD application:

```bash
kubectl apply -f gitops/argo-cd.yml
kubectl get applications -n argocd
```

The checked-in Argo CD application uses manual synchronization because `syncPolicy.automated` is commented out.

Access Argo CD locally:

```bash
kubectl port-forward deployment/argocd-server -n argocd 8081:8080
```

Open http://localhost:8081. The username is `admin`; retrieve the initial password with:

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 --decode
echo
```

## Deploy the workloads

Apply the Kustomize resources and inspect the namespace:

```bash
kubectl apply -k gitops/
kubectl get pods -n boutique
kubectl get services -n boutique
```

`gitops/kustomization.yml` generates the `boutique-db-dump` ConfigMap but does not include the database restore Job. Run the Job separately after PostgreSQL is ready:

```bash
kubectl wait --for=condition=ready pod \
  -l app=postgres \
  -n boutique \
  --timeout=180s
kubectl apply -f gitops/k8s/database/restore-job.yml
kubectl logs -n boutique job/boutique-db-restore
```

Access the frontend Service:

```bash
kubectl port-forward svc/frontend -n boutique 3000:3000
```

The Kubernetes frontend image is served with `serve`, while the browser client defaults to `/api`. Unlike the local Nginx container, that image does not proxy `/api` to the gateway. A functional cluster-facing UI therefore requires either an Ingress/reverse proxy or a frontend image built with `VITE_API_URL` set to a reachable gateway URL.

## Monitoring

The Terraform configuration installs Prometheus, Alertmanager, and Grafana through `kube-prometheus-stack`. The GitOps configuration adds a `ServiceMonitor` that selects the gateway Service and a Grafana dashboard ConfigMap.

Access Prometheus:

```bash
kubectl port-forward \
  -n monitoring \
  svc/kube-prometheus-stack-prometheus \
  9090:9090
```

Example queries:

```promql
up{namespace="boutique"}
```

```promql
rate(http_requests_total{namespace="boutique"}[1m])
```

Access Grafana:

```bash
kubectl port-forward \
  -n monitoring \
  svc/kube-prometheus-stack-grafana \
  3007:80
```

Open http://localhost:3007 with username `admin`. Retrieve the generated password with:

```bash
kubectl get secret kube-prometheus-stack-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 --decode
echo
```

## AIOps assistant

`projects/aiops-assistant` implements a Streamlit chat UI for an Amazon Bedrock Agent named `aiops-assistant`. The deployment script configures the agent to use `qwen.qwen3-32b-v1:0` and three Lambda action groups:

| Action group | Lambda function | Data source |
|---|---|---|
| `fetch_logs` | `aiops-fetch-logs` | CloudWatch Logs |
| `fetch_metrics` | `aiops-fetch-metrics` | Prometheus query API |
| `fetch_service_health` | `aiops-fetch-health` | EKS API and Prometheus |

From `projects/aiops-assistant`:

```bash
./setup-iam.sh
```

Create the three Python 3.12 Lambda functions with the names in the table, using the handlers in `lambda/`, and assign the generated `aiops-lambda-role` execution role. Before uploading the metrics and health handlers, replace `<YOUR_PROMETHEUS_ELB_URL>` in:

```text
lambda/fetch_metrics/lambda_function.py
lambda/fetch_health/lambda_function.py
```

The handlers require a network-reachable Prometheus endpoint. The repository's Helm configuration creates Prometheus as `ClusterIP`; expose it through an appropriate secured endpoint before connecting the Lambda functions.

Create and prepare the Bedrock Agent:

```bash
./deploy.sh
```

Configure and run the UI:

```bash
cp .env.example .env
pip install -r requirements.txt
streamlit run app.py
```

Set these values in `.env` before starting Streamlit:

```dotenv
AWS_REGION=us-east-1
BEDROCK_AGENT_ID=your_agent_id
BEDROCK_AGENT_ALIAS_ID=TSTALIASID
```

AWS access-key variables in `.env` are optional. If they are empty, boto3 uses its normal credential chain, including AWS CLI profiles, SSO, environment credentials, or an IAM role. The UI is available at http://localhost:8501.

## Current configuration constraints

- `docker-compose.yml` sets `USER_SERVICE_URL`, but the gateway reads `USERS_SERVICE_URL`. Rename the Compose variable to route `/api/users` to `user-service:3006`.
- The gateway reads `ORDERS_SERVICE_URL`; the separate `ORDER_SERVICE_URL` value in Compose is not used by the current gateway source.
- The Kubernetes `order-service` Service ports must be changed to `3004` as described above.
- The Kubernetes frontend needs a cluster-accessible API route because its `serve` process has no `/api` proxy.
- The database restore Job is not part of `gitops/kustomization.yml` and must be applied separately.
- The GitOps Secret contains demonstration database credentials in plain text.
- The repository's package test scripts are placeholders or incomplete; `npm run build` is the available application-wide verification command.

## Destroy the AWS environment

The EKS cluster, worker nodes, public networking, ECR repositories, and Helm releases incur AWS charges while provisioned. Remove them from the Terraform directory when they are no longer needed:

```bash
cd projects/Infrastructure
terraform plan -destroy
terraform destroy
```

The ECR module uses `force_delete = true`, so destroying the stack also deletes images stored in the managed repositories.

## Security

The checked-in configuration is not production hardened. Before deploying a real workload:

- Replace the database credentials and manage secrets outside Git.
- Replace long-lived GitHub Actions access keys with AWS IAM federation through OpenID Connect.
- Restrict the public EKS API and move workloads that do not need public addresses into private subnets.
- Secure any Prometheus endpoint exposed for Lambda access.
- Add resource requests and limits, application probes, network policies, and complete automated tests.

## License

No license file is included in this repository.
