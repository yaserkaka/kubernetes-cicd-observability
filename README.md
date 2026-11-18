# kubernetes-cicd-observability

Lab project where I deploy a small multi-service app to a k3s cluster using
GitHub Actions, and monitor it with Prometheus and Grafana.

Work in progress.

## Stack

- k3s on two Ubuntu 24.04 VMs (VMware Workstation)
- App: FastAPI API, static frontend on nginx, PostgreSQL, Redis
- Helm chart with staging and prod values
- NGINX Gateway Fabric + cert-manager (Let's Encrypt, DNS-01 via Cloudflare)
- HPA, PDB, network policies, ConfigMaps and Secrets
- GitHub Actions: tests, image scan, push to GHCR, deploy to the cluster
- kube-prometheus-stack: Prometheus, Grafana, Alertmanager

## Progress

- [x] Repo setup
- [ ] k3s cluster
- [ ] App and Dockerfiles
- [ ] Helm chart
- [ ] Gateway and TLS
- [ ] CI pipeline
- [ ] CD pipeline (staging, then prod with approval)
- [ ] Monitoring and alerts
- [ ] Testing: rollout under load, autoscaling, failure and rollback

## License

MIT
