# NFS Storage Node — ThinkPad P73 to K3s Cluster

Follow-up to [k3s-hybrid-cluster.md](k3s-hybrid-cluster.md). This covers using the laptop purely as an NFS server rather than joining it as a k3s node.

## Do You Need Tailscale?

**Yes, you still need a VPN tunnel** (Tailscale, WireGuard, etc.) because:

```mermaid
flowchart LR
    subgraph Problem["Without VPN"]
        VPS_NO["Hetzner VPS<br/>Public IP"] -.-x|"Cannot reach<br/>laptop directly"| LAPTOP_NO["ThinkPad<br/>Behind NAT"]
    end

    subgraph Solution["With Tailscale"]
        VPS_YES["Hetzner VPS<br/>100.x.x.1"] <-->|"WireGuard tunnel<br/>NAT traversal"| LAPTOP_YES["ThinkPad<br/>100.x.x.2"]
    end

    Problem ~~~ Solution
```

- The laptop sits behind your home router (NAT) — the VPS cannot initiate connections to it
- NFS requires a **routable IP** between server and client — it has no NAT traversal of its own
- Tailscale gives both machines stable IPs on a private mesh (100.x.x.0/24) with automatic NAT hole-punching
- Unlike joining as a k3s node, **you don't need k3s agent on the laptop** — just Tailscale + NFS server

## Architecture Comparison

```mermaid
graph TB
    subgraph Option_A["Option A: Laptop as k3s Agent + Longhorn"]
        VPS_A["k3s server"] <-->|"Tailscale"| AGENT["k3s agent<br/>+ Longhorn"]
        AGENT --> DISK_A["Local Storage"]
    end

    subgraph Option_B["Option B: Laptop as Standalone NFS"]
        VPS_B["k3s server<br/>+ NFS provisioner"] <-->|"Tailscale"| NFS_SRV["NFS server<br/>(no k3s needed)"]
        NFS_SRV --> DISK_B["Exported Storage"]
    end

    style Option_B fill:#f0fff0,stroke:#2d8b2d
```

Option B is **simpler** — fewer moving parts, no kubelet overhead on the laptop, and the laptop doesn't need to run k3s at all.

## Setup

### Step 1: Tailscale (both machines)

```bash
# On both Hetzner VPS and ThinkPad
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Note the Tailscale IPs:
```bash
tailscale ip -4  # e.g., 100.64.0.1 (VPS), 100.64.0.2 (laptop)
```

### Step 2: NFS Server (ThinkPad)

```bash
# Install NFS server
sudo apt install nfs-kernel-server

# Create export directory
sudo mkdir -p /srv/nfs/k3s-data
sudo chown nobody:nogroup /srv/nfs/k3s-data
sudo chmod 777 /srv/nfs/k3s-data

# Export to the VPS Tailscale IP only
echo "/srv/nfs/k3s-data  100.64.0.1/32(rw,sync,no_subtree_check,no_root_squash)" \
  | sudo tee -a /etc/exports

# Apply and start
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server
```

### Step 3: Verify NFS from VPS

```bash
# On the Hetzner VPS
sudo apt install nfs-common

# Test mount
sudo mount -t nfs 100.64.0.2:/srv/nfs/k3s-data /mnt
ls /mnt
sudo umount /mnt
```

### Step 4: NFS Provisioner in k3s

Install the NFS subdir external provisioner via Helm so k3s can dynamically create PersistentVolumes on the laptop's NFS share:

```bash
# On the VPS (where kubectl works)
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=100.64.0.2 \
  --set nfs.path=/srv/nfs/k3s-data \
  --set storageClass.name=nfs-laptop \
  --set storageClass.defaultClass=false
```

### Step 5: Use It

```yaml
# Example PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  storageClassName: nfs-laptop
  accessModes:
    - ReadWriteMany      # NFS supports RWX
  resources:
    requests:
      storage: 50Gi
```

## Full Data Flow

```mermaid
sequenceDiagram
    participant Pod as Pod on VPS
    participant K3S as k3s + NFS Provisioner
    participant TS as Tailscale Tunnel
    participant NFS as NFS Server (ThinkPad)
    participant Disk as /srv/nfs/k3s-data

    Pod->>K3S: Request PVC (storageClass: nfs-laptop)
    K3S->>TS: Mount NFS share 100.64.0.2:/srv/nfs/k3s-data
    TS->>NFS: NFS MOUNT (via WireGuard tunnel)
    NFS-->>TS: Export granted
    TS-->>K3S: Mount ready
    K3S-->>Pod: PV bound

    Note over Pod,Disk: Normal Operation
    Pod->>TS: NFS READ/WRITE
    TS->>NFS: Forwarded over tunnel
    NFS->>Disk: Filesystem I/O
    Disk-->>NFS: Data
    NFS-->>TS: NFS response
    TS-->>Pod: Data returned
```

## Where to Store Data on the Laptop

```mermaid
flowchart TD
    Q1{"What data?"} -->|"General k3s volumes"| ROOT_NFS["/srv/nfs/k3s-data<br/>on root NVMe (ext4)<br/>778 GB free"]
    Q1 -->|"Large archive/backups"| Q2{"Willing to reformat<br/>part of 4TB drive?"}
    Q2 -->|Yes| NEW_PART["Create ext4 partition<br/>on /dev/sda, mount to<br/>/srv/nfs/k3s-archive"]
    Q2 -->|No| BIND["Bind-mount a folder<br/>from /media/Arxv into<br/>/srv/nfs/ (NTFS, limited)"]

    ROOT_NFS -->|Recommended| DONE["Export via /etc/exports"]
    NEW_PART --> DONE
    BIND -->|"No Unix permissions,<br/>no symlinks"| CAVEAT["Works for basic<br/>file storage only"]
```

**Recommendation:** Use the root NVMe (ext4) for the NFS export. NTFS lacks proper Unix permission support, which causes issues with containers expecting POSIX semantics.

## Performance Expectations

| Factor | Impact |
|--------|--------|
| **Direct Tailscale connection** | Best case: adds ~1-2ms latency over the base internet path |
| **Relayed connection (DERP)** | Worse: 50-200ms+ latency — avoid for NFS |
| **Home upload speed** | Bottleneck for writes from VPS to laptop; fine for reads if download is fast |
| **NFS over WAN** | Acceptable for bulk storage, backups, media; not for databases or high-IOPS workloads |
| **Single CPU core** | NFS server itself is lightweight — not a concern |

To check if your connection is direct or relayed:
```bash
tailscale status  # Look for "direct" vs "relay" next to the peer
```

## Laptop Availability

Unlike the k3s agent approach, if the laptop goes offline the cluster itself is unaffected — only pods using `nfs-laptop` volumes will hang on I/O until the laptop returns. To handle this gracefully:

```bash
# Prevent sleep while NFS is serving
sudo systemd-inhibit --what=sleep --who=nfs --why="NFS storage active" sleep infinity &

# Or in /etc/systemd/logind.conf
HandleLidSwitchExternalPower=ignore
```

Consider mounting NFS with soft timeout on the VPS side so pods get errors instead of hanging indefinitely:

```yaml
# In the Helm values or StorageClass
mountOptions:
  - soft
  - timeo=30
  - retrans=3
```
