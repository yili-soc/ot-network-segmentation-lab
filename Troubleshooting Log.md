# Troubleshooting Log — OT Network Segmentation & Modbus Threat Detection Lab

Working log of technical issues encountered during setup, kept chronologically. 
This is a raw working document — polished findings get extracted into the final writeup separately.

---

## Phase 0 — Target Environment Setup

### Issue: conpot 0.6.0 incompatible with Python 3.12

**Context:** Initial plan was to use conpot as the OT honeypot target on conpot-vm.

**Problem:** conpot's last stable release (0.6.0) depends on end-of-life packages —
notably `pycrypto` (unmaintained since ~2013, incompatible with modern Python) and 
`enum34` (a backport of the `enum` module that conflicts with Python's built-in 
`enum` since 3.4+). Installing via pip on Python 3.12 failed at import time 
(`ModuleNotFoundError: No module named 'zope.event'`, part of the gevent/monkey-patch chain).

**Attempted fix:** Ran conpot inside Docker via the `honeynet/conpot` image to sidestep 
host Python version issues. Hit a second layer of problems:
- Image's `conpot` binary not on default `$PATH` inside the container
- Correct binary path (`/home/conpot/.local/bin/conpot`) required both `--template` 
  and `--config` arguments together — passing template alone failed
- Config file location required inspecting the container filesystem directly 
  (`find` inside a shell-overridden entrypoint) since the default `testing.cfg` 
  wasn't documented for this exact image
- Running as `--user root` to test a permissions theory instead surfaced a *different* 
  failure: `DistributionNotFound` — the package was installed under the `conpot` user's 
  `$HOME`, invisible to root's separate home directory

**Decision:** After exceeding the project's own Phase 0 time-box, abandoned conpot 
entirely rather than continue debugging a project with no commits in ~7 years. 
Replaced it with a minimal Modbus TCP server built directly on `pymodbus` — a library 
already required elsewhere in the toolchain (traffic generation on analyst-vm), 
actively maintained, and fully compatible with Python 3.12. This also reduces the 
lab's dependency surface: one library serves both the target and the client role, 
instead of two separately-aging codebases.

**Secondary issue (pymodbus API drift):** pymodbus 3.12+ renamed `ModbusSlaveContext` 
→ `ModbusDeviceContext` and changed constructor argument names (`slaves=` → `devices=`). 
Also, `ModbusSequentialDataBlock`'s starting address argument changed validation rules — 
address `0` is no longer valid (raises `TypeError: 0 <= address < 65535` internally, 
since the block subtracts 1 from the given address); starting address must be `1` or higher.

**Resolution:** Minimal working server confirmed listening on TCP 502, validated via 
both `ss -tulnp` and a live `pymodbus` client read (`read_holding_registers`) from a 
separate VM across the network (not loopback).

---

## Phase 0 — FortiGate VM Acquisition

### Issue: Downloaded wrong platform-specific image on first attempt

**Problem:** First download from Fortinet's support portal was 
`FGT_VM64_ALI-v7.6.7.M-build3704-FORTINET` — the `ALI` suffix indicates an 
Alibaba Cloud-specific build, not a generic hypervisor image. File also had no 
recognizable extension (Windows misidentified it as a Wireshark capture file), 
which was a red herring — the real issue was platform selection, not file format.

**Second issue:** Fortinet's standard "Firmware Images" download portal requires an 
active support contract entitlement (returned "Not Available — no product covered by 
a Fortinet support contract"). This is a different access path than the free 
evaluation license flow.

**Resolution:** Correct download path for free evaluation use is 
**support.fortinet.com → Download → VM Images** (not Firmware Images), which does not 
require a support contract. Selected Product: FortiGate, Platform: VMware ESXi, and 
downloaded the "New deployment of FortiGate for VMware" package 
(`FGT_VM64-vX.X.X-buildXXXX-FORTINET.out.ovf.zip` — no cloud-provider suffix in the 
filename). Import into VirtualBox via the `.ovf` file inside the extracted archive.

---

## Phase 0 — Operational Notes (not technical faults, but worth recording)

- Reused two pre-existing VirtualBox VMs from an earlier Log4Shell project instead of 
  building fresh ones — repurposed as conpot-vm (target) and analyst-vm (attacker/tooling), 
  saving significant setup time. Confirmed via `docker inspect ... working_dir` that 
  the previous project's containers were cleanly separable from the new work.
- `sudo` does not inherit user-level pip installs by default; `sudo -E` (preserve 
  environment) required when running scripts installed via `pip install --break-system-packages`, 
  which places packages under `~/.local`, not system-wide.

---

## Phase 1 — FortiGate Clock Drift

### Issue: FortiGate system clock drifted ~1 hour behind actual time

**Problem:** FortiGate's system clock had drifted roughly one hour behind actual time. 
`config system ntp` with `set ntpsync enable` / `set type fortiguard` was configured, 
but did not actually resync — the `last ntp sync` timestamp remained unchanged after 
applying the setting.

**Root cause:** port1/port2 are bound to isolated Host-only VMnets with no route to the 
internet by design (the lab's segmented network intentionally has no external access), 
so FortiGuard's NTP servers were unreachable. Confirmed via `execute ping 8.8.8.8` 
returning no response.

**Resolution:** Corrected the clock manually via `execute date` / `execute time` rather 
than relying on NTP, since the isolated network is a deliberate design choice, not a 
misconfiguration to fix.

### Issue: conpot-vm unreachable on port 502 — traced to missing default route

**Problem:** After a reboot, analyst-vm could no longer reach conpot-vm on TCP/502. 
FortiGate logs showed the SYN being forwarded cleanly, and conpot-vm's own NIC 
confirmed receiving it (via `tcpdump`) — but no reply ever came back, not even a RST.

**Root cause:** Earlier `ip addr add` / `ip route add` commands were runtime-only and 
never persisted to netplan, so the reboot reverted both VMs' networking. conpot-vm's 
netplan had a static IP but no default route — it could receive packets from a 
different subnet but had no path to reply, so the kernel silently dropped the response 
before it ever reached the TCP stack. This looks identical to a firewall drop (pure 
timeout, no RST) but isn't one.

Also found and fixed during this check: conpot-vm still had a leftover second NIC 
bridged to the IT segment (a VirtualBox migration artifact) that would have let it 
bypass FortiGate entirely — removed. VMnet1's DHCP service was also still enabled, 
contradicting the static-only design — disabled.

**Resolution:** Rewrote both VMs' netplan configs with persistent static IPs and 
explicit default routes, then validated across a full reboot of both VMs (not just a 
live test) before considering it fixed.

**Takeaway:** `ip addr add`/`ip route add` don't survive a reboot — only netplan does. 
A missing return route looks exactly like a firewall drop (timeout, no RST) unless you 
check the route table.

---

*Log will be updated as Phase 1–4 proceed.*
