# PRGuardian

**An AI-powered GitHub pull-request reviewer: four LLM agents review every PR
in parallel, post inline comments, and learn from what gets merged.**

PRGuardian is a set of event-driven microservices deployed on AWS EKS. When a
developer opens a pull request on a repository where the PRGuardian GitHub App
is installed, the diff is fetched, analysed by four specialised agents
(static analysis, security, style, architecture) orchestrated with LangGraph,
and the findings are posted back to the PR as review comments. Merged PRs
feed a learner that stores patterns for future reviews, and a weekly job
scores the quality of the generated comments with RAGAS.

---

## Contents

1. [How it works](#1-how-it-works)
2. [Repository layout](#2-repository-layout)
3. [Requirements](#3-requirements)
4. [Setup, step by step](#4-setup-step-by-step)
5. [Operating the system](#5-operating-the-system)
6. [Troubleshooting](#6-troubleshooting)
7. [Cost and teardown](#7-cost-and-teardown)
8. [Security notes](#8-security-notes)

---

## 1. How it works

```
GitHub (pull_request event)
        │  HTTPS webhook
        ▼
AWS Application Load Balancer  (Kubernetes Ingress)
        │
        ▼
gateway        :8000   verifies the GitHub HMAC-SHA256 signature, forwards the event
        │
        ▼
webhook        :8001   records the PR in PostgreSQL, queues an analysis task
        │
        ▼
webhook-worker         Celery worker on Redis: calls the orchestrator
        │
        ▼
orchestrator   :8002   fetches the diff with the GitHub App token and runs the LangGraph graph
        │
        ├── static-analysis agent ─┐
        ├── security agent         ├─ run in parallel (gpt-4o-mini via OpenAI, traced in Langfuse)
        ├── style agent            │
        └── architecture agent ────┘
        │
        ▼
reviewer       :8003   posts a summary and inline review comments on the PR
        │
        ▼  (when the PR is merged)
learner        :8004   + learner-worker: stores accepted/ignored patterns in PostgreSQL

scripts/evaluate.py    weekly Kubernetes job: RAGAS faithfulness and relevancy over the last 50 findings
```

| Component | Technology |
|---|---|
| Services | Python 3.11, FastAPI, httpx, Pydantic |
| Agents | LangGraph, LangChain, OpenAI `gpt-4o-mini` |
| Queue | Celery on Redis (Amazon ElastiCache) |
| Database | PostgreSQL 15 on Amazon RDS, Alembic migrations |
| Tracing | Langfuse Cloud |
| Metrics | Prometheus and Grafana (Helm charts), dashboard in `monitoring/` |
| Evaluation | RAGAS, run as a Kubernetes Job from GitHub Actions |
| Platform | Amazon EKS (2 × t3.medium), ECR, S3, Terraform |
| CI/CD | GitHub Actions, one pipeline per service: test → build and push → `kubectl set image` |

---

## 2. Repository layout

```
.
├── services/
│   ├── gateway/        HMAC verification of GitHub webhooks        (port 8000)
│   ├── webhook/        event parsing, PR records, Celery producer  (port 8001) + worker.py
│   ├── orchestrator/   GitHub diff fetch and the LangGraph agents  (port 8002)  graph.py
│   ├── reviewer/       posts review comments to GitHub            (port 8003)
│   └── learner/        learns from merged PRs                     (port 8004) + worker.py
│       each with: main.py, models.py (settings), requirements.txt, Dockerfile, deploy.txt
├── db/                 Alembic configuration and migrations
├── scripts/            evaluate.py and its Dockerfile (weekly RAGAS job)
├── infra/
│   ├── terraform/      VPC, EKS, RDS, ElastiCache, S3, six ECR repositories, IAM for the LB controller
│   └── k8s/            Deployments, Services, HPA, Ingress, ConfigMap, migration and evaluate Jobs,
│                       secret.yaml.example (template — copy to secret.yaml, which is git-ignored)
├── monitoring/         prometheus.yml scrape config and the Grafana dashboard JSON
└── .github/workflows/  gateway.yml, webhook.yml, orchestrator.yml, reviewer.yml, learner.yml
                        (all call service-ci.yml), plus evaluate.yml (Mondays 09:00 UTC)
```

---

## 3. Requirements

### Your machine

The application runs entirely on AWS; your machine only runs command-line
tools. Windows 10/11, macOS or Linux, 4 GB RAM, 5 GB free disk and a stable
internet connection are enough. Commands below are shown for a POSIX shell;
on Windows CMD replace the `\` line continuations with `^`.

| Tool | Install from | Check |
|---|---|---|
| AWS CLI v2 | https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html | `aws --version` |
| Terraform ≥ 1.5 | https://developer.hashicorp.com/terraform/install | `terraform --version` |
| kubectl | https://kubernetes.io/docs/tasks/tools/ | `kubectl version --client` |
| Helm 3 | https://helm.sh/docs/intro/install/ | `helm version` |
| Git | https://git-scm.com/downloads | `git --version` |

Nothing is built locally: GitHub Actions builds the Docker images.

### Accounts

| Account | Needed for | Notes |
|---|---|---|
| AWS | EKS, RDS, ElastiCache, ECR, S3 | Billed; about USD 180 per month while running (see §7) |
| GitHub | the repository, the GitHub App, Actions | Free |
| OpenAI | `gpt-4o-mini` for the four agents | Add about USD 10 of credit and set a monthly usage limit |
| Langfuse Cloud | tracing of every LLM call | Free tier is enough |

### Values you will collect along the way

Keep a private notes file with these; later steps refer to them by name.

`AWS_ACCOUNT_ID`, `AWS_ROLE_ARN`, `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`
(the `.pem` file), `GITHUB_WEBHOOK_SECRET`, `OPENAI_API_KEY`,
`LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, the Terraform outputs
(`rds_endpoint`, `redis_endpoint`), and the `db_password` you choose.

---

## 4. Setup, step by step

Do the phases in order. Phases 1–7 are one-time preparation; 8–12 deploy the
system; 13–16 add monitoring, tracing and the final wiring.

### Phase 1 — AWS account and CLI access

1. Create an AWS account and sign in to the console.
2. **IAM → Users → Create user** `pr-guardian-deployer`, attach
   `AdministratorAccess`, then create an **access key** of type *Command Line
   Interface* and save both values.
3. Configure the CLI and confirm it works:

   ```sh
   aws configure            # key, secret, region us-east-1, output json
   aws sts get-caller-identity --query Account --output text   # -> AWS_ACCOUNT_ID
   ```

4. **IAM → Identity providers → Add provider**: type *OpenID Connect*, URL
   `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`.
   This lets GitHub Actions deploy without long-lived AWS keys.
5. **IAM → Roles → Create role**: trusted entity *Web identity*, provider
   `token.actions.githubusercontent.com`, audience `sts.amazonaws.com`, add
   condition `StringLike` on `token.actions.githubusercontent.com:sub` =
   `repo:YOUR_GITHUB_USERNAME/pr-guardian:*`, attach `AdministratorAccess`,
   name it **`github-actions-pr-guardian`**. Save its ARN as `AWS_ROLE_ARN`.
   The Terraform code grants this exact role name access to the cluster.

### Phase 2 — GitHub repository

1. Create a private repository named **`pr-guardian`** (no README).
2. Create a classic personal access token with `repo` and `workflow` scopes
   (name it `pr-guardian-push`) to use as the password when pushing.
3. Push this code:

   ```sh
   git init && git add . && git commit -m "initial: PRGuardian"
   git branch -M main
   git remote add origin https://github.com/YOUR_GITHUB_USERNAME/pr-guardian.git
   git push -u origin main
   ```

   The five service pipelines start immediately. Their **deploy** job fails
   with "deployment not found" until Phase 12 — that is expected.

### Phase 3 — GitHub App

The App is the identity PRGuardian uses to read diffs and post comments.

1. **Settings → Developer settings → GitHub Apps → New GitHub App**.
2. Name **`pr-guardian-bot`** (must be globally unique; add a suffix if
   taken). Homepage: your repository URL.
3. Webhook: *Active*, URL `https://placeholder.example` for now, and a long
   random **webhook secret** — save it as `GITHUB_WEBHOOK_SECRET`.
4. Repository permissions: **Contents: Read-only**, **Pull requests: Read and
   write**. Subscribe to events: **Pull request**. Installable on *Any
   account*.
5. Create the App. Save the **App ID** (`GITHUB_APP_ID`). Under *Private
   keys*, **Generate a private key**; the downloaded `.pem` file is
   `GITHUB_APP_PRIVATE_KEY`.
6. **Install App** on the repositories you want reviewed.

### Phase 4 — Langfuse

Create a project named `pr-guardian` at https://cloud.langfuse.com,
then **Settings → API Keys → Create new API key**. Save the public
(`pk-lf-…`) and secret (`sk-lf-…`) keys. Note whether your project is in the
US or EU region; the host goes into the ConfigMap in Phase 12.

### Phase 5 — OpenAI

Create an API key named `pr-guardian` at https://platform.openai.com and
save it as `OPENAI_API_KEY`.

### Phase 6 — Provision AWS with Terraform (15–25 minutes)

```sh
cd infra/terraform
terraform init
terraform plan  -var="cluster_name=pr-guardian" -var="db_password=CHOOSE_A_STRONG_PASSWORD" -var="environment=production"
terraform apply -var="cluster_name=pr-guardian" -var="db_password=CHOOSE_A_STRONG_PASSWORD" -var="environment=production"
terraform output
```

This creates a VPC, an EKS cluster with two t3.medium nodes, an RDS
PostgreSQL instance (`pr-guardian-postgres`, database `prguardian`, user
`dbadmin`), an ElastiCache Redis cluster (`pr-guardian-redis`), the S3 bucket
`pr-guardian-reports` and six ECR repositories (`gateway`, `webhook`,
`orchestrator`, `reviewer`, `learner`, `evaluate`). Save every line of
`terraform output`; you need `rds_endpoint` and `redis_endpoint` in Phase 12.

If the S3 bucket name is already taken globally, change `bucket` in
`infra/terraform/main.tf` before applying.

### Phase 7 — GitHub Actions secrets

**Repository → Settings → Secrets and variables → Actions**, add exactly three:

| Secret | Value |
|---|---|
| `AWS_ACCOUNT_ID` | the 12-digit account id |
| `AWS_ROLE_ARN` | `arn:aws:iam::<account>:role/github-actions-pr-guardian` |
| `EKS_CLUSTER_NAME` | `pr-guardian` |

Nothing else goes here. Provider keys and GitHub App credentials live only in
the Kubernetes secret (Phase 12); GitHub also rejects Actions secrets whose
names start with `GITHUB_`.

### Phase 8 — Connect kubectl to the cluster

```sh
aws eks update-kubeconfig --name pr-guardian --region us-east-1
kubectl get nodes        # two nodes, STATUS Ready
```

### Phase 9 — Fill in the runtime configuration

1. Edit `infra/k8s/configmap.yaml`: set `REDIS_URL` to
   `redis://<redis_endpoint>/0` from the Terraform output, and
   `LANGFUSE_HOST` to `https://us.cloud.langfuse.com` or
   `https://cloud.langfuse.com` (EU) to match your Langfuse project.
2. Copy the secret template and fill in **every** value:

   ```sh
   cp infra/k8s/secret.yaml.example infra/k8s/secret.yaml
   ```

   `DATABASE_URL` is
   `postgresql+asyncpg://dbadmin:<db_password>@<rds_endpoint>/prguardian`
   (the Terraform output already includes `:5432`). Paste the whole `.pem`
   file, indented, under `GITHUB_APP_PRIVATE_KEY: |`.

   `secret.yaml` is git-ignored. Never commit it.

### Phase 10 — Apply the Kubernetes manifests

```sh
cd infra/k8s
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f migration-job.yaml          # runs the Alembic migrations against RDS
kubectl wait --for=condition=complete job/db-migrate --timeout=180s
kubectl apply -f gateway.yaml -f webhook.yaml -f orchestrator.yaml -f reviewer.yaml -f learner.yaml
kubectl apply -f webhook-worker.yaml -f learner-worker.yaml
kubectl apply -f hpa.yaml                    # orchestrator scales 2–6 replicas on CPU
```

Pods stay in `ImagePullBackOff` until Phase 11 pushes images — that is fine.

### Phase 11 — First real deployment

Trigger all five pipelines by bumping the marker file in each service:

```sh
for s in gateway webhook orchestrator reviewer learner; do echo 1 > services/$s/deploy.txt; done
git add services && git commit -m "deploy: initial images" && git push origin main
```

In **Actions**, five workflows run; each has three jobs: **test** (pytest,
Python 3.11), **build-and-push** (image tagged with the commit SHA pushed to
ECR) and **deploy** (`kubectl set image`). All should be green. Then:

```sh
kubectl get pods     # gateway, webhook, orchestrator ×2, reviewer, learner, webhook-worker, learner-worker: Running
```

To redeploy one service later, change any file under `services/<name>/` or
increment its `deploy.txt`; only that service's pipeline runs.

### Phase 12 — Expose the gateway

Install the AWS Load Balancer Controller once, then apply the Ingress:

```sh
helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=pr-guardian --set serviceAccount.create=true
kubectl get pods -n kube-system | grep aws-load-balancer    # wait until Running
kubectl apply -f infra/k8s/ingress.yaml
kubectl get ingress gateway-ingress     # ADDRESS appears after 2–3 minutes
```

Terraform has already attached the `pr-guardian-lbc-policy` IAM policy that
the controller needs to the node role.

### Phase 13 — Point the GitHub App at the deployment

In the GitHub App settings set the **Webhook URL** to

```
http://<ingress ADDRESS>/webhook/github
```

and save. The webhook secret stays as typed in Phase 3.

### Phase 14 — Prometheus and Grafana

```sh
# storage driver for Prometheus volumes
aws eks create-addon --cluster-name pr-guardian --addon-name aws-ebs-csi-driver --region us-east-1
# attach arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy to the node-group role first

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update

helm install prometheus prometheus-community/prometheus -n monitoring --create-namespace \
  --set server.persistentVolume.size=8Gi --set server.persistentVolume.storageClass=gp2 \
  --set alertmanager.persistence.storageClass=gp2
kubectl create configmap prometheus-config --from-file=prometheus.yml=monitoring/prometheus.yml \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -

helm install grafana grafana/grafana -n monitoring \
  --set persistence.enabled=true --set persistence.size=5Gi --set persistence.storageClassName=gp2 \
  --set service.type=LoadBalancer \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-type=external" \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type=ip" \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-scheme=internet-facing"

kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 -d   # admin password
kubectl get svc grafana -n monitoring                                                       # EXTERNAL-IP
```

In Grafana add the data source `http://prometheus-server.monitoring.svc.cluster.local`
and import `monitoring/grafana-dashboard.json` (dashboard "PRGuardian").
Prometheus itself is reachable with
`kubectl port-forward -n monitoring svc/prometheus-server 9090:80`.

### Phase 15 — End-to-end test

1. Open a pull request on a repository where the App is installed.
2. `kubectl logs deploy/gateway` shows the verified webhook,
   `kubectl logs deploy/orchestrator` the four agents running.
3. Within a minute the PR receives a **"PRGuardian review"** summary comment
   and inline comments on the changed lines.
4. The run appears in Langfuse with prompts, tokens and cost per agent.
5. Merge the PR; `kubectl logs deploy/learner-worker` shows the patterns
   being stored.

---

## 5. Operating the system

| Task | How |
|---|---|
| Health of all pods | `kubectl get pods` and `kubectl get hpa` |
| Logs of one service | `kubectl logs deploy/orchestrator -f` |
| Redeploy a service | push a change under `services/<name>/` (or bump its `deploy.txt`) |
| Rotate a secret | edit `infra/k8s/secret.yaml`, `kubectl apply -f`, then `kubectl rollout restart deploy` |
| Weekly quality check | the **Evaluate** workflow runs Mondays 09:00 UTC (or *Run workflow* manually); it builds `scripts/Dockerfile`, runs `evaluate.py` as the Kubernetes Job `evaluate`, scores the last 50 findings with RAGAS, and fails when mean faithfulness is below 0.7 |
| Metrics | Grafana dashboard "PRGuardian"; raw metrics at each service's `/metrics` |
| Traces and LLM cost | Langfuse project `pr-guardian` |

---

## 6. Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Deploy job fails with `deployment not found` | Phase 10 has not been applied yet; apply the manifests and re-run the pipeline |
| Pods in `ImagePullBackOff` | No image pushed yet (run Phase 11), or the pipeline failed — check Actions |
| Pods in `CrashLoopBackOff` | Usually a bad `DATABASE_URL` or `REDIS_URL`; `kubectl logs <pod>` shows the connection error |
| `db-migrate` job fails | RDS not reachable or wrong password; check the security group and `DATABASE_URL` |
| GitHub shows webhook delivery `401` | `GITHUB_WEBHOOK_SECRET` in the cluster differs from the one in the App; `kubectl logs deploy/gateway` |
| Webhook delivery times out | Ingress has no ADDRESS yet, or the LB controller is not running (`kubectl get pods -n kube-system`) |
| Comments never appear | App lacks *Pull requests: Read and write*, or the App is not installed on that repository; check `kubectl logs deploy/reviewer` |
| Prometheus pods `Pending` | EBS CSI driver add-on missing or its IAM policy not attached to the node role |
| Evaluate job fails | Fewer than 50 findings in the database yet, or OpenAI quota exhausted |

---

## 7. Cost and teardown

Running 24/7 costs roughly USD 180 per month: EKS control plane 72, two
t3.medium nodes 60, RDS db.t3.micro 15, ElastiCache cache.t3.micro 12, load
balancers 18, ECR and S3 about 2. OpenAI usage with `gpt-4o-mini` is about
USD 0.15 per 100 PRs; Langfuse is free within its limits.

To stop all charges:

```sh
kubectl delete ingress --all
helm uninstall grafana prometheus -n monitoring
helm uninstall aws-load-balancer-controller -n kube-system
cd infra/terraform
terraform destroy -var="cluster_name=pr-guardian" -var="db_password=<the same password>" -var="environment=production"
```

Afterwards check for leftovers the controller created outside Terraform:
load balancers (`aws elbv2 describe-load-balancers`), unattached EBS volumes
(`aws ec2 describe-volumes --filters Name=status,Values=available`) and the
`/aws/eks/pr-guardian` log groups.

---

## 8. Security notes

* `infra/k8s/secret.yaml` holds the GitHub App private key, the OpenAI and
  Langfuse keys and the database password. Only the `.example` template is
  in the repository; the real file is git-ignored and must stay that way.
* The IAM user and role created in Phase 1 use `AdministratorAccess` for
  simplicity. For anything beyond a personal deployment, scope them down to
  EKS, ECR, RDS, ElastiCache, S3 and the VPC resources Terraform manages.
* The ingress is plain HTTP. Put an ACM certificate on the load balancer
  (`alb.ingress.kubernetes.io/certificate-arn` on the Ingress) and switch
  the GitHub App webhook URL to `https://` before reviewing anything
  confidential.
* Limit the OpenAI key with a monthly usage cap; the orchestrator calls the
  model four times per PR.
