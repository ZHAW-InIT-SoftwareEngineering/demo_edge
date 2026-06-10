# Demo Edge

Shared Caddy edge proxy for the VM.

This repository owns the public ingress layer:

- ports `80` and `443`
- TLS certificates managed by Caddy
- domain-to-container routing
- attachment to the shared external Docker network `edge`

Application repositories deploy their own containers independently and attach
their public-facing service to the same `edge` network. They should not publish
host ports `80` or `443`.

## Routes

- `https://demo.init.zhaw.ch` -> `demoobject-frontend:80`
- `https://llm-backend.cloudlab.zhaw.ch` -> `llm-backend-api:8000`

## VM Setup

Create the shared network once on the VM:

```bash
docker network create edge
```

Start the edge proxy:

```bash
docker compose up -d
```
