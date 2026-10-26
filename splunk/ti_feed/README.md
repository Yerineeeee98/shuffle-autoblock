# Splunk: TI feed auto-block alert

For use with the **TA-cip-ti-feed** app. This alert takes the critical / high TI matches of the last hour
(from the app's index, `cip_ti_index`), checks the firewall threat logs
(IPS, antivirus, web filter) for the same IP, and sends the IPs that meet the block conditions to the
Shuffle TI workflow, which adds them to the FortiGate block group.

| File | What it is |
|---|---|
| `macros.conf` | 4 macros (`cip_ab_logs`, `cip_ab_normalize`, `cip_ab_dhcp_logs`, `cip_ab_health`) |
| `ti_autoblock_alert.spl` | The alert |
| `health_check.spl` | Scheduled report that checks the logs the alert reads |

`cip_ti_traffic_all`, `cip_ti_normalize`, `cip_ti_index`, `cip_ti_severity` and the lookup
`cip_ti_alert_scope` come with the TA-cip-ti-feed app.

## 1. Macros

Add the 4 macros in `macros.conf` to the TA-cip-ti-feed app.
Change the index and sourcetype in `cip_ab_dhcp_logs` to your own DHCP logs.

1. Settings > Advanced search > Search macros > New Search Macro
2. Destination app: TA-cip-ti-feed
3. Name: the name in brackets in macros.conf (for example cip_ab_logs)
4. Definition: the text after definition = . Leave Arguments empty and "Use eval-based definition" unchecked
5. Save, then in the macro list set Permissions to This app only (or All apps) so the alert can use it

## 2. Alert settings

Create the alert in the TA-cip-ti-feed app (it uses the app's macros and lookup).

1. Open the TA-cip-ti-feed app > **Search**, and paste the content of `ti_autoblock_alert.spl`
2. **Save As** > **Alert**, and enter a title (for example `ti_autoblock_alert`)
3. Set the values in the table below
4. **Add Actions** > **Webhook**, and enter the Webhook URL of the Shuffle TI workflow
5. **Save**

| Setting | Value |
|---|---|
| Alert type | Scheduled, run on cron schedule |
| Cron expression | `5,15,25,35,45,55 * * * *` |
| Time range | Last 82 minutes (the search also sets its own `earliest`) |
| Expires | 24 hours |
| Trigger alert when | Number of results is greater than 0 |
| Trigger | Once |
| Throttle | Off |
| Trigger action | Webhook, URL = the Webhook URL of the Shuffle TI workflow |

## 3. Block decision

1. TI matches of the last hour with severity **critical** or **high**, in the alert scope of
   `cip_ti_alert_scope` (direction, firewall action, minimum confidence). Matches the app marks with `ti_self_egress=1` are skipped.
2. For each match, the firewall threat logs in the same 10-minute window are checked for 3 conditions,
   in the direction toward the TI IP:
   - **IPS attack** (IPS log, severity critical or high)
   - **Antivirus detection**
   - **Phishing and malicious web** (web filter: malicious websites or phishing; outbound only)
3. Decision:

| Severity | Direction | Auto-block when |
|---|---|---|
| critical | any | 1 or more conditions met |
| high | outbound | all 3 conditions met |
| high | inbound | IPS attack and antivirus detection |
| any | any | never, if some windows could not be linked to logs (`Investigation incomplete`) |

Everything else is `recommend` (recorded, not blocked).

Every candidate is saved to `index=summary source="ti_autoblock_alert"` (the `collect` line).
Only the `auto_block` rows are sent to Shuffle.

## 4. What Shuffle needs from each result row

| Field | Meaning |
|---|---|
| `match_ip`, `ti_direction`, `severity` | TI IP, direction (inbound / outbound), TI severity |
| `decision`, `decision_reason` | `auto_block` and which conditions were met |
| `block_ip`, `block_name`, `block_comment` | Values for the FortiGate address object (`SOAR_Deny_TI_<IP>`) |
| `detection_details`, `raw_logs` | Evidence lines and raw firewall logs (for the AI summary and Slack) |
| `fw_devices`, `src_hosts`, `dhcp_logs` | Firewall names, internal IP = PC name, DHCP logs |

## 5. Health check (scheduled report)

1. Open the TA-cip-ti-feed app > **Search**, and paste the content of `health_check.spl`
2. **Save As** > **Report**, and enter a title
3. In the report, **Edit** > **Edit Schedule**, and set the values below

| Setting | Value |
|---|---|
| Schedule | Cron `8 * * * *` (every hour at minute 8) |
| Time range | Last 24 hours (the search itself reads the last 15 minutes) |
| Schedule window | No window |

No setting is needed to save the result: the last line of the search
(`| collect index=summary source="ti_autoblock_health"`) saves it.

It writes one row with a verdict (`OK`, `Warning: ...`, `Problem: ...`) that names the macro to check.
