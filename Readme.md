# OT Network Segmentation & Modbus Threat Detection — Hands-On Lab

Hands-on lab demonstrating IT-OT network segmentation, OT protocol traffic analysis, 
firewall policy configuration, and Splunk detection engineering.

## Goals

- Deploy a segmented lab: FortiGate firewall between an "IT" analyst segment and an "OT" target segment
- Configure and validate a firewall policy enforcing Modbus-only traffic across the boundary
- Generate and analyze legitimate vs. anomalous Modbus traffic (Wireshark)
- Build and validate Splunk detection rules against the captured traffic
- Produce a writeup documenting the process, findings, and limitations

## Stack

- FortiGate VM (evaluation license)
- Modbus target (pymodbus-based server)
- Analyst/attacker tooling: pymodbus, nmap, Metasploit, tcpdump
- Splunk (detection engineering)

## Status

In progress — see `docs/troubleshooting-log.md` for setup notes.

## Structure

```
docs/
  troubleshooting-log.md   — working log of setup issues and fixes
```
