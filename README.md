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

## GitHub Actions Deployment

The workflow `.github/workflows/deploy.yaml` deploys this edge stack to the VM.
It uploads `docker-compose.yaml` and `Caddyfile` to the VM deploy path, ensures
the external Docker network `edge` exists, and starts `demo-edge-caddy`.

Required GitHub configuration:

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `VM_HOST` | Variable | Yes | - | Public hostname or IP address of the VM. |
| `VM_USER` | Variable | No | `ubuntu` | SSH user on the VM. |
| `DEPLOY_PATH` | Variable | No | `./demo_edge` | Target path on the VM where edge files are uploaded. |
| `SSH_PRIVATE_KEY` | Secret | Yes | - | Private key used to SSH into the VM. |

## Deployment Order

On a fresh VM, cleaned VM, or after removing the edge container, deploy this
repository first:

1. `demo_edge`
2. `DemoObject`
3. `llm_backend`

Only `demo_edge` needs to be first. `DemoObject` and `llm_backend` are
independent after the edge proxy is running and can be deployed in either order.

This ordering matters because the app deployment workflows validate public
reachability through Caddy at the end. The app containers may start correctly
without the edge proxy, but their public URL checks can fail if
`demo-edge-caddy` is not already serving ports `80` and `443`.

After the first successful deployment, the repositories can deploy
independently:

- changes in `DemoObject` only require the `DemoObject` workflow
- changes in `llm_backend` only require the `llm_backend` workflow
- changes to domains, Caddy routes, or edge settings require this workflow

Re-run this workflow before app workflows when:

- the VM was wiped or Docker was fully pruned
- `demo-edge-caddy` was removed
- the `edge` network was removed
- ports `80` or `443` are occupied by another container
- a Caddy route, domain, app container name, or exposed app port changed

## Cleanup

Before moving from an old app-owned Caddy setup to this shared edge setup,
remove any old Caddy container that owns public ports `80` and `443`.

For example:

```bash
docker rm -f demoobject-caddy
```

If all app data is disposable, Docker can be fully cleaned before redeploying:

```bash
docker rm -f demoobject-caddy demoobject-frontend demoobject-api demoobject-mongo
docker system prune -af --volumes
rm -rf ~/demoobject ~/demo_edge ~/llm_backend
```

Do not delete GitHub Actions runner files from the VM home directory unless the
runner itself is being removed.
