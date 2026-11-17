# Troubleshooting Notes – Wazuh Custom Log Decoder Development

## Environment

- Rocky Linux 9.8
- Wazuh 4.13.1
- Custom application log: `/var/log/custom-app/auth.log`

## Issue 1 – Initial Decoder Did Not Match

### Observation

The initial test included a timestamp:

```text
2026-11-16 09:30:15 AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password
```

Phase 2 output:

```text
name: 'windows-date-format'
```

The custom decoder did not match.

### Troubleshooting

The timestamp-prefixed input was not successfully parsed by the custom decoder configuration.

The test was simplified by removing the timestamp:

```text
AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password
```

The custom decoder then matched successfully.

### Learning

Test the exact log format expected by the decoder. A timestamp-prefixed event may require separate handling of the header and log body.

---

## Issue 2 – Incorrect Decoder Field Mapping

### Initial Configuration

```xml
<order>user, source_ip, reason</order>
```

### Problem

The configuration used field names that were not supported as Wazuh static decoder fields.

### Corrected Configuration

```xml
<order>user,srcip,extra_data</order>
```

### Result

```text
dstuser: 'admin'
srcip: '192.168.1.50'
extra_data: 'invalid_password'
```

### Learning

Use supported Wazuh decoder field names.

The `user` field is represented as `dstuser` in the output. The `srcip` field extracts the source IP, and `extra_data` stores the failure reason.

---

## Issue 3 – Custom Rule Not Initially Matching

### Observation

After successful decoding, Phase 3 initially returned generic rule `1002`.

### Troubleshooting

A custom detection rule was added to:

```text
/var/ossec/etc/rules/local_rules.xml
```

### Corrected Rule

```xml
<rule id="100200" level="5">
    <decoded_as>custom-auth</decoded_as>
    <description>Custom Application Authentication Failed</description>
    <group>authentication_failed,custom_app,</group>
</rule>
```

### Result

```text
id: '100200'
level: '5'
description: 'Custom Application Authentication Failed'
**Alert to be generated.
```

### Learning

A successful decoder match does not automatically generate a custom alert. A matching detection rule is required.

---

## Issue 4 – No Results in Wazuh Discover

### Observation

The following queries initially returned no results:

```text
rule.id: "100200"
```

```text
decoder.name: "custom-auth"
```

### Investigation

The custom application log had not initially been configured as a monitored log source.

### Resolution

Added the following configuration to:

```text
/var/ossec/etc/ossec.conf
```

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/custom-app/auth.log</location>
</localfile>
```

Restarted the Wazuh Manager:

```bash
systemctl restart wazuh-manager
```

Verified:

```bash
systemctl is-active wazuh-manager
```

Output:

```text
active
```

### Learning

`wazuh-logtest` validates decoder and rule behavior but does not itself send events into the alert index.

A log source must be configured and collected before the event can generate a normal Wazuh alert.

---

## Issue 5 – Verify Alert Generation Locally

### Verification Command

```bash
grep -E '"id":"100200"|custom-auth' /var/ossec/logs/alerts/alerts.json | tail -10
```

### Confirmed Result

The alert was found with:

- Rule ID: `100200`
- Level: `5`
- Decoder: `custom-auth`
- Username: `admin`
- Source IP: `192.168.1.50`
- Extra data: `invalid_password`

### Learning

The local `alerts.json` file provides direct evidence that the manager generated the alert.

Dashboard visibility requires successful downstream indexing and should be verified separately.

---

## Troubleshooting Summary

| Issue | Resolution |
|---|---|
| Timestamp-prefixed test did not match | Tested without timestamp |
| Unsupported decoder field names | Used `user,srcip,extra_data` |
| Generic rule 1002 | Created custom rule 100200 |
| Custom file not monitored | Added localfile configuration |
| No initial Discover results | Verified collection and local alert generation |

## Final Status

Custom decoder: Successful

Field extraction: Successful

Custom rule: Successful

Log collection: Confirmed

Alert generation: Confirmed

Dashboard indexing: Not independently verified
