# Splunk SIEM Dashboard — Authentication Monitoring

## Objective
Import synthetic authentication events into Splunk, write SPL searches, and build dashboard panels for failed/successful logons, top source IPs, and account activity.

## Data
`data/soc_auth_events.csv` is synthetic training data. It contains normalized fields: `_time`, `event_type`, `event_id`, `username`, `src_ip`, `dest_host`, `status`, `details`.

## Import
1. In Splunk Web, open **Add Data** (or Settings → Add Data).
2. Upload `soc_auth_events.csv`.
3. Choose CSV and review field extraction.
4. Set the sourcetype to `soc_auth_csv` if prompted.
5. Select an index you can access, for example `main` in a lab.
6. Finish, then run `index=main sourcetype=soc_auth_csv` over **All time**.
7. If you used a different index or sourcetype, replace those values in the SPL below.

## SPL panels

### All events
```spl
index=main sourcetype=soc_auth_csv
| table _time event_id username src_ip dest_host status details
| sort - _time
```

### Failed logons over time
```spl
index=main sourcetype=soc_auth_csv status="failure"
| timechart span=1m count as failed_logons
```

### Successful versus failed logons
```spl
index=main sourcetype=soc_auth_csv (status="success" OR status="failure")
| stats count by status
```

### Top source IPs for failures
```spl
index=main sourcetype=soc_auth_csv status="failure" src_ip!="-"
| stats count as failures by src_ip
| sort - failures
```

### Failed logons by account
```spl
index=main sourcetype=soc_auth_csv status="failure"
| stats count as failures by username
| sort - failures
```

### Possible brute-force pattern (threshold 3+)
```spl
index=main sourcetype=soc_auth_csv status="failure" src_ip!="-"
| stats count as failures values(username) as accounts by src_ip
| where failures >= 3
| sort - failures
```

### Successful logon after repeated failures (basic correlation)
```spl
index=main sourcetype=soc_auth_csv
| eval failed=if(status="failure",1,0), success=if(status="success",1,0)
| stats sum(failed) as failures sum(success) as successes values(username) as accounts values(dest_host) as hosts by src_ip
| where failures >= 3 AND successes >= 1
```
This last query groups over the selected time range; it does not prove strict temporal ordering. For time-aware correlation, use a time-windowed transaction or streamstats approach and validate against your Splunk version/data model.

## Build the dashboard
1. Open Dashboards → Create New Dashboard.
2. Name it `SOC Authentication Monitoring`.
3. Add panels using the searches above.
4. Suggested layout: timechart first, then status split and top IPs, then account failures and all-events table.
5. Set the time picker to cover the sample data date, or use All time.
6. Save and capture screenshots for the project portfolio.

## Expected finding
`203.0.113.45` has four synthetic failures and a successful event in the sample. Treat it as a lead for investigation, not proof of attack.

## Troubleshooting
- No results: check index, sourcetype, and time picker.
- Fields missing: inspect a raw event and verify CSV header extraction.
- `src_ip` missing: ensure the CSV was imported with headers as field names.
- `timechart` empty: ensure `_time` was parsed from ISO timestamps and time range includes the sample date.

## Limitations
This is a lab dashboard over synthetic normalized data, not a production SIEM integration. Field names vary across data sources.
