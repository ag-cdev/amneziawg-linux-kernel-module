# WG_CMD_GET_DEVICE_STATS — Lightweight Statistics Dump

## Overview

A new Generic Netlink command for the AmneziaWG kernel module that returns
**only runtime peer statistics**, skipping all static configuration
(allowed IPs, endpoints, preshared keys, AmneziaWG extentions, etc.).

Designed for monitoring daemons that poll frequently at scale (~300k peers).

## What Changed

### Two files modified

| File | Change |
|------|--------|
| `src/uapi/wireguard.h` | Added `WG_CMD_GET_DEVICE_STATS` (= 3) to `enum wg_cmd`; added `WGDEVICE_A_HANDSHAKE_CUTOFF` (= 26) to `enum wgdevice_attribute`; added full documentation comment |
| `src/netlink.c` | Added shared helper `put_peer_stats_attrs()`; refactored `get_peer()` to use it (no behavior change); added `wg_get_device_stats_dump()` handler; added `genl_ops` entry; added policy entry; added `handshake_cutoff` to `dump_ctx` |

### What stayed the same

- `WG_CMD_GET_DEVICE` — byte-identical output
- `WG_CMD_SET_DEVICE` — completely unchanged
- Family name: `"amneziawg"`
- Family version: `2`
- All existing peer/device/allowedip attribute IDs

### Total diff

```
 src/netlink.c          | 102 ++++++++++++++++++++++++++++++++++++++--
 src/uapi/wireguard.h   |  49 ++++++++++++++++++
 2 files changed, 151 insertions(+), 16 deletions(-)
```

## Reply Format

```
NLMSG_HDR | GENL_HDR (cmd=WG_CMD_GET_DEVICE_STATS)
  WGDEVICE_A_IFINDEX   (NLA_U32)
  WGDEVICE_A_IFNAME    (NLA_NUL_STRING)
  WGDEVICE_A_PEERS     (NLA_NESTED)
    [0] NLA_NESTED
      WGPEER_A_PUBLIC_KEY           (32 bytes)
      WGPEER_A_LAST_HANDSHAKE_TIME  (struct __kernel_timespec)
      WGPEER_A_RX_BYTES             (NLA_U64)
      WGPEER_A_TX_BYTES             (NLA_U64)
    [0] NLA_NESTED
      ...
```

Only four fields per peer. No allowed IPs, endpoints, preshared keys,
keepalive intervals, protocol version, flags, or AmneziaWG config.

## Request Attributes

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `WGDEVICE_A_IFINDEX` | `NLA_U32` | One of | Interface index |
| `WGDEVICE_A_IFNAME` | `NLA_NUL_STRING` | One of | Interface name |
| `WGDEVICE_A_HANDSHAKE_CUTOFF` | `NLA_U32` | No | Only include peers with a handshake in the last N seconds |

## Building

### Prerequisites

- Linux kernel headers installed (matching running kernel)
- GCC, make, and standard build tools

### Build the module

```bash
cd src
make module
```

This produces `amneziawg.ko` in `src/`.

### Debug build (verbose, with debug symbols)

```bash
make module-debug
```

### Cross-compile for a different kernel

```bash
make -C /path/to/kernel/build M=$PWD modules
```

## Deploying on Servers

### 1. Load on a running system

```bash
# Unload existing module first (will disrupt VPN traffic!)
sudo rmmod amneziawg 2>/dev/null

# Load the new module
sudo insmod src/amneziawg.ko

# Verify
sudo modinfo amneziawg | grep version
sudo dmesg | tail
```

### 2. Persistent install (DKMS)

```bash
# Install the module files to the DKMS source tree
sudo make dkms-install

# Build and register with DKMS
sudo dkms add -m amneziawg -v $(cat src/version.h | grep -o '"[^"]*"' | head -1 | tr -d '"')
sudo dkms build -m amneziawg -v <version>
sudo dkms install -m amneziawg -v <version>
```

### 3. Persistent install (manual module-install)

```bash
sudo make module-install
sudo depmod -a
```

Then load at boot via `modprobe amneziawg` or an initramfs hook.

## Calling the New Command

### C (libnl-3.2)

