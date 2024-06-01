---
title: On-Prem K8S
sidebar_position: 98
hide_title: true
categories: onprem, platform
---

# Building an On-Prem Kubernetes Cluster from Bare Metal

You have a rack of Dell servers, a network team that handed you Redfish IPs, and a mandate to run Kubernetes on-prem. Here's how to go from powered-off hardware to a production-ready cluster.

---

## Network Architecture

Each server has two network paths: a **data network** interface (eth0) and a dedicated **Redfish/iDRAC management** port. Both networks are accessible only via VPN (Tailscale/NetBird/WireGuard) — there is no public access to any management or cluster network. The only publicly reachable path is through a load balancer fronting the Kubernetes ingress controller, which is provisioned after the cluster is up.

The data network carries all cluster traffic — initially the Landing Zone VLAN for PXE provisioning, and later the production VLANs for Kubernetes workloads. The Redfish network is a separate out-of-band management plane used exclusively for hardware discovery, power control, and boot configuration.

```bash
[ Internet ] -> [ LB ] -> Ingress (only public path)

      ┌─────────┼─────────┐
      │         │         │
 Data Network / Landing Zone (eth0) — VPN only
      │         │         │
 ┌───────┐ ┌───────┐ ┌───────┐
 │ srv-01│ │ srv-02│ │ srv-03│  ...
 └───────┘ └───────┘ └───────┘
      │         │         │
 Redfish/iDRAC Network (iDRAC port) — VPN only
      └─────────┼─────────┘
                │
       [ Operator via VPN ]
```

---

## 1. Starting Point: Dell Servers & Redfish

The datacenter team racks and cables the Dell servers (PowerEdge R750, R650, etc.) and provides:

- **Redfish/iDRAC IPs** for out-of-band management
- **Redfish credentials** (service account, not personal)
- **Serial numbers & asset tags** mapped to rack positions

The Redfish BMC network is **not publicly accessible** — it sits behind the VPN. Only automation hosts and operators on the VPN can reach iDRAC endpoints.

---

## 2. Redfish Discovery & Network Parameters

Before touching any OS, use the Redfish API to discover hardware details from each server's BMC. Query the Systems endpoint to gather model, serial number, memory, and processor info. Query the EthernetInterfaces endpoint to get the MAC addresses for each NIC — you need the MAC of the PXE boot interface specifically.

```bash
# Query each server's system info
curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
  https://10.1.0.11/redfish/v1/Systems/System.Embedded.1 \
  | jq '{Model, SerialNumber, SKU, MemorySummary, ProcessorSummary}'

# Get NIC MAC addresses for PXE boot interfaces
curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
  https://10.1.0.11/redfish/v1/Systems/System.Embedded.1/EthernetInterfaces \
  | jq '.Members[]."@odata.id"'
```

Collect for each server:
- MAC address of the PXE boot NIC
- Current boot order (set to PXE-first via Redfish PATCH)
- NIC port mapping to switch ports

Set boot order to PXE via Redfish:

```bash
curl -sk -u "${REDFISH_USER}:${REDFISH_PASS}" \
  -X PATCH https://10.1.0.11/redfish/v1/Systems/System.Embedded.1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Pxe", "BootSourceOverrideEnabled": "Continuous"}}'
```

---

## 3. Switch Configuration: Landing Zone VLAN

Configure the Top-of-Rack (ToR) switches with a **Landing Zone VLAN** — a temporary network where new servers boot for provisioning.

- **VLAN 100** (example): Landing Zone — DHCP, PXE, Matchbox access only
- **VLAN 200+**: Production VLANs for the actual Kubernetes cluster

The Landing Zone VLAN is:
- **Not publicly routable** — VPN-protected, same as Redfish
- **Isolated** — no access to production networks
- **Temporary** — servers move to production VLANs after provisioning

All server-facing switch ports are initially configured as access ports on the Landing Zone VLAN so any new server that powers on will land here by default.

