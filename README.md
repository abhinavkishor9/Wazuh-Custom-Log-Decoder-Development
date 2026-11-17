# Wazuh Custom Log Decoder Development

## Overview

This lab demonstrates how to develop and implement a custom Wazuh decoder for application authentication logs. The objective was to transform raw application log messages into structured security events and generate alerts using a custom detection rule.

**Environment:** Rocky Linux 9.8  
**Wazuh Version:** 4.13.1  
**Log Source:** `/var/log/custom-app/auth.log`

## Objectives

- Understand Wazuh decoder architecture.
- Create a custom decoder for application authentication events.
- Extract username, source IP and failure reason.
- Test decoder functionality using `wazuh-logtest`.
- Develop a custom detection rule.
- Configure Wazuh to monitor a custom application log.
- Verify alert generation through Wazuh alert logs.

## Scenario

A Linux application generates authentication failure messages that are not part of a standard application log format.

The objective was to configure Wazuh to collect these messages, decode their contents, and generate a security alert when an authentication failure event is detected.

### Sample Log

```text
AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password
```

## Lab Implementation

### 1. Custom Log Creation

Created a dedicated application log directory:

```bash
mkdir -p /var/log/custom-app
```

Created the test log:

```bash
echo 'AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password' >> /var/log/custom-app/auth.log
```

### 2. Custom Decoder Development

Modified:

```text
/var/ossec/etc/decoders/local_decoder.xml
```

Custom decoder:

```xml
<decoder name="custom-auth">
    <prematch>AUTH_FAILED</prematch>
    <regex>AUTH_FAILED user=(\S+) source_ip=(\S+) reason=(\S+)</regex>
    <order>user,srcip,extra_data</order>
</decoder>
```

The decoder extracts:

| Field | Extracted value |
|---|---|
| Decoder name | custom-auth |
| User | admin |
| Source IP | 192.168.1.50 |
| Extra data | invalid_password |

Wazuh maps the `user` field to `dstuser`.

### 3. Decoder Testing

Tested the custom decoder using:

```bash
/var/ossec/bin/wazuh-logtest
```

Successful Phase 2 output:

```text
name: 'custom-auth'
dstuser: 'admin'
extra_data: 'invalid_password'
srcip: '192.168.1.50'
```

This confirmed that the custom decoder successfully extracted the expected fields.

### 4. Custom Detection Rule

Modified:

```text
/var/ossec/etc/rules/local_rules.xml
```

Custom rule:

```xml
<rule id="100200" level="5">
    <decoded_as>custom-auth</decoded_as>
    <description>Custom Application Authentication Failed</description>
    <group>authentication_failed,custom_app,</group>
</rule>
```

Successful Phase 3 output:

```text
id: '100200'
level: '5'
description: 'Custom Application Authentication Failed'
groups: '['local', 'authentication_failed', 'custom_app']'
**Alert to be generated.
```

### 5. Custom Log Collection

Configured Wazuh to monitor the application log in:

```text
/var/ossec/etc/ossec.conf
```

Configuration:

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

Verified service status:

```bash
systemctl is-active wazuh-manager
```

Result:

```text
active
```

### 6. Alert Verification

Generated a fresh authentication failure event:

```bash
echo 'AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password' >> /var/log/custom-app/auth.log
```

Checked Wazuh log collection:

```bash
grep -i 'custom-app/auth.log' /var/ossec/logs/ossec.log | tail -10
```

Observed:

```text
wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/custom-app/auth.log'.
```

Checked generated alerts:

```bash
grep -E '"id":"100200"|custom-auth' /var/ossec/logs/alerts/alerts.json | tail -10
```

Confirmed alert:

```json
{
  "rule": {
    "level": 5,
    "description": "Custom Application Authentication Failed",
    "id": "100200"
  },
  "decoder": {
    "name": "custom-auth"
  },
  "data": {
    "srcip": "192.168.1.50",
    "dstuser": "admin",
    "extra_data": "invalid_password"
  },
  "location": "/var/log/custom-app/auth.log"
}
```

## Results

| Component | Result |
|---|---|
| Custom application log | Created |
| Decoder development | Successful |
| Field extraction | Successful |
| Decoder testing | Successful |
| Custom detection rule | Successful |
| Log collection configuration | Successful |
| Wazuh Manager restart | Successful |
| Alert generation | Confirmed |
| Dashboard indexing | Not independently verified |

## Key Learnings

- Decoders extract structured fields from raw log messages.
- Rules evaluate decoded events and generate alerts.
- Wazuh supports custom application log monitoring through `localfile`.
- `wazuh-logtest` helps validate decoder and rule behavior.
- Wazuh static decoder fields must use supported field names.
- Successful alert generation in `alerts.json` does not independently confirm dashboard indexing.

## Conclusion

Successfully developed and tested a custom Wazuh decoder and detection rule for application authentication failure logs.

Wazuh collected the custom log, extracted the expected fields, and generated a Level 5 alert with rule ID `100200`.

The lab demonstrated the complete workflow of custom log collection, decoding, rule development and alert verification.