```c
#include <netlink/genl/genl.h>
#include <netlink/genl/ctrl.h>
#include "src/uapi/wireguard.h"

int family_id;

static int parse_stats(struct nl_msg *msg, void *arg) {
    struct nlattr *attrs[WGDEVICE_A_MAX + 1];
    struct genlmsghdr *gnlh = nlmsg_data(nlmsg_hdr(msg));
    struct nlattr *peer_entry;
    int rem;

    nla_parse(attrs, WGDEVICE_A_MAX, genlmsg_attrdata(gnlh, 0),
              genlmsg_attrlen(gnlh, 0), NULL);

    printf("ifindex=%u ifname=%s\n",
           nla_get_u32(attrs[WGDEVICE_A_IFINDEX]),
           nla_get_string(attrs[WGDEVICE_A_IFNAME]));

    if (!attrs[WGDEVICE_A_PEERS])
        return NL_OK;

    nla_for_each_nested(peer_entry, attrs[WGDEVICE_A_PEERS], rem) {
        struct nlattr *peer[WGPEER_A_MAX + 1];
        nla_parse(peer, WGPEER_A_MAX, nla_data(peer_entry),
                  nla_len(peer_entry), NULL);

        char key[65];
        bin2hex(key, sizeof(key),
                nla_data(peer[WGPEER_A_PUBLIC_KEY]), 32);
        key[64] = '\0';

        struct __kernel_timespec *ts =
            nla_data(peer[WGPEER_A_LAST_HANDSHAKE_TIME]);

        printf("  peer=%s rx=%llu tx=%llu last_handshake=%lld.%09ld\n",
               key,
               nla_get_u64(peer[WGPEER_A_RX_BYTES]),
               nla_get_u64(peer[WGPEER_A_TX_BYTES]),
               (long long)ts->tv_sec, ts->tv_nsec);
    }
    return NL_OK;
}

int main(int argc, char **argv) {
    struct nl_sock *sock = nl_socket_alloc();
    genl_connect(sock);
    family_id = genl_ctrl_resolve(sock, "amneziawg");

    struct nl_msg *msg = nlmsg_alloc();
    genlmsg_put(msg, NL_AUTO_PID, NL_AUTO_SEQ, family_id,
                0, NLM_F_DUMP, WG_CMD_GET_DEVICE_STATS, 2);
    nla_put_u32(msg, WGDEVICE_A_IFINDEX, if_nametoindex("wg0"));

    /* Optional: only peers active in the last 60 seconds */
    nla_put_u32(msg, WGDEVICE_A_HANDSHAKE_CUTOFF, 60);

    nl_socket_modify_cb(sock, NL_CB_VALID, NL_CB_CUSTOM, parse_stats, NULL);
    nl_send_auto(sock, msg);
    nl_recvmsgs_default(sock);

    nlmsg_free(msg);
    nl_socket_free(sock);
    return 0;
}
```

Compile with:

```bash
gcc -o wg-stats wg-stats.c $(pkg-config --cflags --libs libnl-3.0 libnl-genl-3.0)
```

### Python (ctypes)

```python
import socket, struct, ctypes

# Build a raw netlink message and send via NETLINK_GENERIC
# (see kernel documentation for GENL_CTRL_CMD_GETFAMILY resolution)

NLMSG_DONE     = 3
NLM_F_REQUEST  = 1
NLM_F_DUMP     = 0x300
GENL_ID_CTRL   = 0x10
SOL_NETLINK    = 270

WG_GENL_NAME    = b"amneziawg\x00"
WG_GENL_VERSION = 2
WG_CMD_GET_DEVICE_STATS = 3
WGDEVICE_A_IFINDEX      = 1
WGDEVICE_A_HANDSHAKE_CUTOFF = 26

# First resolve the family ID (omitted for brevity — use GENL_CTRL_CMD_GETFAMILY)
family_id = ...  # result of control socket lookup

sock = socket.socket(socket.NETLINK_GENERIC, socket.SOCK_RAW, 0)
sock.bind((0, 0))

# Build the dump request
msg = bytearray()
msg += struct.pack('IHHBBII',  # nlmsghdr
    0,              # len (patched later)
    family_id,      # nlmsg_type
    NLM_F_REQUEST | NLM_F_DUMP, 0, 0,  # flags, seq, pid
)
msg += struct.pack('BBH',  # genlmsghdr
    WG_CMD_GET_DEVICE_STATS, WG_GENL_VERSION, 0,
)
# WGDEVICE_A_IFINDEX
msg += struct.pack('HH', 1, 4) + struct.pack('I', if_nametoindex('wg0'))
# Optional: WGDEVICE_A_HANDSHAKE_CUTOFF = 300
msg += struct.pack('HH', WGDEVICE_A_HANDSHAKE_CUTOFF, 4) + struct.pack('I', 300)

struct.pack_into('I', msg, 0, len(msg))  # fix length

sock.send(msg)

# Receive in a loop until NLMSG_DONE
while True:
    data = sock.recv(65535)
    # parse NLMSGHDR + GENLHDR + attributes
    # ... (see kernel netlink docs for parsing)
```