```bash
# Switch port config (Cumulus/SONiC style)
# interface swp1-swp20
#   bridge-access 100    <- Landing Zone VLAN for initial boot
```

---

## 4. Provisioning Server: Matchbox + PXE + Services

Stand up a Ubuntu Server (physical or VM) on the Landing Zone VLAN that runs:

| Service | Purpose |
|---------|---------|
| **Matchbox** | iPXE/PXE boot server — serves Talos Linux images |
| **DHCP (dnsmasq)** | Hands out IPs + points to Matchbox for PXE |
| **TFTP** | Serves iPXE bootloader |
| **HTTP** | Serves kernel/initrd/config |
| **Git server (Gitea)** | Stores cluster configs, Talos machine configs |
| **Docker Registry** | Local mirror for container images |

**Matchbox** runs as a container and serves boot profiles based on MAC address or labels. It listens on HTTP and provides iPXE scripts, kernel images, and initrd to booting servers.

**DHCP (dnsmasq)** is configured to hand out IPs from the landing zone range, set the default gateway, and chain-load iPXE. Non-iPXE clients get the undionly.kpxe bootloader via TFTP; iPXE clients get redirected to the Matchbox HTTP boot endpoint.

```bash
dnsmasq.conf essentials
dhcp-range=10.100.0.100,10.100.0.200,255.255.255.0,1h
dhcp-boot=tag:!ipxe,undionly.kpxe
dhcp-boot=tag:ipxe,http://10.100.0.10:8080/boot.ipxe
enable-tftp
```

---

## 5. PXE Boot into Talos Linux (Maintenance Mode)

When a server powers on in the Landing Zone VLAN:

1. **DHCP** assigns an IP from the landing zone range
2. **PXE/iPXE** loads from the TFTP/HTTP server
3. **Matchbox** serves a Talos Linux image based on a default profile
4. **Talos boots in maintenance mode** — no cluster config applied yet

The boot flow: Server powers on → PXE boot on VLAN 100 → gets a DHCP IP → iPXE fetches the Matchbox profile → downloads Talos kernel and initrd → boots Talos in maintenance mode → Talos API becomes available on port 50000.

```bash
 Boot sequence:
 Server powers on
   -> PXE boot (VLAN 100)
   -> DHCP: 10.100.0.150
   -> iPXE -> Matchbox profile
   -> Downloads Talos kernel + initrd
   -> Boots Talos in maintenance mode
   -> Talos API on 10.100.0.150:50000
```

In maintenance mode, Talos exposes its API but does **not** join any cluster. It waits for configuration.

---

## 6. Discovery: Identify Servers by MAC

Now you have servers booted into Talos maintenance mode. Correlate them using three data sources:

| Source | Data |
|--------|------|
| Redfish | MAC → Serial → Rack position |
| DHCP leases | MAC → Landing Zone IP |
| Talos maintenance API | IP → hardware info (disks, NICs) |

Query each Talos node in maintenance mode using **talosctl** to get disk info and network links:

```bash
talosctl -n 10.100.0.150 --talosconfig=maintenance.yaml disks
talosctl -n 10.100.0.150 --talosconfig=maintenance.yaml get links
```

Cross-reference the MAC addresses from Redfish discovery with the DHCP lease table and the Talos API responses. Build a server mapping:

```yaml
servers:
  - serial: "FXTK123"
    mac: "b8:ce:f6:aa:bb:01"
    landing_ip: "10.100.0.150"
    role: control-plane
    cluster: prod-01
  - serial: "FXTK124"
    mac: "b8:ce:f6:aa:bb:02"
    landing_ip: "10.100.0.151"
    role: worker
    cluster: prod-01
```

---

## 7. Generate Talos Machine Configs

This is an independent step — generate configs before or in parallel with discovery.

