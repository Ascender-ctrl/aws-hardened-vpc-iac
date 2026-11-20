# Hardened AWS Security Baseline (Terraform + Checkov)

A security-hardened Infrastructure-as-Code (IaC) baseline provisioning an isolated AWS VPC, zero-trust S3 storage, customer-managed KMS encryption, and tamper-evident multi-region CloudTrail auditing.

## Architecture Overview

```mermaid
flowchart TB
    subgraph KMS["Encryption Boundary (AWS KMS)"]
        CMK["Customer-Managed Key (CMK)<br/>Annual Auto-Rotation Enabled"]
    end

    subgraph VPC["Isolated VPC (10.0.0.0/16)"]
        SG["Default Security Group<br/>(All Ingress/Egress Denied)"]
        SubA["Private Subnet A (10.0.1.0/24)<br/>No Public IP Auto-Assign"]
        SubB["Private Subnet B (10.0.2.0/24)<br/>No Public IP Auto-Assign"]
        Flow["VPC Flow Logs (ALL Traffic)"]
        CW["CloudWatch Log Group<br/>(365-Day Retention)"]
        Flow --> CW
    end

    subgraph Storage["Zero-Trust S3 & Audit Pipeline"]
        DataB["Hardened Data Bucket<br/>Public Access Blocked | TLS-Only | Versioned"]
        Trail["Multi-Region CloudTrail<br/>Log File Validation Enabled"]
        TrailB["CloudTrail Audit Bucket"]
        AccessB["Centralized Access Logs Bucket"]
        SNS["SNS Security Alert Topic"]

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
eof

## Threat Model & Attack-Path Remediation

Each control directly mitigates real-world AWS misconfigurations observed during cloud penetration testing labs:

| Attack Vector | Terraform Control Implemented | File |
| :--- | :--- | :--- |
| **Unauthenticated S3 Enumeration & Bucket Exfiltration** | Enforced all 4 `aws_s3_bucket_public_access_block` flags and explicit `Deny` policy on `aws:SecureTransport = "false"`. | `s3.tf` |
| **Silent Data Tampering or Accidental Secret Deletion** | Enabled S3 object versioning (`Enabled`) across all data, audit, and access-logging buckets. | `s3.tf`, `cloudtrail.tf` |
| **Unencrypted Data at Rest & Stale Keys** | Provisioned a dedicated Customer-Managed KMS Key (`enable_key_rotation = true`) encrypting S3 buckets, CloudTrail logs, SNS alerts, and Flow Logs. | `main.tf` |
| **Unmonitored API Recon & Log Tampering** | Configured a multi-region `aws_cloudtrail` with `enable_log_file_validation = true`, SNS delivery notifications, and isolated S3 log storage. | `cloudtrail.tf` |
| **Lateral Movement & Unintended Public Exposure** | Locked down the VPC default security group (`aws_default_security_group`), disabled public IP assignment on subnets, and enabled `ALL` VPC Flow Logs. | `main.tf` |

## Static Security Verification (Checkov)

Validated locally with Terraform `validate` and Bridgecrew Checkov:

    terraform scan results:
    Passed checks: 46, Failed checks: 0, Skipped checks: 0

### Reproduce Locally

    terraform fmt -check
    terraform init -backend=false
    terraform validate
    checkov -d . --framework terraform
