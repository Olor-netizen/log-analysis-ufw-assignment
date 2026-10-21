# Log Analysis Basics: Network-Based Attack Detection Using UFW

**Student:** Abass Adam TEMITOPE
**Course:** Advanced Cybersecurity Class
**Tutor:** Mr. Oluwasola
**Date:** 21 October 2026
**GitHub:** @Olor-netizen

---

## 1. Objective

Simulate a network-based port scan and detect it using UFW (Uncomplicated Firewall) logs on a Linux system. A port scan was launched with Nmap, blocked by a UFW rule, and the blocked traffic was found in the firewall logs.

## 2. Lab Environment

| Role | System | IP address | Notes |
| :--- | :--- | :--- | :--- |
| Attacker | Windows (host machine) | 192.168.159.1 | VMware VMnet8 adapter, Nmap installed |
| Target | Kali Linux (VMware VM) | 192.168.159.128 | UFW and rsyslog installed |

The lab handout suggests Kali as the attacker and Ubuntu as the target. I did not have an Ubuntu machine, so I used the Kali VM as the target (UFW works the same on Debian-based systems) and the Windows host as the attacker. Both machines belong to me, and no other system was scanned.

## 3. Tools Used

- **UFW:** firewall rules and logging
- **rsyslog:** writes UFW messages to `/var/log/ufw.log`
- **Nmap:** port scan from the attacker machine
- **grep, awk, tail, journalctl:** log filtering

## 4. Steps Performed

### Step 1: Install UFW, enable it and turn on logging (Kali target)

```bash
sudo apt update
sudo apt install ufw rsyslog -y
sudo systemctl enable --now rsyslog
sudo systemctl restart rsyslog
sudo ufw enable
sudo ufw logging on
sudo ufw logging high
```

### Step 2: Deny HTTP traffic from the attacker (Kali target)

```bash
sudo ufw deny from 192.168.159.1 to any port 80 proto tcp
sudo ufw reload
sudo ufw status numbered
```

### Step 3: Monitor the log (Kali target)

```bash
sudo tail -F /var/log/ufw.log | grep --line-buffered "UFW BLOCK" | grep "SRC=192.168.159.1 "
```

### Step 4: Run the port scan (Windows attacker)

```powershell
nmap -p 80 192.168.159.128
```

Nmap reported port 80 as `filtered`, which means the firewall dropped the probes without replying.

### Step 5: Find the evidence in the logs (Kali target)

```bash
sudo grep "UFW BLOCK" /var/log/ufw.log | grep "SRC=192.168.159.1 " | tail
```

## 5. Evidence

Blocked entries from `/var/log/ufw.log`:

```
2026-10-21T10:13:32.693146+01:00 kali kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00:0c:29:1e:9c:5c:00:50:56:c0:00:08:08:00 SRC=192.168.159.1 DST=192.168.159.128 LEN=44 TOS=0x00 PREC=0x00 TTL=52 ID=22334 PROTO=TCP SPT=37702 DPT=80 WINDOW=1024 RES=0x00 SYN URGP=0
2026-10-21T10:13:32.805177+01:00 kali kernel: [UFW BLOCK] IN=eth0 OUT= MAC=00:0c:29:1e:9c:5c:00:50:56:c0:00:08:08:00 SRC=192.168.159.1 DST=192.168.159.128 LEN=44 TOS=0x00 PREC=0x00 TTL=55 ID=28727 PROTO=TCP SPT=37703 DPT=80 WINDOW=1024 RES=0x00 SYN URGP=0
```

The same events also appear in the system journal (`sudo journalctl -k | grep "UFW BLOCK"`).

Short view with only the key fields:

```bash
sudo grep "UFW BLOCK" /var/log/ufw.log | grep "SRC=192.168.159.1 " | awk '{printf "%s ", $1; for(i=1;i<=NF;i++) if ($i ~ /^(SRC|DST|PROTO|DPT)=/) printf "%s ", $i; print ""}' | tail
```

```
2026-10-21T10:13:32.693146+01:00 SRC=192.168.159.1 DST=192.168.159.128 PROTO=TCP DPT=80
2026-10-21T10:13:32.805177+01:00 SRC=192.168.159.1 DST=192.168.159.128 PROTO=TCP DPT=80
```

## 6. Findings

| Item | Value |
| :--- | :--- |
| Source IP of the scan | 192.168.159.1 |
| Destination IP | 192.168.159.128 |
| Targeted port | 80/tcp (HTTP) |
| Timestamp | 2026-10-21T10:13:32.693146+01:00 |
| Firewall action | `[UFW BLOCK]` |
| Log file | `/var/log/ufw.log` (also `journalctl -k`) |

### What the log fields show

- `SRC=192.168.159.1`: the machine that sent the scan (the Windows attacker).
- `DST=192.168.159.128`: the protected machine (Kali target).
- `DPT=80`: the targeted port, a web service port that scanners commonly probe.
- `PROTO=TCP` with `SYN`: a connection attempt that was never completed, which is how an Nmap SYN scan works.
- `SPT=37702` and `SPT=37703`: different source ports about 0.1 seconds apart, which are the scan probe and its retry.
- `[UFW BLOCK]`: the firewall enforced the deny rule and logged the packet.

## 7. Screenshots
![UFW rule list](ufw%20status.jpeg)

![Nmap scan showing the port as filtered](nmap%20filtered%20ping.png)

![UFW BLOCK log entry and clean extraction](Clean%20Extraction.jpeg)

## 8. Mitigation and Recommendations

- Keep the source blocked and keep monitoring for further scans from the same IP.
- Search the logs for other ports probed by the same source: `sudo grep "SRC=192.168.159.1 " /var/log/ufw.log`.
- Escalate to the next tier with the indicators: source IP, destination IP, port and timestamp.
- Forward firewall logs to a central log system (SIEM) for correlation.
- When the investigation is finished, list the rules with `sudo ufw status numbered` and remove the rule with `sudo ufw delete <rule number>`.
- `ufw logging high` produces a lot of data. After the lab, `sudo ufw logging low` reduces it.

## 9. Conclusion

A port scan from 192.168.159.1 against port 80 on 192.168.159.128 was blocked by UFW and recorded in `/var/log/ufw.log`. The log entries show the source IP, the targeted port and the timestamp. Firewall logging made it possible to detect early network reconnaissance, and the deny rule blocked it.

## 10. References

- UFW manual: `man ufw`
- Nmap documentation: https://nmap.org
- Course lab handout: Log Analysis Basics, Network-Based Attack Detection Using UFW