## Performance

### Which CPU costs are **eliminated**

| Cost | `GET_DEVICE` | `GET_DEVICE_STATS` |
|------|-------------|-------------------|
| Allowed IP trie walk (`wg_allowedips_read_node`) | O(n * cidrs) | — |
| Allowed IP serialization (`get_allowedips`) | O(n * cidrs) | — |
| Preshared key (rwsem + memcpy) | per peer | — |
| Endpoint (rwlock + sockaddr copy + AF check) | per peer | — |
| Persistent keepalive / protocol version | per peer | — |
| Device keys / port / fwmark / AmneziaWG config | 18 nla_put | 2 nla_put |
| `peer->flags` + `advanced_security` flag | per peer | — |

### Which CPU costs are **unavoidable**

| Cost | Reason |
|------|--------|
| Peer list iteration | Must visit every peer (or until skb full) |
| `rtnl_lock` + `device_update_lock` | Consistency across dump |
| Public key (rwsem + 32-byte copy) | Required to identify the peer |
| 3 stats fields (direct read + nla_put) | The actual data requested |
| `genlmsg_put` / `nla_nest_start` / `nla_nest_end` | Netlink framing |

### Expected improvement

- **CPU cycles**: ~5–25× fewer per peer depending on CIDR count
- **Message size**: ~80 bytes per peer vs. 200+ for `GET_DEVICE`
- **Fewer syscalls**: more peers fit per dump message

## Benchmarking

```bash
# Compare kernel-side cycles
perf stat -e cycles,instructions,cache-misses \
    -r 5 ./wg-monitor --cmd get_device
perf stat -e cycles,instructions,cache-misses \
    -r 5 ./wg-monitor --cmd get_device_stats

# Flamegraph per peer
perf record -g ./wg-monitor --cmd get_device
perf report -g --sort=comm,dso,symbol

perf record -g ./wg-monitor --cmd get_device_stats
perf report -g
```

Key functions to disappear from the GET_DEVICE flamegraph:

- `wg_allowedips_read_node`
- `get_allowedips`
- `nla_put` calls for preshared key, endpoint, keepalive
- `generate_ipv4_address_with_prefix` / `generate_ipv6_address_with_prefix`

## Upstream Porting

This feature is designed to be portable to upstream WireGuard with
minimal changes:

1. Copy `put_peer_stats_attrs()` and `wg_get_device_stats_dump()`
   to WireGuard's `netlink.c`
2. Replace `"amneziawg"` references with `"wireguard"`
3. Add `WG_CMD_GET_DEVICE_STATS` and `WGDEVICE_A_HANDSHAKE_CUTOFF`
   to WireGuard's `uapi/wireguard.h`
4. Wire up in `genl_ops[]`

The reply attributes (`WGPEER_A_PUBLIC_KEY`, `WGPEER_A_LAST_HANDSHAKE_TIME`,
`WGPEER_A_RX_BYTES`, `WGPEER_A_TX_BYTES`) use the exact same numeric IDs
as WireGuard's existing UAPI — no redefinition needed.

## Kernel Version Update Guide

When upgrading to a newer kernel, the `COMPAT_*` macros in `netlink.c`
may need adjustment. Here is what to check and update.

