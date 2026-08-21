# Lab 5: Monitoring, Logging and Incident Detection

| Item | Details |
| --- | --- |
| Student name | MUHAMMAD ARIF SHAFIRA BIN SHAHRIN AMRI |
| Lab | L02 - B04 |
| Course | IKB42603 Cloud Computing |
| Topic | Monitoring, Logging and Incident Detection |

## Objectives

This lab centralised application authentication logs in CloudWatch Logs through LocalStack, queried security-relevant activity, made the log tamper-evident with a hash chain, correlated events to detect an incident, and performed containment and evidence collection.

## Task 1: Generate Application Logs

An `auth.log` file was created with one normal login, four failed administrator login attempts from `203.0.113.9`, a successful login from the same IP address, and a 500 MB data export. This dataset represents an attacker attempting to guess credentials before accessing and exporting data.

![Task 1 - generated authentication log](Task%201/Lab%205%20Task%201.png)

## Task 2: Centralise Logs

LocalStack was used as the local CloudWatch Logs endpoint. The `/ccse/app` log group and `auth` stream received each line from `auth.log`. The `get-log-events` output shows that all seven entries were successfully read back from the central log service, including the failed attempts and export event.

![Task 2 - CloudWatch Logs read-back](Task%202/Task%202.png)

## Task 3: Query Security-Relevant Activity

The failed-login records were filtered, grouped by source IP, and counted. The result was:

```text
4 ip=203.0.113.9
```

This shows four failed logins originating from `203.0.113.9`.

![Task 3 - failed-login count by IP](Task%203/Lab%206%20Task%203.png)

## Task 4: Tamper-Proof Hash-Chained Logs

Each log line was combined with the previous hash and hashed with SHA-256. The resulting `auth.chain` file stored a hash beside every event. The original final hash was:

```text
8072200785da77199ee9936cfd049e9e7d246d3dc96644812fd210aa21ef190c
```

The export value was then changed from `size=500MB` to `size=5MB` in `auth.tampered`, and the chain was recomputed. Its final hash was:

```text
55152489e18ac285c0906265a92808257b89b3418c4e1a2e7d8c8d5832bb0ef5
```

The hashes are different, proving the modified log no longer matches the original integrity chain. The screenshot's attempted `awk` display selects the event field rather than the hash; the values above are the actual SHA-256 final-chain values.

![Task 4 - original hash chain](Task%204/Task%204%20-%201.png)

![Task 4 - tampered log chain creation](Task%204/Task%204%20-%202.png)

![Task 4 - attempted final-hash display](Task%204/Task%204%20-%203.png)

## Task 5: Detect the Incident by Correlation

The activity for `203.0.113.9` was correlated. It produced `fails=4`, `success=1`, and `export=1`, which exceeded the detection condition of at least three failures followed by a successful login and an export. The detection output was:

```text
ALERT: probable brute-force -> compromise -> data exfiltration
```

![Task 5 - correlation alert](Task%205/Task%205.png)

## Task 6: Contain the Incident and Collect Evidence

The attacker IP address was blocked with an `iptables` INPUT rule. The displayed rule confirms `203.0.113.9` was configured with the `DROP` action. An evidence copy of the original log was created as `evidence_20260821.log`, and its SHA-256 checksum was recorded in `evidence.sha256`:

```text
215268ee95d18e1b20af5e9c25dae2f95f736a5a0fb5433b93e91c03a7d655a0  evidence_20260821.log
```

![Task 6 - containment rule](Task%206/Task%206%20-%201.png)

![Task 6 - evidence hash](Task%206/Task%206%20-%202.png)

## Incident Report

### Detection

An alert was raised for source IP `203.0.113.9` after correlation identified four failed administrator login attempts, followed by one successful administrator login and a 500 MB data export. This pattern indicates a probable brute-force attack leading to account compromise and data exfiltration.

### Analysis

The failed attempts occurred at 09:01:10, 09:01:12, 09:01:15, and 09:01:18. A successful login from the same IP followed at 09:01:22, then an `EXPORT_DATA` event at 09:01:40. Although any single record may appear harmless by itself, the time-ordered sequence establishes a credible attack path.

### Containment

The source address `203.0.113.9` was blocked using an `iptables` INPUT rule with the `DROP` action. This limits further connections from that IP while the incident is investigated. In a production environment, the corresponding permanent network, security-group, and account-control actions should also be applied.

### Evidence and Integrity

The original authentication log was copied to the timestamped evidence file `evidence_20260821.log`. Its SHA-256 hash was saved in `evidence.sha256` so that later verification can detect changes. The source log was also hash-chained: changing the export size altered the final chain hash, making the modification evident.

### Lesson Learned

Centralising logs provides visibility, but meaningful detection requires correlation across events. Audit records should be protected with integrity controls and copied to a separate, append-only location so an attacker cannot silently modify the evidence after compromise.

## Short-Answer Questions

### 1. What is the difference between a log and an event? Give an example of each from this lab.

A log is a durable record of an action that has occurred. For example, `LOGIN_FAIL user=admin ip=203.0.113.9` is a stored authentication log entry. An event is a meaningful occurrence or trigger that can be acted upon, often generated from one or more logs. In this lab, the correlation alert for four failures followed by a success and export is an event.

### 2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?

Audit logs must be tamper-proof so they remain trustworthy for incident investigation, accountability, and compliance evidence. In a hash chain, each entry's hash is calculated from both the current log entry and the previous hash. Changing one record produces a different hash for that entry and every following entry, exposing the alteration when the chain is verified against a protected final hash.

### 3. How did correlation detect an incident that no single log line revealed?

Correlation combined records from the same IP address and evaluated their order and count. Four failures alone could be user error, a successful login alone may be normal, and an export alone may be authorised. The sequence of repeated failures, successful login, and large export from `203.0.113.9` revealed the probable compromise and exfiltration pattern.

### 4. List the incident-response steps you performed and the goal of each.

1. **Detect:** Queried and correlated failed logins, successful login, and export events to identify the suspected incident.
2. **Contain:** Added an `iptables` DROP rule for `203.0.113.9` to prevent further traffic from the suspected attacker.
3. **Collect evidence:** Created a timestamped log copy and SHA-256 checksum to preserve evidence for investigation.
4. **Preserve integrity:** Used hash chaining to expose any modification to the log records.
5. **Document:** Recorded the detection, analysis, containment, evidence, and lesson learned in this report.

### 5. How do the same logs serve both security monitoring and compliance evidence?

For security monitoring, logs provide the data used to detect abnormal behaviour, correlate events, alert responders, and investigate incidents. For compliance, the same centralised and integrity-protected records demonstrate who performed actions, when they occurred, how the organisation monitored activity, and that evidence has not been altered. Retention and protected storage allow the records to support later audits.

## Verification Commands

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

The first command confirms the central log group is available. The second command verifies that the evidence file still matches its recorded SHA-256 hash.

## Security Best-Practices Checklist

- [x] Logs were centralised in CloudWatch Logs through LocalStack.
- [x] Failed login activity was queried and grouped by IP address.
- [x] Logs were made tamper-evident using a SHA-256 hash chain.
- [x] The incident was detected through event correlation.
- [x] Containment, evidence collection, and documentation were completed.
