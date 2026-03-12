# proxmox-homelab

A production-grade private cloud built from the ground up as a learning environment and personal infrastructure platform. What started as a few old desktop PCs in early 2025 has grown into a 13-node Proxmox cluster running enterprise-pattern security, networking, and DevOps tooling.

---

## Architecture Overview

**Compute:** 6 tower/rack servers (serv01–06) and 7 mini PCs (mini01–07), all running Proxmox VE. Workloads are distributed across dedicated VMs and LXC containers depending on resource requirements.

**Networking:** VLAN-segmented flat network with a management VLAN (10.0.0.0/24) and planned expansion into internal services, internet-exposed, and IoT/WiFi tiers. All inter-VLAN routing is controlled by firewall policy.

**Security posture:** Default-deny inbound and outbound on all hosts, enforced by Proxmox firewall using Security Groups and IPSets. Direct web GUI access restricted to Traefik IPs and specific management machines. Break-glass local accounts exist per service with complex passwords stored separately in Vaultwarden; all other local accounts disabled.

---

## Core Services

| Service | Host | Role |
|---|---|---|
| Technitium DNS | dns-a + 2 redundant LXCs | Internal authoritative DNS, split-horizon resolution |
| Traefik | rp-01, rp-02 | Redundant reverse proxies, wildcard TLS termination |
| Authentik | auth-01 | SSO and MFA enforcement across all web services |
| Gitea | mgmt-01 | Local Git hosting, mirrored to GitHub for offsite backup |
| Portainer | mgmt-01 (+ agents on all hosts) | Centralized Docker management |
| Ansible + Semaphore | mgmt-01 | Cluster-wide automation and scheduled health checks |
| Prometheus + Grafana + Loki | obs-01 | Metrics, dashboards, log aggregation |
| Alertmanager | obs-01 | Alert routing and email notifications |
| Proxmox Backup Server | pbs-serv02, pbs-serv03 | Redundant VM/LXC backup chain |
| Vaultwarden | vault-01 | Secrets and password management |
| step-ca + Offline Root CA | Internal PKI | Certificate issuance for services requiring direct TLS |

---

## Key Design Patterns

**GitOps deployments** — Complex stacks (reverse proxies, observability, auth, management) each have a dedicated Gitea repository. Docker Compose files live in a `/compose` subdirectory. Portainer is configured to pull from the repo and redeploy on change. All infrastructure changes go through Git.

**Zero-trust firewall** — Proxmox firewall enforces default-deny at the hypervisor level for both inbound and outbound traffic. Services communicate only on explicitly permitted ports to permitted sources, using IPSets for source groups and Security Groups for reusable rule sets.

**Automated health checking** — Semaphore runs a weekly "Baseline Health Check" playbook targeting every host. It verifies reachability, checks disk usage, validates ZFS pool health (on Proxmox nodes), checks DNS resolution, and verifies Chrony sync status. Results are consolidated into an email report.

**Layered backup chain** — `pbs-serv02` runs backup, verify, and prune jobs against all VMs and containers. `pbs-serv03` runs sync jobs from `pbs-serv02` approximately 2 hours later. A weekly rsync job replicates a backup of the PBS datastore to a Cloudflare R2 bucket for offsite cold storage.

**Observability stack** — Prometheus scrapes metrics from cAdvisor, node_exporter, pve_exporter, pbs_exporter, and blackbox_exporter across all hosts. Grafana provides dashboards. Alloy ships logs to Loki. Weekly iPerf3 tests run between hosts to baseline network throughput and catch degradation early.

---

## Storage

NFS storage is provided cluster-wide by `serv01`, which runs a ZFS RAIDZ mirror across 4 hard drives. Proxmox hosts mount NFS shares for VM storage, ISO libraries, and backup staging.

---

## Domains & TLS

- **reladox.net** — owned, managed via Cloudflare
- **\*.lab.reladox.net** — internal service FQDNs, resolved by Technitium
- **\*.web.reladox.net** — web GUI access via Traefik on rp-01/rp-02, wildcard TLS cert
- **\*.reladox.net** — Proxmox hosts (legacy naming); Technitium CNAME records bridge lab FQDNs to these hostnames

TLS is terminated at Traefik for all proxied services. Backend services communicate over HTTP internally. Services requiring direct TLS (bypassing Traefik) use certificates issued by step-ca.

---

## Internet-Exposed Services

A dedicated internet-exposed VLAN hosts game servers accessible to friends and family. Exposure is managed through Cloudflare Dynamic DNS (running in a Debian 13 container), Cloudflare DNS records, and a Cloudflare Tunnel for applicable services — minimizing direct port exposure where possible.

---

## Remote Access

Tailscale is deployed on all primary servers, providing secure remote access without exposing management interfaces to the public internet.

---

## Tech Stack Summary

**Hypervisor:** Proxmox VE  
**Containers/VMs:** LXC (Debian), VMs (Ubuntu Server LTS, Debian 13, Windows 11)  
**Networking:** Technitium DNS, Traefik, Cloudflare, Tailscale, VLAN segmentation  
**Security:** Proxmox firewall (IPSets/Security Groups), Authentik, step-ca, Vaultwarden  
**DevOps:** Gitea, GitHub, Portainer, Ansible, Semaphore, Docker Compose  
**Observability:** Prometheus, Grafana, Loki, Alertmanager, Alloy, cAdvisor, multiple exporters  
**Storage:** ZFS RAIDZ, NFS, Cloudflare R2 (offsite), Proxmox Backup Server  
**Scripting:** Python, Bash
