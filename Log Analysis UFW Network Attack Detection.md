# Log Analysis Basics – Network-Based Attack Detection Using UFW
# Name: Comfort Ozohu Roberts
# Course: Advanced Cybersecurity Class
# Date: 21 October 2026

## Objective
Simulate a network port scan between two virtual machines and demonstrate how Uncomplicated Firewall (UFW) logging can be used to detect and block early-stage reconnaissance activity.

## Lab Environment

| Role | Machine | IP Address |
|------|---------|------------|
| Target (defender) | Kali Linux | 192.168.59.128 |
| Attacker (scanner) | Metasploitable2 | 192.168.59.129 |

Both machines were run as VMware virtual machines on the same host-only/NAT network segment, allowing direct communication between them.

## Step 1: Simulate a Port Scan

From the attacker machine (Metasploitable2), an Nmap scan was run against the target's HTTP port:

```bash
nmap -PN -p 80 192.168.59.128
```

The `-PN` flag skips host discovery (ping) and forces Nmap to probe the specified port directly, which is useful in lab environments where ICMP may be filtered.

## Step 2: Enable UFW and Logging on the Target

On the Kali (target) machine, UFW was installed, enabled, and logging was turned on:

```bash
sudo apt install ufw
sudo ufw enable
sudo ufw logging on
sudo ufw logging high
```

## Step 3: Add a Firewall Rule to Block the Attacker

A rule was added to explicitly deny and log TCP traffic from the attacker's IP address on port 80:

```bash
sudo ufw deny log from 192.168.59.129 to any port 80 proto tcp
sudo ufw reload
```

**Note:** An earlier broad rule (`ALLOW IN from 192.168.59.0/24`) was found to be permitting all subnet traffic, including the attacker's scan, before the deny rule could take effect. This rule was removed so the specific deny rule would be evaluated and enforced correctly:

```bash
sudo ufw delete <rule number for Anywhere ALLOW IN 192.168.59.0/24>
```

## Step 4: Monitor and Capture the Blocked Traffic

The UFW log was monitored for entries referencing the attacker's IP address:

```bash
sudo tail -f /var/log/ufw.log | grep "192.168.59.129"
```

Confirmation that the rule was active at the packet-filtering level was also checked directly in iptables:

```bash
sudo iptables -L ufw-user-input -n -v --line-numbers | grep -i 59.129
```

## Result: Blocked Scan Log Entry

After re-running the scan, UFW successfully logged and blocked the connection attempt:

```
2026-10-20T07:10:02.287530-04:00 kali kernel: [UFW BLOCK] IN=eth0 OUT=
MAC=00:0c:29:f9:b4:c1:00:0c:29:c1:38:0e:08:00
SRC=192.168.59.129 DST=192.168.59.128 LEN=44 TOS=0x00 PREC=0x00 TTL=44
ID=47965 PROTO=TCP SPT=63503 DPT=80 WINDOW=1024 RES=0x00 SYN URGP=0
```

**Key details:**
- **Source IP (attacker):** 192.168.59.129
- **Targeted port:** 80 (DPT=80)
- **Timestamp:** 2026-10-20T07:10:02.287530-04:00
- **Action:** BLOCK

*(See attached screenshot: `ufw-block-log.png`)*

## Troubleshooting Notes

During the lab, the deny rule initially blocked the traffic at the packet-filtering level (confirmed via iptables packet counters) but did not generate a log entry. This was because a plain `ufw deny` rule does not log by default — only the `log` keyword on the rule, or the default-policy drop, triggers a logged entry. Adding `log` to the rule (`ufw deny log from ...`) resolved this and produced visible `[UFW BLOCK]` entries.

## Conclusion

- The `ufw.log` file is an effective tool for identifying early-stage network reconnaissance when firewall logging is properly enabled and scoped.
- A port scan against a specific service (in this case, HTTP on port 80) is a common early indicator that an attacker is mapping a system's network exposure.
- Firewall rule *order* matters: a broad allow rule placed before a specific deny rule can silently permit traffic that should be blocked, with no corresponding log entry. Reviewing both the rule set (`ufw status numbered`) and the packet counters (`iptables -L ... -v`) helped identify this misconfiguration.
- Reviewing logs and applying correctly ordered, explicitly logged firewall rules allows defenders to detect and block unauthorized scanning activity, and to produce auditable evidence of that detection.
