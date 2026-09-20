# k3s-HermesAgent

Kubernetes manifests to deploy [Hermes Agent](https://github.com/NousResearch/hermes-agent) on a personal K3s cluster using Telegram in the default polling mode.

## Overview

This project provides the Kubernetes resources needed to run a single Hermes Agent gateway pod in a non-cloud K3s environment. The deployment uses the official Docker image `nousresearch/hermes-agent:v2026.7.1`, stores all mutable Hermes state in `/opt/data`, and keeps Telegram connectivity outbound-only by relying on long polling rather than webhooks.

The seeded Hermes configuration is set up for:

- Local llama.cpp server on `https://ai.styxut.net/v1` (Qwen3.8-27B GGUF)
- OpenCode Go models via `OPENCODE_GO_API_KEY`
- Self-hosted Mem0 memory backed by the cluster PostgreSQL/pgvector service
- OpenBrain OB1 via the in-cluster MCP service
- Optional OpenAI Codex / ChatGPT OAuth after a one-time interactive login
- Internal Hermes dashboard on port `9119` with required basic auth

The included Service is internal-only (`ClusterIP`) and exists to expose Hermes' optional API/health port on `8642` inside the cluster. Telegram itself does not require an inbound Kubernetes Service for this deployment model.

## Manifests

| File | Kind | Description |
|------|------|-------------|
| `hermes-agent-config.yaml` | `ConfigMap` | Provides the initial seed content for `/opt/data/config.yaml`: a llama.cpp-first model configuration, model aliases for OpenCode Go and OpenAI OAuth, plus unattended gateway loop guardrails. |
| `hermes-agent-deployment.yaml` | `Deployment` | Single-replica deployment of `nousresearch/hermes-agent:v2026.7.1`. Uses a `Recreate` strategy. Starts Hermes with `gateway run`, mounts `synology-k8s-pv02` at `/opt/data`, enables the internal API server on port `8642`, and injects secrets from `hermes-agent-secret`. |
| `hermes-agent-svc.yaml` | `Service` | Internal `ClusterIP` Service exposing Hermes API/health on port `8642`. Not exposed to the LAN or internet by default. |
| `kustomization.yaml` | `Kustomization` | Kustomize overlay that bundles the Deployment and Service, generates the `hermes-agent-secret` from files in `secrets/`, and disables the name-suffix hash. |
| `mem0-db-init-job.yaml` | `Job` | Creates the `mem0_app` database in the existing `postgres-service` PostgreSQL server. |
| `mem0-deployment.yaml` | `Deployment` | Runs the self-hosted Mem0 REST API using the existing pgvector database and Ollama endpoint configuration. |
| `mem0-svc.yaml` | `Service` | Internal `ClusterIP` endpoint used by Hermes at `http://mem0:8000`. |

## Prerequisites

- A K3s (or any Kubernetes) cluster
- `kubectl` configured for the target cluster
- (Optional) `kustomize` or `kubectl` v1.14+
- A `synology-k8s-pv02` PersistentVolumeClaim available in the cluster (or adjust the volume claim name)
- A Telegram bot token created with `@BotFather`
- Your Telegram numeric user ID
- At least one model provider API key compatible with your Hermes configuration
- A long Mem0 API key and JWT secret
- A reachable llama.cpp server on `https://ai.styxut.net` for the Hermes model
- A reachable Ollama instance on `http://192.168.0.13:11434` for Mem0's LLM/embedder (see Mem0 notes)
- The existing `postgres-service` and `postgres-password` Secret from `../k3s-Postgres`

## Usage

### 1. Prepare secrets

Create the secret files in the `secrets/` directory:

```bash
echo -n '123456789:ABCdefGHIjklMNOpqrSTUvwxYZ' > secrets/telegram_bot_token.txt
echo -n '123456789' > secrets/telegram_allowed_users.txt
echo -n 'sk-opencode-...' > secrets/opencode_go_api_key.txt
openssl rand -hex 32 > secrets/api_server_key.txt
echo -n 'hermes-admin' > secrets/dashboard_username.txt
echo -n 'change-me-now' > secrets/dashboard_password.txt
openssl rand -hex 32 > secrets/dashboard_auth_secret.txt
openssl rand -hex 32 > secrets/mem0_api_key.txt
openssl rand -base64 48 > secrets/mem0_jwt_secret.txt
```

`TELEGRAM_ALLOWED_USERS` accepts a comma-separated list if you want to allow multiple Telegram users.

Set a real dashboard username and password before deploying. The dashboard is enabled by default in this manifest set, but it stays internal-only unless you deliberately expose it beyond the cluster.

> The `secrets/` directory is gitignored and will never be committed.

### 2. Review deployment settings

Edit `hermes-agent-deployment.yaml` as needed for your environment:

- Change the PVC name if you do not use `synology-k8s-pv02`
- Change the `subPath` if you want Hermes data stored elsewhere on the shared volume
- Adjust CPU and memory requests/limits
- Add `HERMES_UID` / `HERMES_GID` (or `PUID` / `PGID`) if your storage requires a specific runtime UID/GID

Edit `hermes-agent-config.yaml` if you want to change:

- The default llama.cpp model (`/opt/models/Qwen3.8-27B-Q4_K_M.gguf`)
- The llama.cpp base URL (`https://ai.styxut.net/v1`)
- The OpenCode Go alias model (`deepseek-v4-flash`)
- The OpenAI OAuth alias model (`gpt-5.4`)

The seed config is copied into the PVC only if `/opt/data/config.yaml` does not already exist. That means:

- Your later Hermes-driven config changes remain writable and persistent
- Editing `hermes-agent-config.yaml` after first boot will not overwrite an existing live config on the PVC
- If you want to re-seed from the manifest, remove or rename the existing `config.yaml` in the PVC path first

### 3. Deploy

```bash
kubectl apply -k .
```

The Mem0 server image's bundled OpenAI-compatible provider is pointed at the existing
Ollama endpoint. Mem0 uses `smollm2:1.7b` for extraction and `qwen3-embedding` for
embeddings; the Hermes model itself continues using its existing local configuration.
The upstream Mem0 API image is currently ARM64-only, so this deployment uses the
AMD64 image built from the official Mem0 server source and loaded on `k3s-worker-03`.

Apply `../k3s-Postgres` first if PostgreSQL is not already installed. The Mem0 init Job
waits for `postgres-service` and creates only the `mem0_app` database; memory vectors
remain in the existing pgvector-enabled `postgres` database.

This will:
1. Generate the `hermes-agent-secret` Kubernetes Secret from the secret files.
2. Create the `mem0_app` database in the existing PostgreSQL service.
3. Create the Mem0 API and internal `mem0` Service.
4. Create the `hermes-agent` Deployment with one replica.
5. Create the internal `hermes-agent` Service on ports `8642` and `9119`.

### 4. Verify

```bash
kubectl get pods -l app=hermes-agent
kubectl get svc hermes-agent
kubectl logs -f deploy/hermes-agent
```

Once the pod is running, send your bot a Telegram message to confirm the gateway is connected.

To reach the dashboard from your workstation without exposing it on the LAN:

```bash
kubectl port-forward svc/hermes-agent 9119:9119
```

Then open:

```text
http://127.0.0.1:9119
```

Sign in with the dashboard username and password from your Kubernetes secret files.

### 5. Bootstrap OpenAI OAuth (optional)

Hermes documents OpenAI Codex / ChatGPT access as an interactive device-code login rather than a plain Kubernetes secret. That login state persists on the PVC under `/opt/data/auth.json`.

Run this once if you want the `openai-oauth` alias to work:

```bash
kubectl exec -it deploy/hermes-agent -- hermes auth add codex-oauth
```

Complete the device-code login in your browser. The resulting credentials will persist across pod restarts because `/opt/data` is on the PVC.

After that, you can switch Hermes to OpenAI OAuth with:

```text
/model openai-oauth
```

### 6. Remove

```bash
kubectl delete -k .
```

## Configuration Notes

### Telegram mode

This deployment is intended for a personal, always-on cluster and uses Telegram polling mode by default.

- Do not set `TELEGRAM_WEBHOOK_URL`
- Do not expose a webhook port for Telegram
- No public ingress is required for Telegram messaging to work

If you use Hermes in Telegram groups, remember:

- Telegram privacy mode is enabled by default for bots
- If the bot should see normal group messages, disable privacy mode in `@BotFather` or make the bot a group admin
- After changing privacy mode, remove and re-add the bot to each group

### API and health port

The deployment enables Hermes' API/health listener on port `8642` so Kubernetes can probe it and so you can optionally consume it from inside the cluster.

If you do not want the API server enabled, remove these settings from `hermes-agent-deployment.yaml`:

- `API_SERVER_ENABLED=true`
- `API_SERVER_HOST=0.0.0.0`
- The `API_SERVER_KEY` secret
- The container port, probes, and Service manifest

### Dashboard

The deployment enables Hermes' built-in dashboard on port `9119` with basic authentication.

Enabled env vars:

- `HERMES_DASHBOARD=1`
- `HERMES_DASHBOARD_HOST=0.0.0.0`
- `HERMES_DASHBOARD_BASIC_AUTH_USERNAME`
- `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD`
- `HERMES_DASHBOARD_BASIC_AUTH_SECRET`

The manifest does not create an Ingress and does not expose the dashboard outside the cluster. The intended access pattern is `kubectl port-forward`.

If you do not want the dashboard, remove from the manifests:

- Dashboard env vars in `hermes-agent-deployment.yaml`
- Dashboard secrets from `kustomization.yaml`
- Dashboard port `9119` from the Deployment and Service

### Persistent storage

Hermes stores all mutable runtime state under `/opt/data`, including:

- `.env`
- `config.yaml`
- sessions
- memories
- skills
- logs

Do not run multiple Hermes gateway pods against the same `/opt/data` volume at the same time.

### Model configuration

The seeded `config.yaml` defaults Hermes to your local llama.cpp server and defines these aliases:

- `local` → `custom:llamacpp` / `/opt/models/Qwen3.8-27B-Q4_K_M.gguf`
- `opencode-go` → `opencode-go` / `deepseek-v4-flash`
- `openai-oauth` → `openai-codex` / `gpt-5.4`

In Telegram or any Hermes chat surface, you can switch models with commands like:

```text
/model local
/model opencode-go
/model openai-oauth
```

OpenCode Go is secret-based and works as soon as the pod starts. OpenAI OAuth requires the one-time bootstrap step above.

### Model endpoint notes

Hermes is pointed at the local llama.cpp server: `https://ai.styxut.net/v1`, serving `/opt/models/Qwen3.8-27B-Q4_K_M.gguf` (Qwen3.8 27B Q4_K_M). The server is configured with a 102400-token context, so the 100k `context_length` in the seed config is served for real.

Note: the self-hosted Mem0 OSS stack still uses Ollama on `http://192.168.0.13:11434` for its extraction LLM (`smollm2:1.7b`) and embedder (`qwen3-embedding`). If Ollama is retired, Mem0's config in `hermes-agent-config.yaml` / `mem0-deployment.yaml` must be moved to an OpenAI-compatible LLM + embedding endpoint as well.

### Resource sizing

The included limits follow Hermes' documented guidance for a general-purpose deployment:

- Requests: `500m` CPU, `1Gi` memory
- Limits: `2` CPU, `4Gi` memory

If you disable browser-heavy workflows, you can often run comfortably with less memory.

## Security Notes

- Keep the Service internal unless you intentionally need LAN or internet access to the Hermes API
- The Hermes dashboard is enabled here with basic auth; keep using internal-only access or a trusted tunnel unless you intentionally add a hardened ingress
- Telegram polling mode avoids the need for a public webhook endpoint and is the safer default for a homelab K3s cluster
- Hermes' `/opt/data` volume contains sensitive state and secrets; treat the PVC contents as confidential
