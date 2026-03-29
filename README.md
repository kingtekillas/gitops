# gitops

gitops argo cd

## Ingress controller

This repository is authoritative for **Traefik** ingress usage.

Traefik is managed **externally** (outside this repository and outside Argo CD applications in `applications/`).
Workloads in this repo are configured to use the Traefik ingress class (for example, `className` / `ingressClassName: traefik`).
