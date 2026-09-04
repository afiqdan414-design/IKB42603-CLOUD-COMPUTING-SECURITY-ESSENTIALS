# IKB42603 Cloud Computing Security Essentials
## Lab 5 Report: Monitoring, Logging & Incident Detection

**Course Code:** IKB42603  
**Course Name:** Cloud Computing Security Essentials  
**Lab Title:** Lab 5 — Monitoring, Logging & Incident Detection  
**Institution:** UniKL MIIT  
**Lecturer:** Prof. Dr. Shahrulniza Musa  

---

## Executive Summary & Overview

This laboratory report documents the complete implementation of centralised cloud logging, log integrity protection via cryptographic hash-chaining, multi-event SIEM correlation for threat detection, and active incident containment/forensics. Using Docker and LocalStack to emulate Amazon CloudWatch Logs alongside standard shell tools (`grep`, `awk`, `sha256sum`), this lab demonstrates both the operational and security compliance dimensions of cloud telemetry.

---

## Lab Learning Outcomes Mapping

| Learning Outcome | Lab Demonstration |
| :--- | :--- |
| **1. Collect & centralise logs** | Centralised authentication logs from local host into LocalStack CloudWatch Logs (`/ccse/app` group). |
| **2. Distinguish logs vs. events & query activity** | Queried durable `auth.log` records to isolate 4 `LOGIN_FAIL` events originating from external IP `203.0.113.9`. |
| **3. Build a tamper-evident (hash-chained) log** | Constructed `auth.chain` with SHA-256 state chaining; proved tamper detection against modified `auth.tampered`. |
| **4. Detect an incident via event correlation** | Automated correlation script detecting high-frequency brute-force failure $\rightarrow$ login success $\rightarrow$ 500MB data exfiltration. |
| **5. Execute incident response lifecycle** | Enforced IP containment via `iptables`, collected immutable timestamped evidence (`evidence_20260905.log`), and verified SHA-256 integrity. |

---

## Technical Setup & Prerequisites

* **LocalStack Container:** Emulating AWS CloudWatch Logs service endpoint on `http://localhost:4566`.
* **AWS CLI v2:** Configured to target LocalStack endpoint via `EP='--endpoint-url=http://localhost:4566'`.
* **Shell Utilities:** `grep`, `awk`, `sha256sum`, `cut`, `paste`, `sed`, `docker`.

---

## Step-by-Step Task Execution & Findings

### Setup — Initialise LocalStack & CloudWatch Resources

LocalStack container was deployed and CloudWatch log structures were initialised:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack
EP='--endpoint-url=http://localhost:4566'

# Create Log Group and Log Stream
aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

---

### Task 1 — Generate Application Logs

An authentication log file (`auth.log`) was generated to simulate realistic application traffic, including reconnaissance and exploitation by an external IP (`203.0.113.9`).

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
EOF
```

---

### Task 2 — Centralise Logs (Ship to CloudWatch)

Logs were shipped sequentially to CloudWatch Logs using AWS CLI `put-log-events`, illustrating cascading telemetry collection.

```bash
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events --log-group-name /ccse/app --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null
  TS=$((TS+1000))
done < auth.log

# Retrieve and verify logs from central store
aws $EP logs get-log-events --log-group-name /ccse/app --log-stream-name auth \
  --query 'events[].message' --output text
```

#### Evidence Output:
```text
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5	2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9	2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9	2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9	2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9	2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9	2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
```

![Task 2 — Centralise Logs](file:///d:/UNIKL%20WOI/UNIKL%20MIIT/SEM%205/Short%20Sem/Cloud%20Computing/Lab5/Task%202%20%E2%80%94%20Centralise%20Logs%20%28Ship%20to%20CloudWatch%29.png)

---

### Task 3 — Query for Security-Relevant Activity

Using text processing tools, failed authentication attempts were aggregated by source IP address.

```bash
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

#### Evidence Output:
```text
      4 user=admin ip=203.0.113.9
```

