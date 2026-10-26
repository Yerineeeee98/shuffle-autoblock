# Criminal IP + Shuffle auto-block

Splunk files for two Shuffle workflows that check outside IPs with Criminal IP and block the risky ones on FortiGate.

| Workflow | Started by | What it does |
|---|---|---|
| Scenario auto-block | 5 Splunk alerts (risky port, non-standard port, phishing, malware, firewall admin login) | Looks up the public IPs a suspicious PC talked to in Criminal IP, judges them with 6 indicators, and blocks the risky ones |
| TI feed auto-block | 1 Splunk alert (TA-cip-ti-feed app) | Blocks critical / high TI feed IPs that also show up in the firewall threat logs (IPS, antivirus, web filter) |

Both workflows add the IP to a FortiGate address group, send a Slack alert with an AI summary of the raw logs,
and save the result back to Splunk (`index=summary`).

## Files

| Folder | Contents |
|---|---|
| [`splunk/scenario`](splunk/scenario) | Macros, 5 alert searches and a health check report for the scenario workflow |
| [`splunk/ti_feed`](splunk/ti_feed) | Macros, the alert search and a health check report for the TI feed workflow |
| [`splunk/dashboards`](splunk/dashboards) | Dashboard Studio dashboards for both workflows |

Each folder has a README with the settings.