```bash
# Generate cluster secrets (do this once, store securely)
talosctl gen secrets -o secrets.yaml

# Generate configs for the cluster
talosctl gen config prod-01 https://k8s-api.internal:6443 \
  --with-secrets secrets.yaml \
  --config-patch-control-plane @patches/controlplane.yaml \
  --config-patch-worker @patches/worker.yaml \
  --output rendered/
```

Patches customize each config for:
- **Network config** — production VLAN IPs, NIC bonding (LACP 802.3ad), static routes
- **Disk config** — which disk to install Talos onto
- **Cluster-specific settings** — CNI choice, etcd settings, API server flags

Per-node patches assign static IPs on the production VLAN:

```yaml
machine:
  network:
    interfaces:
      - interface: bond0
        addresses:
          - 10.200.0.11/24
        routes:
          - network: 0.0.0.0/0
            gateway: 10.200.0.1
        bond:
          mode: 802.3ad
          interfaces:
            - enp1s0f0
            - enp1s0f1
        vlans:
          - vlanId: 200
            addresses:
              - 10.200.0.11/24
```

---

## 8. Apply Talos Config & Boot into Production

Apply the generated config to each server in maintenance mode:

```bash
# Apply control plane config
talosctl apply-config --insecure \
  -n 10.100.0.150 \
  --file rendered/controlplane.yaml \
  --config-patch @patches/node-srv01.yaml

# Apply worker config
talosctl apply-config --insecure \
  -n 10.100.0.151 \
  --file rendered/worker.yaml \
  --config-patch @patches/node-srv02.yaml
```

Once config is applied:
1. Talos writes config to disk
2. Server reboots
3. Network reconfigures to **production VLAN (200)**
4. Server is now off the Landing Zone
5. Talos starts kubelet and joins the cluster

---

## 9. Bootstrap the Cluster

Once control plane nodes are up on the production network:

```bash
# Bootstrap etcd on the first control plane node
talosctl bootstrap -n 10.200.0.11 --talosconfig=talosconfig

# Get kubeconfig
talosctl kubeconfig -n 10.200.0.11 --talosconfig=talosconfig

# Verify
kubectl get nodes
```

You should see your control-plane and worker nodes in Ready state.

---

## 10. Day-2: AuthN/AuthZ & Cluster Components

Bootstrap the cluster with essential components (GitOps via Flux/ArgoCD or direct apply):

### Authentication & Authorization
- **Dex / Gangway** — OIDC for kubectl access tied to corporate IdP
- **RBAC policies** — namespace-scoped roles, cluster-admin limited to break-glass

### Networking
- **Cilium** (or Calico) — CNI with network policies
- **MetalLB** — bare-metal LoadBalancer for Services

### Storage
- **Rook-Ceph** or **Longhorn** — distributed storage from local disks
- **CSI driver** for any external SAN/NAS

### Observability
- **Prometheus + Grafana** — metrics
- **Loki** — logs
- **Hubble** (if Cilium) — network observability

### Ingress & Certificates
- **Ingress-NGINX** or **Envoy Gateway**
- **cert-manager** — automated TLS from internal CA

### GitOps
- **ArgoCD** or **Flux** — declarative cluster state from Git

---

## Summary: The Flow

1. Datacenter racks servers, provides Redfish IPs (VPN-protected)
2. Redfish discovery → MAC, serial, boot config
3. Switches set to Landing Zone VLAN (PXE boot)
4. Matchbox + DHCP serves Talos images
5. Servers PXE boot → Talos maintenance mode
6. Identify servers (Redfish + Talos + DHCP correlation)
7. Generate Talos configs (control-plane / worker)
8. Apply config → servers reboot into production VLAN
9. Bootstrap etcd → Kubernetes cluster is live
10. Deploy authn/authz, CNI, storage, observability

The entire provisioning path — from bare metal to running pods — touches **zero public networks**. Redfish, Landing Zone, and production VLANs are all VPN-protected or isolated. No SSH. No manual OS install. Immutable infrastructure from the start.

