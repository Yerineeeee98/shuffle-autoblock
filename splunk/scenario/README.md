# Splunk: 5 scenario alerts

Each alert finds internal PCs that did something suspicious, collects the public IPs the PC talked to
in the 5 minutes before, and sends the result to the Shuffle workflow. Shuffle looks up the public IPs
in Criminal IP and blocks the risky ones on FortiGate.

| File | Scenario (`scen_id`) | Trigger |
|---|---|---|
| `1_risky_port.spl` | Risky port access (`outbound_riskport`) | Connection to a public IP on port 22, 445, 1104, 3306 or 3389 |
| `2_non_standard_port.spl` | Non-standard port access (`outbound_nonstd_port`) | TCP/UDP to a public IP on a port that is not 80, 443 or a risky port. Only public IPs with 3+ sessions are sent to Shuffle |
| `3_phishing.spl` | Phishing / malicious link click (`phishing_click`) | Web filter block, or antivirus detection with "phish" or "redir" in its name |
| `4_malware.spl` | Malware infection (`malware_outbound`) | Defender malware alert, or firewall antivirus detection |
| `5_admin_login.spl` | Firewall admin login (`login_outbound`) | Admin login (see the login rules below) |
| `scen_health_check.spl` | (scheduled report) | Checks that the logs and fields the alerts need are coming in |

## 1. Macros

Add the 6 macros in `macros.conf` first (Settings > Advanced search > Search macros), in the app where you will create the alerts.
Change the index and sourcetype in `cip_sc_traffic`, `cip_sc_threat_logs`, `cip_sc_dhcp` and
`cip_sc_defender_alerts` to your own. The alerts use only these macros, so the SPL files do not need changes.

## 2. Alert settings (same for all 5)

Create each alert in the same app as the macros (the alerts can only use macros they can see).

1. Open the app > **Search**, and paste the content of one `.spl` file
2. **Save As** > **Alert**, and enter a title (for example `scen_1_risky_port`)
3. Set the values in the table below
4. **Add Actions** > **Webhook**, and enter the Webhook URL of the Shuffle workflow
5. **Save**, then repeat for the other 4 files

| Setting | Value |
|---|---|
| Alert type | Scheduled, run on cron schedule |
| Cron expression | `*/10 * * * *` |
| Time range | Not used (each search sets its own `earliest` / `latest`) |
| Expires | 24 hours |
| Trigger alert when | Number of results is greater than 0 |
| Trigger | Once |
| Throttle | Off |
| Trigger action | Webhook, URL = the Webhook URL of the Shuffle workflow |

The searches read the last 15 minutes (`-16m@m` to `-1m@m`) and trigger only on events of the last
10 minutes, so with a 10-minute schedule every event is checked once.

## 3. What Shuffle needs from each result row

| Field | Meaning |
|---|---|
| `scen_id`, `scen_name` | Scenario |
| `srcip`, `srcname`, `dhcp_name` | Internal PC (IP, name from the log, name from DHCP) |
| `dstip`, `dstport`, `proto` | Public IP the PC talked to |
| `sess`, `ip_sess` | Sessions on this port / to this public IP (all ports) |
| `cond1` | `met` when the IP can be auto-blocked (3+ sessions, or the login rule in scenario 5) |
| `hit_time`, `evt_time` | Event time |
| `trig_raw`, `ip_raw` | Raw logs for the AI summary (trigger log, and up to 2 logs to the public IP) |
| Scenario fields | e.g. `hit_port`, `blocked_host`, `mw_threat`, `login_user`, `cond_reason` |

Shuffle auto-blocks an IP only when `cond1=met` **and** Criminal IP finds it risky.
Alert 2 (non-standard port) sends only the `cond1=met` rows (only these can be auto-blocked, so only these go to Criminal IP).
The other alerts send every public IP, and the IPs without `cond1=met` are saved as recommendations.

**Login rules (scenario 5)**
- Rule 1: 3+ failures then a success, from the same PC, account and firewall, within 10 minutes
- Rule 2: 3+ failures from the same PC on 2+ firewalls within 10 minutes

## 4. Health check (scheduled report)

1. Open the same app > **Search**, and paste the content of `scen_health_check.spl`
2. **Save As** > **Report**, and enter a title
3. In the report, **Edit** > **Edit Schedule**, and set the values below

| Setting | Value |
|---|---|
| Schedule | Cron `37 * * * *` (every hour at minute 37) |
| Time range | Last 24 hours (the search sets its own `earliest` for each part) |
| Schedule window | No window |

No setting is needed to save the result: the last line of the search
(`| collect index=summary source="scen_autoblock_health"`) saves it.

It checks the logs the alerts read (traffic: last 15 minutes, security logs: 24 hours, DHCP: 24 hours,
Defender: 7 days) and writes one row with a verdict (`OK`, `Warning: ...`, `Problem: ...`).
The verdict names the macro to check.
