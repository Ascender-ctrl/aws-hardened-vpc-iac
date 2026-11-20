# Hardened Amazon Web Services Security Baseline (Terraform + Checkov)

A security-hardened Infrastructure-as-Code baseline provisioning an isolated Amazon Web Services Virtual Private Cloud (VPC), zero-trust Simple Storage Service (S3) storage, customer-managed Key Management Service (KMS) encryption, and tamper-evident multi-region CloudTrail auditing.

## Architecture Overview

```mermaid
flowchart TB
    subgraph KMS["Encryption Boundary (Key Management Service)"]
        CMK["Customer-Managed Key<br/>Annual Auto-Rotation Enabled"]
    end

    subgraph VPC["Isolated Virtual Private Cloud (10.0.0.0/16)"]
        SG["Default Security Group<br/>(All Ingress and Egress Denied)"]
        SubA["Private Subnet A (10.0.1.0/24)<br/>No Public IP Auto-Assign"]
        SubB"Private Subnet B (10.0.2.0/24)<br/>No Public IP Auto-Assign"]
        Flow["Virtual Private Cloud Flow Logs (ALL Traffic)"]
        CW["CloudWatch Log Group<br/>(365-Day Retention)"]
        Flow --> CW
    end

    subgraph Storage["Zero-Trust Storage and Audit Pipeline"]
        DataB["Hardened Data Bucket<br/>Public Access Blocked | Transport Layer Security Only | Versioned"]
        Trail["Multi-Region CloudTrail<br/>Log File Validation Enabled"]
        TrailB["CloudTrail Audit Bucket"]
        AccessB["Centralized Access Logs Bucket"]
        SNS["Simple Notification Service Alert Topic"]

        DataB -. "Server Access Logs" .-> AccessB
        TrailB -. "Server Access Logs" .-> AccessB
        Trail --> TrailB
        Trail --> SNS
    end

    CMK -. "Encrypts" .-> CW
    CMK -. "Encrypts" .-> DataB
    CMK -. "Encrypts" .-> Trail
    CMK -. "Encrypts" .-> TrailB
    CMK -. "Encrypts" .-> AccessB
    CMK -. "Encrypts" .-> SNS
```

## Threat Model and Attack-Path Remediation

Each control directly mitigates real-world Amazon Web Services misconfigurations observed during cloud penetration testing labs:

| Attack Vector | Terraform Control Implemented | File |
| :--- | :--- | :--- |
| **Unauthenticated Storage Enumeration and Bucket Exfiltration** | Enforced all 4 Public Access Block flags and an explicit Deny policy when Transport Layer Security (`aws:SecureTransport`) is false. | `s3.tf` |
| **Silent Data Tampering or Accidental Secret Deletion** | Enabled Simple Storage Service object versioning across all data, audit, and access-logging buckets. | `s3.tf`, `cloudtrail.tf` |
| **Unencrypted Data at Rest and Stale Encryption Keys** | Provisioned a dedicated Customer-Managed Key (`enable_key_rotation = true`) encrypting storage buckets, CloudTrail logs, notification alerts, and network flow logs. | `main.tf` |
| **Unmonitored API Reconnaissance and Log Tampering** | Configured a multi-region CloudTrail with log file integrity validation (`enable_log_file_validation = true`), Simple Notification Service delivery alerts, and isolated log storage. | `cloudtrail.tf` |
| **Lateral Movement and Unintended Public Exposure** | Locked down the Virtual Private Cloud default security group, disabled public Internet Protocol address assignment on subnets, and enabled network flow logs for all traffic. | `main.tf` |

## Static Security Verification (Checkov)

Validated locally with Terraform `validate` and Bridgecrew Checkov:

```text
terraform scan results:
Passed checks: 46, Failed checks: 0, Skipped checks: 0
```

### Reproduce Locally

```bash
terraform fmt -check
terraform init -backend=false
terraform validate
checkov -d . --framework terraform
```
