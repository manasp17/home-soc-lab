# Home SOC Lab — Unified Security Dashboard

A home Security Operations Center (SOC) lab built on Splunk Enterprise (free license) that integrates two custom Python security tools into a unified real-time dashboard. Designed to simulate a professional SOC environment for threat detection, network monitoring, and vulnerability management.

---

## Features

- **Real-time network monitoring** — Live ingestion of TCP, UDP, ICMP, and OTHER traffic from a custom packet analyzer
- **Automated vulnerability scanning** — Daily scheduled scans with risk-ranked findings fed directly into Splunk
- **Custom field extractions** — Regex-based parsing of both log formats into structured searchable fields
- **SOC dashboard** — Nine live panels covering packet counts, protocol breakdown, source IPs, alert feed, and CVE-mapped vulnerability findings
- **Port scan detection** — Alerts trigger automatically when a source IP hits 10+ unique ports
- **Risk ranking** — Open ports automatically classified as HIGH, MEDIUM, or LOW based on service type

---

## Why I Built This

SOC analysts spend most of their time in SIEM platforms correlating data from multiple sources. This project was built to understand how that pipeline works end to end — from raw log generation, through ingestion and field extraction, to a live dashboard. Rather than using pre-built data sources, I integrated two Python security tools I had already written from scratch, giving full visibility into every layer of the stack.

---

## Tech Stack

- Python 3.13
- `Scapy` — Live packet capture and protocol analysis
- `Splunk Enterprise (free license)` — Log ingestion, field extraction, dashboarding
- `Windows Task Scheduler` — Automated daily vulnerability scans
- `PCRE2 regex` — Custom field extractions for both log formats
- No paid Splunk add-ons required

---

## Ethical Use

This lab is intended only for use on systems you own or have explicit permission to monitor. All packet capture and port scanning was performed on localhost (127.0.0.1) and a personal home network in a controlled environment. Unauthorized packet capture and port scanning is illegal under the Computer Fraud and Abuse Act and equivalent laws in other jurisdictions.

---

## Architecture

```
Network Packet Analyzer          Vulnerability Scanner
       (Python/Scapy)                  (Python)
            |                               |
            v                               v
      packet_log.txt                scan_report.txt
            |                               |
            +---------------+---------------+
                            |
                   Splunk Enterprise
                (monitored file inputs)
                            |
         Custom Source Types + Field Extractions
                            |
                   SOC Dashboard (localhost:8000)
                            |
           +----------------+----------------+
           |                |                |
    Network Panels      Alert Feed      Vuln Panels
```

---

## Dashboard Panels

**Network monitoring:**
- Live packet count (auto-refresh every 10s)
- Protocol breakdown bar chart (TCP/UDP/ICMP/OTHER)
- Most active source IPs table
- Real-time alert feed (port scans and sensitive port access)
- Traffic timeline with alert spike overlay

**Vulnerability findings:**
- Open port count
- Risk-ranked findings table (HIGH/MEDIUM/LOW)
- High-risk service alerts (SMB, RPC, PostgreSQL, Printer)

---

## Splunk Configuration

### Packet Analyzer Field Extraction
Sourcetype: `packet_analyzer`
```
\[(?P<log_time>[^\]]+)\]\s+\[(?P<alert_type>[^\]]+)\]\s+(?:(?P<protocol>TCP|UDP|ICMP|OTHER)\s+\|\s+(?P<src_ip>[\d.]+)\s+->\s+(?P<dst_ip>[\d.]+)\s+\|\s+Port:\s+(?P<port>\S+)|(?P<alert_msg>(?:Possible|Sensitive).+))
```

Fields extracted: `log_time`, `alert_type`, `protocol`, `src_ip`, `dst_ip`, `port`, `alert_msg`

### Vulnerability Scanner Field Extraction
Sourcetype: `vuln_scanner`
```
\[(?P<scan_time>[^\]]+)\]\s+(?:OPEN\s+\|\s+Port\s+(?P<port>\d+)\s+\|\s+(?P<service>[^|]+?)\s+\|\s+(?P<vuln_notes>.+)|(?P<scan_event>Scan\s+(?:started|complete)[^$]*)|Banner:\s+(?P<banner>.+))
```

Fields extracted: `scan_time`, `port`, `service`, `vuln_notes`, `banner`

### Risk Level Calculated Field
```
if(service="RPC" OR service="SMB", "HIGH", if(service="PostgreSQL" OR service="Printer", "MEDIUM", "LOW"))
```

---

## Example Output

**Packet analyzer log:**
```
[2026-04-03 14:23:01] [INFO]  TCP | 192.168.x.x -> 8.8.8.8 | Port: 443
[2026-04-03 14:23:05] [ALERT] Possible port scan detected from 192.168.x.x
```

**Vulnerability scanner log:**
```
[2026-04-03 16:38:28] OPEN | Port 135  | RPC        | Windows RPC. Common attack vector - restrict externally.
[2026-04-03 16:38:28] OPEN | Port 445  | SMB        | Windows file sharing. Historically exploited (EternalBlue).
[2026-04-03 16:38:28] OPEN | Port 5432 | PostgreSQL  | Database port. Verify it is not exposed externally.
[2026-04-03 16:38:28] OPEN | Port 6463 | Discord     | Discord local RPC server. Normal if Discord is running.
```

---

## Key SPL Searches

**Latest vulnerability findings:**
```
index=main sourcetype=vuln_scanner port=*
| dedup port sortby -_time
| table scan_time port service risk_level vuln_notes
```

**Protocol breakdown:**
```
index=main sourcetype=packet_analyzer protocol=*
| stats count by protocol
```

**Real-time alert feed:**
```
index=main sourcetype=packet_analyzer alert_type=ALERT
| table _time alert_msg src_ip
| sort -_time
| head 20
```

---

## Vulnerability Reference

| Port | Service | Risk Level | Notes |
|------|---------|------------|-------|
| 135 | RPC | High | Common Windows attack vector |
| 445 | SMB | Critical | Exploited by EternalBlue/WannaCry |
| 21 | FTP | High | Credentials sent in plaintext |
| 23 | Telnet | Critical | Unencrypted - replace with SSH immediately |
| 3389 | RDP | High | High value remote access target |
| 4444 | Metasploit | Critical | Common backdoor port |
| 5432 | PostgreSQL | Medium | Database should not be externally exposed |
| 9100 | Printer | Medium | Raw printing port - disable if unused |
| 6463 | Discord | Low | Normal if Discord is running |

---

## Setup

**Requirements:**
- Windows 10/11
- Python 3.x
- Splunk Enterprise free license (500MB/day, runs locally)
- Admin privileges for packet capture (Scapy requires Npcap)

**Steps:**

1. Install Splunk Enterprise and log in at `http://localhost:8000`
2. Configure monitored file inputs pointing to both log files
3. Create custom source types (`packet_analyzer`, `vuln_scanner`)
4. Apply field extractions and calculated field from the config above
5. Import dashboard XML from `/splunk-config/dashboard.xml`
6. Run the packet analyzer in a terminal (runs continuously)
7. Schedule the vulnerability scanner via Windows Task Scheduler (daily)

---

## Demo

To trigger a live port scan alert during a demo:
```bash
nmap 127.0.0.1
```
The alert will appear in the Splunk dashboard alert feed within ~10 seconds.

---

## Future Improvements

- GeoIP mapping of external source IPs on the dashboard
- Slack or email alerting when HIGH risk events are detected
- Automated Splunk saved searches for weekly threat summaries
- CVE lookup integration via NVD API
- Multi-host monitoring across the home network