### Files that may need changes

| File | What to check |
|------|---------------|
| `src/netlink.c` | `COMPAT_CANNOT_USE_GENL_NOPS`, `COMPAT_CANNOT_USE_NETLINK_START`, `COMPAT_GENL_HAS_RESV_START_OP` — verify against kernel source |
| `src/compat.h` | All compat macros for the new kernel version |
| `src/uapi/wireguard.h` | Only if upstream UAPI changes (rare) |

### Step-by-step

1. **Build test** — compile against new kernel headers:
   ```bash
   make -C /lib/modules/$(uname -r)/build M=$PWD modules
   ```
   Fix any compile errors from changed kernel APIs.

2. **Verify compat macros** — check `src/compat.h` for each macro:
   - `COMPAT_CANNOT_USE_GENL_NOPS`: In kernels < 4.10, `genl_family` had
     `.ops` as a pointer to a separate `genl_ops` array. Newer kernels embed
     ops in the family struct. If 6.8 changed this, update the guard.
   - `COMPAT_CANNOT_USE_NETLINK_START`: Kernels < 4.13 lack the `.start`
     callback in genl_ops. If removed in a newer kernel, handle differently.
   - `COMPAT_GENL_HAS_RESV_START_OP`: Kernels >= 5.2 require
     `.resv_start_op` in `genl_family`. Verify the value stays at
     `WG_CMD_SET_DEVICE + 1`.

3. **Genl family struct** — if new kernel adds/removes fields in
   `struct genl_family`, update the definition in `netlink.c`:
   ```c
   static struct genl_family genl_family = {
       .ops = genl_ops,
       .n_ops = ARRAY_SIZE(genl_ops),
       .resv_start_op = WG_CMD_SET_DEVICE + 1,
       // ... kernel version may add .netnsok, .parallel_ops, etc.
   };
   ```

4. **NLA policy** — if `nla_policy` type system changes (e.g. NLA_U32
   renamed to NLA_POLICY_MIN or similar), update `device_policy[]`.

5. **Verify runtime** — after loading on the new kernel:
   ```bash
   # Family registered
   sudo awg show wg0

   # Both commands work
   cd src && sudo ./wg-stats-test

   # No kernel errors
   sudo dmesg | grep -i amnezia

   # Cutoff filter works
   sudo ./wg-stats-test 2>&1 | grep -A2 "active last"
   ```

### Common issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `EOPNOTSUPP` for all commands | genl_ops not registered properly | Check `COMPAT_CANNOT_USE_GENL_NOPS` and `.n_ops` count |
| `EOPNOTSUPP` only for new command | `resv_start_op` excludes cmd=3 | Set `.resv_start_op = WG_CMD_SET_DEVICE + 1` |
| Family not found by `awg` | Family registration failed | Check `genl_register_family` return code in dmesg |
| Dump returns 0 peers | Device not found by module | Verify interface created with `type amneziawg`, not `type wireguard` |
| Last handshake always 0 | `__kernel_timespec` layout changed | Check struct packing in kernel headers |
| Cutoff filter always filters everything | `ktime_get_real_seconds()` semantics changed | Verify cutoff comparison logic |

## Whole-codebase update checklist

When making any change to the GET_DEVICE_STATS feature, verify these:

- [ ] `src/uapi/wireguard.h`: `enum wg_cmd` has `WG_CMD_GET_DEVICE_STATS`
- [ ] `src/uapi/wireguard.h`: `enum wgdevice_attribute` has `WGDEVICE_A_HANDSHAKE_CUTOFF`
- [ ] `src/uapi/wireguard.h`: `WGDEVICE_A_MAX` updated to cover new attribute
- [ ] `src/uapi/wireguard.h`: Documentation comment block updated
- [ ] `src/netlink.c`: `device_policy[]` has entry for `WGDEVICE_A_HANDSHAKE_CUTOFF`
- [ ] `src/netlink.c`: `put_peer_stats_attrs()` emits public key + 3 stats
- [ ] `src/netlink.c`: `wg_get_device_stats_dump()` iterates peers, applies cutoff
- [ ] `src/netlink.c`: `genl_ops[]` has entry with `.cmd = WG_CMD_GET_DEVICE_STATS`
- [ ] `src/netlink.c`: `dump_ctx` has `handshake_cutoff` field
- [ ] `GET_DEVICE_STATS.md`: examples and docs match actual behavior
- [ ] `testing.md`: test results updated
- [ ] Builds clean with `make` (no warnings)
- [ ] Loads on target kernel without error
- [ ] `awg show wg0` still works (backward compat)
- [ ] GET_DEVICE + GET_DEVICE_STATS produce same stats for same peer
- [ ] Cutoff filter correctly excludes stale peers
- [ ] `wg-stats-test.c` compiles and passes

