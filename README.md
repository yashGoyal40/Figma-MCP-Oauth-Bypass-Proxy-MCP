# Figma MCP Server – Setup Guide

This guide walks you through installing Claude Code, authenticating with Figma OAuth, obtaining credentials, and deploying the Figma MCP server to Kubernetes.

---

## Step 1: Install Claude Code and Figma Plugin

1. **Install Claude Code**  
   Download and install from [https://claude.ai/download](https://claude.ai/download).

2. **Add the Figma MCP**  
   The setup script will add `figma-remote-mcp` to Claude CLI. Alternatively, add it manually:
   ```bash
   claude mcp add --transport http figma-remote-mcp https://mcp.figma.com/mcp
   ```

---

## Step 2: Authenticate with Figma OAuth

1. **Run the setup script:**
   ```bash
   chmod +x scripts/setup-figma-oauth.sh
   ./scripts/setup-figma-oauth.sh
   ```

2. **Follow the prompts:**
   - The script adds figma-remote-mcp to Claude CLI
   - In another terminal, run `claude`
   - Type `/mcp`
   - Select `figma-remote-mcp` and click **Authenticate**
   - Complete the Figma login in your browser
   - Return to the script terminal and press ENTER

3. **Capture the output** – the script prints:
   ```
   FIGMA_OAUTH_CLIENT_ID=<value>
   FIGMA_OAUTH_CLIENT_SECRET=<value>
   FIGMA_OAUTH_REFRESH_TOKEN=<value>
   ```

---

## Step 3: Add Credentials to k8s/secret.yaml

1. Open `k8s/secret.yaml`
2. Replace the placeholders with your values:
   - `FIGMA_OAUTH_CLIENT_ID` – from script output
   - `FIGMA_OAUTH_CLIENT_SECRET` – from script output
   - `FIGMA_OAUTH_REFRESH_TOKEN` – from script output
3. Generate and set `MCP_AUTH_TOKEN`:
   ```bash
   openssl rand -hex 32
   ```

---

## Step 4: Update k8s/httpProxy.yaml

1. Set your FQDN (e.g. `figma-mcp.example.com`):
   ```yaml
   virtualhost:
     fqdn: figma-mcp.example.com
   ```
2. Set your TLS secret name for the wildcard cert

---

## Step 5: Build the Docker Image

```bash
# Build (replace REGISTRY and TAG as needed)
docker build -t figma-mcp-server:latest .

# Optional: push to your registry
docker tag figma-mcp-server:latest <REGISTRY>/figma-mcp-server:latest
docker push <REGISTRY>/figma-mcp-server:latest
```

---

## Step 6: Update k8s/deployment.yaml

1. Set the image in `k8s/deployment.yaml`:
   ```yaml
   image: <REGISTRY>/figma-mcp-server:latest
   ```
   Example: `image: ghcr.io/myorg/figma-mcp-server:latest`

2. **(Optional)** Add OpenTelemetry env vars for traces/metrics. The server runs fine without them (OTEL is disabled by default). To enable, add to the deployment `env` section:
   ```yaml
   - name: OTEL_SERVICE_NAME
     value: "figma-mcp-server"
   - name: OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
     value: "http://<your-otel-collector>:4318/v1/traces"
   - name: OTEL_EXPORTER_OTLP_METRICS_ENDPOINT
     value: "http://<your-otel-collector>:4318/v1/metrics"
   - name: OTEL_METRICS_EXPORTER
     value: "otlp"
   - name: OTEL_EXPORTER_OTLP_COMPRESSION
     value: "gzip"
   ```
   Replace `<your-otel-collector>` with your OTLP endpoint (e.g. `otel-collector`, `cubeapm`, or a full URL).

---

## Step 7: Apply Kubernetes Manifests

```bash
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/httpProxy.yaml
```

---

## Quick Reference

| Step | Action |
|------|--------|
| 1 | Install Claude Code + Figma plugin |
| 2 | Run `./scripts/setup-figma-oauth.sh` and complete OAuth |
| 3 | Add credentials to `k8s/secret.yaml` |
| 4 | Update `k8s/httpProxy.yaml` (FQDN, TLS) |
| 5 | Build image: `docker build -t figma-mcp-server:latest .` |
| 6 | Update `k8s/deployment.yaml` with image name |
| 7 | Apply: `kubectl apply -f k8s/` |
