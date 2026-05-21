# k8s-infra-pocharlies

Infraestructura del cluster k3s: networking (Traefik, MetalLB, cert-manager), storage (Longhorn, MinIO, NFS), platform (Harbor, Vault, Authelia, Kyverno, GPU Operator, Velero), shared PostgreSQL + RabbitMQ + Redis

## Cluster
- **Master**: x86 ubuntu (192.168.50.142), k3s v1.32.5
- **Workers**: nvidia-dgx (dgx1 ARM64), gx10-ec3d (dgx2 ARM64), sauvage (WireGuard edge)

## GitOps
Gestionado por ArgoCD desde [k8s-gitops-pocharlies](https://github.com/pocharlies/k8s-gitops-pocharlies).