## Generic Netlink Family

The module registers a single genl family in `src/netlink.c:1102`:

```c
static struct genl_family genl_family = {
    .ops = genl_ops,
    .n_ops = ARRAY_SIZE(genl_ops),
    .resv_start_op = WG_CMD_SET_DEVICE + 1,  // = 2; cmd 0-1 are "parallel", cmd 2+ are "reserved"
    .name = "amneziawg",
    .version = WG_GENL_VERSION,    // = 2
    .maxattr = WGDEVICE_A_MAX,
    .policy = device_policy,
    .module = THIS_MODULE,
};
```

### Registered operations (genl_ops[])

| cmd | Name | Handler | Type |
|-----|------|---------|------|
| 0 | `WG_CMD_GET_DEVICE` | `wg_get_device_dump` | Dump (start/done) |
| 1 | `WG_CMD_SET_DEVICE` | `wg_set_device` | Do (no dump) |
| 2 | `WG_CMD_UNKNOWN_PEER` | (notify only) | Multicast |
| 3 | `WG_CMD_GET_DEVICE_STATS` | `wg_get_device_stats_dump` | Dump (start/done) |

### Family properties

| Property | Value | Notes |
|----------|-------|-------|
| Name | `"amneziawg"` | Distinct from upstream WireGuard's `"wireguard"` family |
| Version | `2` | Must match `WG_GENL_VERSION` in `uapi/wireguard.h` |
| `resv_start_op` | `2` | Commands 0-1 use "parallel" ops path; 2+ use "reserved" path. Without this, cmd=3 would be rejected with `EOPNOTSUPP` |
| Netns | Not set (default) | Family accessible from all network namespaces (global genl socket) |
| Multicast group | `"auth"` (`WG_MULTICAST_GROUP_AUTH`) | Used for `WG_CMD_UNKNOWN_PEER` notifications |
| Max attr | `WGDEVICE_A_MAX` | Validates all device-level attributes |
| Policy | `device_policy[]` | Type-checking for each `WGDEVICE_A_*` attribute |

### How userspace resolves the family

```c
// libnl
fam = genl_ctrl_resolve(sock, "amneziawg");

// raw netlink
// send CTRL_CMD_GETFAMILY to GENL_ID_CTRL with CTRL_ATTR_FAMILY_NAME="amneziawg"
// response contains CTRL_ATTR_FAMILY_ID (assigned dynamically at module load)
```

The family ID is **dynamically assigned** by the kernel at module load time.
It is not a fixed number. Userspace must always resolve by name.

### Why `resv_start_op` matters for GET_DEVICE_STATS

Without `.resv_start_op = WG_CMD_SET_DEVICE + 1`, the genl core treats
cmd=3 (`GET_DEVICE_STATS`) as belonging to the "parallel" ops array and
may reject it or fail to find a handler. Setting `resv_start_op = 2`
tells the kernel that commands >= 2 use the newer "reserved" lookup path.
This is required on kernels >= 5.2 (controlled by
`COMPAT_GENL_HAS_RESV_START_OP`).

## Compatibility

- **Family name**: `"amneziawg"` — unique, no collision with `"wireguard"`
- **Command IDs**: scoped to the family; even if upstream adds
  `GET_DEVICE_STATS` at numeric value 3, the two modules coexist
  because they are separate Generic Netlink families
- **Existing commands**: `GET_DEVICE` and `SET_DEVICE` are byte-identical
- **Old clients**: continue working against the new module unchanged
