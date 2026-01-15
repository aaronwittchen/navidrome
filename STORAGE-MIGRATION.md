# Navidrome Storage Migration: NFS to Longhorn

## What We Did

Migrated the music storage from NFS (network file system) to Longhorn (local cluster storage).

### Before
```
Music files on NFS server (192.168.68.105)
    ↓ network read
Navidrome pod
    ↓ cluster network
Cloudflare Tunnel
    ↓ internet
User
```

### After
```
Music files on Longhorn (local SSD on worker node)
    ↓ local disk read
Navidrome pod
    ↓ cluster network
Cloudflare Tunnel
    ↓ internet
User
```

## Why

NFS was slow for streaming music because:

1. **Network overhead**: Every file read went over the network to the NFS server
2. **Latency**: Each request had to wait for network round-trip
3. **Bandwidth**: Limited by network speed between NFS server and cluster

Longhorn storage is faster because:

1. **Local disk**: Files are read directly from SSD on the worker node
2. **No network hop**: Data doesn't traverse the network for reads
3. **Replicated**: Longhorn replicates data across nodes for redundancy

## Changes Made

### New Files
- `base/music-pvc.yaml` - 20GB Longhorn PVC for music storage

### Modified Files
- `base/deployment.yaml` - Changed volume claim from `navidrome-music-pvc` to `navidrome-music-local-pvc`
- `base/kustomization.yaml` - Replaced `nfs-music.yaml` with `music-pvc.yaml`

### Removed Files
- `base/nfs-music.yaml` - Old NFS PV and PVC definitions

### Deleted Resources
- `navidrome-music-pv` - Old NFS PersistentVolume
- `navidrome-music-pvc` - Old NFS PersistentVolumeClaim

## Migration Process

1. Created new 20GB Longhorn PVC (`navidrome-music-local-pvc`)
2. Ran a Job to copy music from NFS to Longhorn:
   ```
   cp -rv /nfs-music/* /local-music/
   ```
3. Updated deployment to use new PVC
4. Deleted old NFS resources

## Adding New Music

Since music is now stored in the cluster (not on NFS), to add new music:

1. Create a temporary pod that mounts the music PVC
2. Copy files into it using `kubectl cp`

Example:
```bash
# Create a temporary pod
kubectl run music-upload --rm -it --restart=Never \
  --image=alpine \
  --overrides='{"spec":{"containers":[{"name":"music-upload","image":"alpine","command":["sleep","3600"],"volumeMounts":[{"name":"music","mountPath":"/music"}]}],"volumes":[{"name":"music","persistentVolumeClaim":{"claimName":"navidrome-music-local-pvc"}}]}}' \
  -n navidrome

# In another terminal, copy files
kubectl cp /path/to/new/album navidrome/music-upload:/music/

# Exit the pod when done
```

Or re-enable NFS temporarily for bulk uploads, then copy to Longhorn.