![Task 3 — Query Security Activity](file:///d:/UNIKL%20WOI/UNIKL%20MIIT/SEM%205/Short%20Sem/Cloud%20Computing/Lab5/Task%203%20%E2%80%94%20Query%20for%20Security-Relevant%20Activity.png)

---

### Task 4 — Tamper-Proof (Hash-Chained) Logs

To protect against log alteration by compromised accounts, a cryptographic hash chain was constructed where line $N$'s hash incorporates line $N-1$'s hash:

$$H_n = \text{SHA256}(H_{n-1} \parallel \text{Line}_n)$$

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain

cat auth.chain
```

#### Hash Chain Records (`auth.chain`):
```text
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5 | 82da89a49dc1ca7d23b8a59f98d7e557ab36ce0c2d0c6e106fabe76e1f0acf39
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9 | 790aef7176d6effe76d077831c071f8500204bf842e7fd8aeda1b67b2e271a97
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9 | 1e0b2e8aaf5143fb95070a8e57b009f058f0d37c257d19409b4131894d29a9a8
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9 | 7fb62c66ded511605e22c8db9c4f57c9360aa27309ce65024a3e5ea35e3b6e94
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9 | 143253b549a74b9626e910fbe54ca12cb5431a0a4c9c4f2189ff27a3e2a17e01
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9 | 4cbfab7fecb703cf21f5df81b47dbf3a727c94442b09b714ac4bfaa3584cc638
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB | ababa787b4bf524d9daddca8c48e4909fc105769a6f17574f42cefe8f81233cf
```

#### Tamper Verification Test:
Simulating an adversary modifying exfiltration logs from `500MB` to `5MB`:

```bash
sed 's/500MB/5MB/' auth.log > auth.tampered
```

When re-running the hash chain against `auth.tampered`, the modified line alters its hash, causing all subsequent hashes in the chain to diverge and failing validation against `auth.chain`.

![Task 4 — Tamper-Proof Logs](file:///d:/UNIKL%20WOI/UNIKL%20MIIT/SEM%205/Short%20Sem/Cloud%20Computing/Lab5/Task%204%20%E2%80%94%20Tamper-Proof%20%28Hash-Chained%29%20Logs.png)

---

### Task 5 — Detect the Incident (Correlation)

Individually, a failed login or data export may seem benign or operational. However, correlating sequential events from the same IP reveals an attack kill-chain:

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)
echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration';
fi
```

#### Evidence Output:
```text
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

![Task 5 — Detect Incident](file:///d:/UNIKL%20WOI/UNIKL%20MIIT/SEM%205/Short%20Sem/Cloud%20Computing/Lab5/Task%205%20%E2%80%94%20Detect%20the%20Incident%20%28Correlation%29.png)

---

### Task 6 — Incident Response (Containment & Forensics)

#### 1. Containment (Network Isolation)
The attacker's IP (`203.0.113.9`) was blocked at the network interface using an `iptables` rule executed within a container having `NET_ADMIN` capabilities:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

#### Output:
```text
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       all  --  203.0.113.9          0.0.0.0/0
```

#### 2. Evidence Collection & Integrity Preservation
An immutable evidence log copy was created with a timestamp filename, followed by SHA-256 hash generation for chain-of-custody verification:

```bash
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

#### Output:
```text
0adc5d2ac06cbbdd366099bcc0540c4c0f76946e71b52e4c99322731696a203b  evidence_20260905.log
```

![Task 6 — Incident Response](file:///d:/UNIKL%20WOI/UNIKL%20MIIT/SEM%205/Short%20Sem/Cloud%20Computing/Lab5/Task%206%20%E2%80%94%20Incident%20Response.png)

---

## Short Incident Report

### 1. Detection
At 09:01:40, correlation rule triggers identified suspicious behavior originating from external IP `203.0.113.9`. The SIEM correlation logic detected 4 consecutive authentication failures (`LOGIN_FAIL`), followed immediately by 1 successful login (`LOGIN_OK`) as `admin`, culminating in a 500MB data export (`EXPORT_DATA`).

### 2. Analysis
* **Attacker IP:** `203.0.113.9`
* **Target Account:** `admin`
* **Attack Sequence:** Brute-force credential guessing $\rightarrow$ Account compromise $\rightarrow$ Data exfiltration (500MB).
* **Root Cause:** Inadequate rate-limiting and missing multi-factor authentication (MFA) on the administrative authentication endpoint allowed persistent password probing.

### 3. Containment
Immediate active containment was executed at the firewall level by deploying an `iptables DROP` rule for source address `203.0.113.9` (`iptables -A INPUT -s 203.0.113.9 -j DROP`).

### 4. Evidence & Integrity
Evidence was extracted to `evidence_20260905.log` and cryptographically locked with SHA-256 hash `0adc5d2ac06cbbdd366099bcc0540c4c0f76946e71b52e4c99322731696a203b`. Log integrity was verified using CloudWatch central storage and hash-chain audit trails (`auth.chain`).

### 5. Lesson Learned
Relying solely on single-line alert thresholds (e.g. flagging only failed logins) fails to capture full attack lifecycles. Multi-event correlation rules, mandatory MFA, automated IP rate-limiting/shunning (SOAR), and offsite immutable log archiving are vital to prevent and detect multi-stage compromise.

---

## Short-Answer Questions

### Q1. What is the difference between a log and an event? Give an example of each from this lab.
* **Log:** A durable, immutable append-only record of a past activity stored for auditing and analysis.  
  * *Example from lab:* The file entry `2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9` in `auth.log`.
* **Event:** A real-time signal, trigger, or notification emitted when specific conditions or thresholds are satisfied.  
  * *Example from lab:* The SIEM output `ALERT: probable brute-force -> compromise -> data exfiltration` triggered when 4 failures, 1 success, and 1 export were detected from IP `203.0.113.9`.

### Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?
* **Why tamper-proof:** Attackers routinely edit or delete local application logs to cover their tracks and hinder forensic investigation.
* **How hash chain achieves this:** Each log entry hash is computed using both its own content and the hash of the preceding line ($H_n = \text{SHA256}(H_{n-1} \parallel \text{Line}_n)$). Modifying, deleting, or reordering any log entry breaks all subsequent hashes in the chain, instantly exposing log tampering during verification.

### Q3. How did correlation detect an incident that no single log line revealed?
* Individually, a `LOGIN_FAIL` line could be a simple typo, a `LOGIN_OK` line looks like standard user activity, and an `EXPORT_DATA` line appears to be a legitimate backup operation.
* Correlation links these disparate log lines across time and context (matching IP `203.0.113.9`), revealing the underlying attack narrative: **Brute Force $\rightarrow$ Compromise $\rightarrow$ Exfiltration**.

### Q4. List the incident-response steps you performed and the goal of each.
1. **Detect:** Run correlation script across centralized telemetry to identify suspicious multi-stage activity.
2. **Contain:** Apply an `iptables -j DROP` rule to immediately block traffic from attacker IP `203.0.113.9` and halt data loss.
3. **Collect Evidence:** Create timestamped immutable copy `evidence_20260905.log` and record SHA-256 hash `evidence.sha256` for forensic chain-of-custody.
4. **Document:** Compile an incident report detailing timelines, root cause, containment measures, and remediation lessons.

### Q5. How do the same logs serve both security monitoring and compliance evidence (Weeks 6, 11)?
* **Security Monitoring:** Logs provide real-time operational visibility, enabling SIEM detection rules to alert on active threats and trigger automated containment.
* **Compliance Evidence:** Immutable, centralized, and tamper-proof logs provide historical, auditable proof of compliance for regulatory frameworks (e.g. ISO 27001, SOC 2, PCI-DSS) proving access control monitoring, data movement traceability, and non-repudiation.

---

## Final Verification Command & Checklist

### Verification Execution

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

#### Verification Result Output:
```json
{
    "logGroups": [
        {
            "logGroupName": "/ccse/app",
            "creationTime": 1788542751308,
            "metricFilterCount": 0,
            "arn": "arn:aws:logs:us-east-1:000000000000:log-group:/ccse/app:*",
            "storedBytes": 397,
            "logGroupClass": "STANDARD",
            "logGroupArn": "arn:aws:logs:us-east-1:000000000000:log-group:/ccse/app"
        }
    ]
}
evidence_20260905.log: OK
```

![Verification Command](file:///d:/UNIKL%20WOI/UNIKL%20MIIT/SEM%205/Short%20Sem/Cloud%20Computing/Lab5/Verification%20command.png)

---

### Security Best-Practices Checklist

- [x] **Centralised Telemetry:** Logs are shipped to CloudWatch Logs (`/ccse/app`) rather than remaining isolated on local endpoints.
- [x] **Queryable Audit Trail:** Security-relevant events (failed logins by IP) can be queried and parsed programmatically.
- [x] **Tamper-Evident Logs:** Log entries are protected using SHA-256 state hash-chaining (`auth.chain`) to detect downstream modification.
- [x] **Multi-Event Correlation:** Incident detection correlates sequential events across time to uncover complex attack patterns.
- [x] **Incident Response Lifecycle:** Executed full IR workflow: Containment (`iptables`), Evidence Collection (`sha256sum`), and Incident Documentation.
