# KnowsWlf Helm Charts

Production-ready Helm charts for Kubernetes, served as a Helm repository via GitHub Pages.

```bash
helm repo add knowswlf https://knowswlf.github.io/Charts/
helm repo update
helm search repo knowswlf
```

## Available Charts

| Chart | Description |
|-------|-------------|
| [maxkb](charts/maxkb) | [MaxKB](https://github.com/1Panel-dev/MaxKB) (Max Knowledge Brain) — an all-in-one enterprise-grade agent platform (RAG + agentic workflow + MCP) bundling the app, PostgreSQL + pgvector, and Redis. Deployed as a single-replica StatefulSet, with optional external PostgreSQL/Redis. |
| [hermes-web-ui](charts/hermes-web-ui) | [Hermes Web UI](https://github.com/EKKOLearnAI/hermes-web-ui) — a single pod bundling the full [Hermes Agent](https://hermes-agent.nousresearch.com/) (`FROM nousresearch/hermes-agent`), Playwright/Chromium headless browser, and managed gateway. |
| [docker-proxy](charts/docker-proxy) | Pull-through registry mirror proxies (Docker Hub, GCR, GHCR, Quay, `registry.k8s.io`, MCR, Elastic, NVCR, …) plus optional hosted private registries, behind Traefik. |

## Usage

```bash
# Install a chart
helm install my-release knowswlf/<chart-name>

# Show configurable values
helm show values knowswlf/<chart-name>

# Upgrade / uninstall
helm upgrade my-release knowswlf/<chart-name>
helm uninstall my-release
```

Example — install Hermes Web UI behind Traefik:

```bash
helm install hermes knowswlf/hermes-web-ui \
  --set secrets.anthropicApiKey="sk-ant-..." \
  --set traefik.enabled=true \
  --set traefik.host="hermes.example.com" \
  --set traefik.tlsSecretName="example-tls"
```

See the chart's own README for full configuration: **[charts/hermes-web-ui](charts/hermes-web-ui/README.md)**.

## Requirements

- Kubernetes 1.19+
- Helm 3.8+
- A persistent storage provisioner

## License

MIT
