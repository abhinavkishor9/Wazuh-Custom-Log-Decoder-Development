# Wazuh-Custom-Log-Decoder-Development
## Overview
### What Is a Wazuh Decoder?

A Wazuh decoder is a component that interprets raw log messages and extracts structured information from them.

Applications often generate logs in custom formats. Wazuh decoders allow security analysts to parse these messages and make their contents available to detection rules.

### Example

Raw application log:

```text
AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password
```

A custom decoder can extract:

| Field | Value |
|---|---|
| Event type | AUTH_FAILED |
| Username | admin |
| Source IP | 192.168.1.50 |
| Failure reason | invalid_password |

### How Wazuh Decoding Works

#### 1. Raw Log Collection

Wazuh Logcollector reads logs from configured sources.

A custom application log can be monitored through a `localfile` configuration in `ossec.conf`.

#### 2. Pre-decoding

Wazuh performs pre-decoding to identify standard log header information, when available.

This may include timestamps, hostnames and program names.

#### 3. Decoder Matching

The decoder identifies the log format using elements such as:

- `prematch`
- `program_name`
- `regex`
- `order`

The regular expression extracts values from the log message.

#### 4. Field Extraction

The decoder maps captured values to Wazuh-supported fields.

Example:

```xml
<decoder name="custom-auth">
    <prematch>AUTH_FAILED</prematch>
    <regex>AUTH_FAILED user=(\S+) source_ip=(\S+) reason=(\S+)</regex>
    <order>user,srcip,extra_data</order>
</decoder>
```

#### 5. Rule Matching

After decoding, Wazuh evaluates the event against its detection rules.

A custom rule can match a decoder by name:

```xml
<decoded_as>custom-auth</decoded_as>
```

If the rule conditions match, Wazuh generates an alert.

### Decoder vs Rule

| Decoder | Rule |
|---|---|
| Parses raw logs | Evaluates decoded events |
| Extracts fields | Applies detection conditions |
| Identifies log structure | Assigns alert level and description |
| Does not independently generate an alert | Generates an alert when matched |

### Important Wazuh Decoder Fields

| Field | Purpose |
|---|---|
| `user` | Destination username |
| `srcuser` | Source username |
| `srcip` | Source IP address |
| `dstip` | Destination IP address |
| `status` | Event status |
| `action` | Event action |
| `extra_data` | Additional extracted information |

### Decoder Testing

Wazuh provides:

```bash
/var/ossec/bin/wazuh-logtest
```

It displays three phases:

- Phase 1: Pre-decoding
- Phase 2: Decoder matching and field extraction
- Phase 3: Rule matching and alert evaluation

### Why Custom Decoders Matter

Custom decoders help security teams:

- Integrate application-specific logs into SIEM monitoring.
- Extract useful investigation fields.
- Develop custom detection rules.
- Improve visibility into application activity.
- Build detections for log sources not covered by existing decoders.

### Key Takeaway

A decoder transforms raw application messages into structured security events. Detection rules then evaluate those events and generate alerts.

Successful detection requires the complete workflow:

**Log Collection → Pre-decoding → Decoder → Field Extraction → Rule Matching → Alert Generation**

This lab demonstrates how to develop and implement a custom Wazuh decoder for application authentication logs. The objective was to transform raw application log messages into structured security events and generate alerts using a custom detection rule.

**Environment:** Rocky Linux 9.8  
**Wazuh Version:** 4.13.1  
**Log Source:** `/var/log/custom-app/auth.log`

## Lab Objectives

The objective of this lab is to understand how Wazuh processes custom application logs and converts unstructured log messages into structured security events using custom decoders and detection rules.

The lab focuses on the following objectives:

- Understand the role of Wazuh decoders in custom log parsing and security monitoring.
- Develop a custom decoder using `prematch`, regular expressions, and field mapping.
- Extract relevant authentication information, including username, source IP address, and failure reason.
- Validate decoder functionality and field extraction using `wazuh-logtest`.
- Create a custom Wazuh detection rule to identify authentication failure events.
- Configure Wazuh Logcollector to monitor a custom application log file.
- Verify that collected events are correctly decoded and matched against the custom detection rule.
- Investigate generated alerts through Wazuh alert logs and understand the distinction between local alert generation and dashboard indexing.

By completing this lab, the analyst should be able to develop a basic custom log parsing and detection workflow and understand how application-specific events can be integrated into Wazuh monitoring.

## Lab Scenario

### Scenario Overview

An organization uses an internally developed application that records authentication activity in a custom log format. These events contain useful information such as the attempted username, source IP address, and reason for authentication failure.

However, the application log format is not automatically interpreted by the existing Wazuh decoder configuration. As a result, the security monitoring team needs to develop a custom decoder to extract meaningful fields and make the events suitable for security detection.

### Investigation Context

A controlled authentication failure event is generated in the lab:

```text
AUTH_FAILED user=admin source_ip=192.168.1.50 reason=invalid_password
```

The objective is to integrate this application log into Wazuh and build a detection mechanism that identifies authentication failure events.

### Analyst Tasks

The analyst is required to:

1. Examine the existing Wazuh decoder configuration and identify where custom decoders should be maintained.
2. Develop a decoder that recognizes the authentication failure message and extracts the relevant fields.
3. Test the decoder and investigate any parsing or field-mapping issues.
4. Create a custom detection rule that matches events decoded by the new decoder.
5. Configure Wazuh to collect the custom application log file.
6. Generate a fresh test event and verify that the manager processes it.
7. Examine the generated alert to confirm the decoder name, extracted fields, rule ID, and alert level.

### Investigation Goal

Determine whether Wazuh can successfully collect, interpret, and generate an alert from the custom application authentication event.

The investigation should distinguish between successful log parsing, detection rule matching, alert generation, and dashboard visibility.

### Expected Learning Outcome

The analyst should understand how custom decoders extend Wazuh's log analysis capabilities and how detection rules transform extracted application information into actionable security alerts.

The lab also demonstrates the importance of validating each stage of the event-processing pipeline rather than assuming that successful decoder testing guarantees end-to-end alert visibility.

## Sample Log

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

