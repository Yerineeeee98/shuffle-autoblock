# Splunk: dashboards

Dashboard Studio dashboards that read the records the alerts and the Shuffle workflows save to `index=summary`.

| File | Dashboard | Records it reads (`index=summary`) |
|---|---|---|
| `scenario_autoblock_status.json` | Scenario Auto-Block Status | `scen_autoblock_eval`, `scen_autoblock_fg`, `scen_autoblock_health` |
| `ti_autoblock_status.json` | TI Auto-Block Status | `ti_autoblock_alert`, `ti_autoblock_fg`, `ti_autoblock_health` |

Both have 3 tabs: **Overview** (blocked / recommended IPs and history), **Health** (are the alerts and logs working),
**Setup** (are the logs, alerts and records the auto-block needs in place).

## How to add

1. Dashboards > Create New Dashboard > Dashboard Studio (Absolute layout)
2. Edit > Source
3. Replace everything with the content of the JSON file, then Save

Create each dashboard in the app that has its macros: `scenario_autoblock_status.json` where the `cip_sc_*` macros are,
`ti_autoblock_status.json` in the TA-cip-ti-feed app (it uses the `cip_ab_*` and `cip_ti_*` macros).
