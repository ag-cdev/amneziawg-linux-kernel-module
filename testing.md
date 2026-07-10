# Testing: WG_CMD_GET_DEVICE_STATS

## Environment

- Lab VM: `ssh sysadmin@ubuntu-starvpn` (192.168.229.138)
- Kernel: 6.8.0-124-generic
- Module: Custom-built `amneziawg.ko` with GET_DEVICE_STATS
- Userspace: `amneziawg-tools` (`awg`) installed from PPA `ppa:amnezia/ppa`
- Standard `wireguard` kernel module **blacklisted** to avoid genl family conflict

## Setup

```bash
# Load custom module
sudo insmod ~/amneziawg/src/amneziawg.ko

# Create wg0 using amneziawg link type
sudo ip link add wg0 type amneziawg

# Configure with awg
sudo awg set wg0 listen-port 51820 private-key /tmp/server-privkey
sudo awg set wg0 peer <client-pubkey> allowed-ips 10.0.0.3/32
sudo ip addr add 10.0.0.1/24 dev wg0
sudo ip link set wg0 up

# Add client netns
sudo ip netns exec awg-client awg set wg0 peer <server-pubkey> \
  allowed-ips 10.0.0.0/24 endpoint 192.168.99.1:51820
```

## Results

### WG_CMD_GET_DEVICE (full dump) — 3 peers
```
ifindex=14 ifname=wg0
  public_key (raw bytes)
  rx_bytes=0
  tx_bytes=0
  last_handshake=0.000000000        ← peer 1 (never handshaked)
  public_key (raw bytes)
  rx_bytes=0
  tx_bytes=0
  last_handshake=0.000000000        ← peer 2 (never handshaked)
  public_key (raw bytes)
  rx_bytes=11252
  tx_bytes=11196
  last_handshake=1783711466.478...  ← peer 3 (actively connected)
  total peers reported: 3
```

### WG_CMD_GET_DEVICE_STATS (no filter) — 3 peers
Same output as GET_DEVICE but without allowed-ips, private-key, etc.

### WG_CMD_GET_DEVICE_STATS (cutoff=60s) — 1 peer
```
  public_key (raw bytes)
  rx_bytes=11252
  tx_bytes=11196
  last_handshake=1783711466.478...
  total peers reported: 1           ← stale peers filtered out
```
Correctly excludes 2 peers with no handshake (last_handshake=0 < cutoff).

## Verification Checklist

- [x] GET_DEVICE returns all peers with full config + stats
- [x] GET_DEVICE_STATS returns all peers with only runtime stats
- [x] GET_DEVICE_STATS + cutoff filters peers with no recent handshake
- [x] Stats counters (rx_bytes/tx_bytes) update with traffic
- [x] Last handshake timestamp reports correctly
- [x] `awg` userspace tool successfully configures and queries the module
- [x] Standard `wireguard` module blacklisted — no genl family conflict
- [x] Module auto-loads at boot (via `/etc/modules-load.d/amneziawg.conf`)
- [x] Backward compatible: existing GET_DEVICE behavior unchanged

## Test Tool

`src/wg-stats-test.c` — libnl-based test that calls all three dump commands.

## Files

| File | Purpose |
|------|---------|
| `src/uapi/wireguard.h` | Command/attribute enums |
| `src/netlink.c` | Handler + shared helper |
| `GET_DEVICE_STATS.md` | Feature design doc |
| `testing.md` | This file |
