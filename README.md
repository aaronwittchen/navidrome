# Navidrome Setup for K8s

Self-hosted music streaming server using Navidrome.

## Prerequisites

### 1. Set up NFS Server on Arch VM (192.168.68.105)

On your Arch VM, install and configure NFS server to share the Music folder:

```bash
# Install NFS server
sudo pacman -S nfs-utils

# Create exports file
sudo tee /etc/exports <<EOF
/home/onion/Music 192.168.68.0/24(ro,sync,no_subtree_check,all_squash,anonuid=1000,anongid=1000)
EOF

# Enable and start NFS services
sudo systemctl enable --now nfs-server
sudo exportfs -arv

# Open firewall if needed (or ensure firewall allows NFS)
# sudo firewall-cmd --permanent --add-service=nfs
# sudo firewall-cmd --reload
```

### 2. Verify NFS Export

From any K8s node, verify you can access the NFS share:

```bash
# Install nfs-utils on NixOS nodes if needed
# Then test mount
sudo mount -t nfs 192.168.68.105:/home/onion/Music /mnt
ls /mnt
sudo umount /mnt
```

## Deployment

### Using Longhorn Storage (Recommended)

```bash
kubectl apply -k overlays/longhorn
```

### Using Default Storage Class

```bash
kubectl apply -k base
```

## DNS Setup

Add to your DNS or `/etc/hosts`:

```
<GATEWAY_IP>  navidrome.k8s.local
```

## Access

Once deployed, access Navidrome at: https://navidrome.k8s.local

On first access, you'll be prompted to create an admin account.

## Configuration

Environment variables in `deployment.yaml`:

| Variable | Default | Description |
|----------|---------|-------------|
| ND_SCANSCHEDULE | 1h | How often to scan for new music |
| ND_LOGLEVEL | info | Log level (debug, info, warn, error) |
| ND_SESSIONTIMEOUT | 24h | Session timeout |
| ND_ENABLETRANSCODINGCONFIG | true | Allow transcoding settings in UI |

## Troubleshooting

### Pod not starting - NFS mount issues

1. Verify NFS server is running on Arch VM:
   ```bash
   systemctl status nfs-server
   ```

2. Check exports:
   ```bash
   exportfs -v
   ```

3. Test from a K8s node:
   ```bash
   showmount -e 192.168.68.105
   ```

### Music not showing up

1. Check if music folder is mounted:
   ```bash
   kubectl exec -n navidrome deploy/navidrome -- ls /music
   ```

2. Trigger a manual scan via the Navidrome web UI (Settings > Scan)

### Permission issues

The NFS export uses `all_squash` with `anonuid=1000` to map all access to UID 1000 (the typical first user). Ensure your Music folder is readable by this user.
