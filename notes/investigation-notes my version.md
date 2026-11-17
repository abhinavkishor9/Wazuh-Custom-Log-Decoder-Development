# Investigation Notes 

## 1. Investigation Overview

**Platform:** Rocky Linux 9.8  
**Wazuh Version:** 4.13.1  
**Investigation Type:** Custom Log Parsing and Detection Engineering  
**Log Source:** `/var/log/custom-app/auth.log`

### Objective

Develop a custom Wazuh decoder to parse application authentication failure messages and generate a corresponding detection alert.

## 2. Test Scenario

A controlled authentication failure event was created:

```text
AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password
```

The event contained:

- Event type: AUTH_FAILED
- Username: admin
- Source IP: 192.168.1.50
- Failure reason: invalid_password

This was a controlled lab event, not evidence of an actual unauthorized login.

## 3. Decoder Investigation

### Configuration File

```text
/var/ossec/etc/decoders/local_decoder.xml
```

### Custom Decoder

```xml
<decoder name="custom-auth">
    <prematch>AUTH_FAILED</prematch>
    <regex>AUTH_FAILED user=(\S+) source_ip=(\S+) reason=(\S+)</regex>
    <order>user,srcip,extra_data</order>
</decoder>
```

### Decoder Test Results

Tool:

```bash
/var/ossec/bin/wazuh-logtest
```

Successful Phase 2:

```text
name: 'custom-auth'
dstuser: 'admin'
extra_data: 'invalid_password'
srcip: '192.168.1.50'
```

### Analysis

The decoder successfully identified the custom authentication event and extracted the expected values.

The `user` decoder field was represented as `dstuser` in the output.

The initial decoder configuration used unsupported field names in the extraction order. Updating the order to `user,srcip,extra_data` resolved the field mapping.

## 4. Detection Rule Investigation

### Configuration File

```text
/var/ossec/etc/rules/local_rules.xml
```

### Custom Rule

```xml
<rule id="100200" level="5">
    <decoded_as>custom-auth</decoded_as>
    <description>Custom Application Authentication Failed</description>
    <group>authentication_failed,custom_app,</group>
</rule>
```

### Rule Test Results

Successful Phase 3:

```text
id: '100200'
level: '5'
description: 'Custom Application Authentication Failed'
**Alert to be generated.
```

### Analysis

The custom rule matched the decoded event and generated an alert.

The rule identifies the custom authentication failure event. It does not establish that the source IP is malicious or that an account was compromised.

## 5. Log Collection Investigation

### Configuration File

```text
/var/ossec/etc/ossec.conf
```

### Localfile Configuration

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/custom-app/auth.log</location>
</localfile>
```

### Collection Verification

Command:

```bash
grep -i 'custom-app/auth.log' /var/ossec/logs/ossec.log | tail -10
```

Observed:

```text
wazuh-logcollector: INFO: (1950): Analyzing file: '/var/log/custom-app/auth.log'.
```

This confirmed that Wazuh Logcollector was analyzing the configured file.

## 6. Alert Investigation

### Alert Verification Command

```bash
grep -E '"id":"100200"|custom-auth' /var/ossec/logs/alerts/alerts.json | tail -10
```

### Confirmed Alert Details

| Attribute | Value |
|---|---|
| Rule ID | 100200 |
| Rule Level | 5 |
| Description | Custom Application Authentication Failed |
| Decoder | custom-auth |
| Username | admin |
| Source IP | 192.168.1.50 |
| Extra data | invalid_password |
| Log location | /var/log/custom-app/auth.log |
| Agent ID | 000 |
| Agent name | localhost.localdomain |

### Alert Timestamp

```text
2026-11-17T10:59:50.036+0530
```

The timestamp represents the generated alert time, not necessarily the original application event time.

## 7. Investigation Findings

### Confirmed

- Custom application log file was created.
- Custom decoder successfully extracted the expected fields.
- Custom detection rule matched the decoded event.
- Wazuh Logcollector analyzed the configured log file.
- Wazuh Manager was active after restart.
- Alert with rule ID `100200` was generated.
- The alert was present in `alerts.json`.

### Not Independently Verified

- Whether the alert was indexed and searchable in Wazuh Discover.
- Whether the original timestamp-prefixed log format is supported by the final custom decoder configuration.
- Whether repeated authentication failures would trigger a separate correlation or threshold rule.
