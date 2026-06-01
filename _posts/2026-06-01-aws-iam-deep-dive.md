---
layout: post
title: "AWS IAM Deep Dive - Quản lý truy cập AWS chuyên sâu"
date: 2026-06-01
categories: [security]
tags: [iam, policy, sts, cross-account, access-analyzer]
---

## Mục lục
1. [Góc nhìn tổng quan - Hệ thống an ninh tòa nhà AWS](#overview)
2. [Policy Types - Các loại chính sách](#policy-types)
3. [Policy Evaluation Logic - Luật đánh giá](#evaluation)
4. [STS và AssumeRole](#sts)
5. [Cross-Account Access](#cross-account)
6. [Condition Keys](#conditions)
7. [IAM Access Analyzer](#access-analyzer)
8. [Permission Boundaries](#boundaries)
9. [Service Control Policies (SCPs)](#scps)
10. [Tổng kết và best practices](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

IAM giống **hệ thống an ninh tòa nhà** nhiều tầng:
- **Identity-based Policy** = Thẻ nhân viên ghi quyền (Alice được vào phòng server)
- **Resource-based Policy** = Biển trên cửa (phòng này cho phép team IT vào)
- **Permission Boundary** = Giới hạn tối đa (intern chỉ được đi tầng 1-2 dù thẻ ghi gì)
- **SCP** = Quy định toàn công ty (không ai được vào phòng X, bất kể chức vụ)
- **STS AssumeRole** = Thẻ tạm thời (visitor badge, hết hạn sau 1 giờ)

---

## 2. Policy Types {#policy-types}

```
1. Identity-based Policies (gắn với user/role/group):
   - AWS Managed: AmazonS3ReadOnlyAccess
   - Customer Managed: Custom policies
   - Inline: Embedded in single user/role

2. Resource-based Policies (gắn với resource):
   - S3 bucket policy
   - SQS queue policy
   - Lambda function policy
   - Trust policy (who can assume a role)

3. Permission Boundaries (giới hạn tối đa):
   - Limits effective permissions of identity-based policies
   - Used for delegation (admin creates roles for devs)

4. Service Control Policies (SCPs):
   - AWS Organizations level
   - Limits ALL accounts in OU
   - Cannot grant, only restrict

5. Session Policies:
   - Passed when assuming role or federating
   - Further restricts session permissions
```

---

## 3. Policy Evaluation Logic {#evaluation}

```
AWS Policy Evaluation Algorithm:
1. Start with IMPLICIT DENY (default: everything denied)
2. Evaluate ALL applicable policies
3. If ANY policy has EXPLICIT DENY → DENY (always wins!)
4. If ANY policy has ALLOW → ALLOW
5. Otherwise → DENY (implicit)

Decision flowchart:
  Explicit Deny? → YES → DENY
  ↓ NO
  SCP Allow? → NO → DENY
  ↓ YES
  Resource-based Allow? → YES → ALLOW (for same-account)
  ↓ NO
  Identity-based Allow? → NO → DENY
  ↓ YES
  Permission Boundary Allow? → NO → DENY
  ↓ YES
  Session Policy Allow? → NO → DENY
  ↓ YES
  → ALLOW

Key insight: Deny ALWAYS wins. An explicit Deny in ANY policy 
overrides Allow in ALL other policies.
```

---

## 4-5. STS và Cross-Account {#sts}

### STS AssumeRole
```json
// Trust Policy (on the role being assumed):
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:role/admin-role"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": { "sts:ExternalId": "my-external-id" }
    }
  }]
}
```

### Cross-Account Access
```
Account A (111111111111) wants to access S3 in Account B (222222222222):

Option 1: Resource-based policy on S3 bucket
Option 2: Role in Account B, assumed from Account A

Method 2 (recommended):
1. Account B creates role with trust policy allowing Account A
2. Account A user assumes role in Account B
3. Gets temporary credentials scoped to Account B

aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/cross-account-role \
  --role-session-name my-session
```

---

## 6-10. Conditions, Access Analyzer, Boundaries, SCPs {#conditions}

### Condition Keys
```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "NotIpAddress": { "aws:SourceIp": "10.0.0.0/8" },
    "Bool": { "aws:MultiFactorAuthPresent": "false" },
    "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "eu-west-1"] }
  }
}
// Deny everything if: not from VPN AND no MFA AND wrong region
```

### IAM Access Analyzer
```
Analyzes resource policies to find external access:
- S3 buckets accessible from outside account
- IAM roles assumable by external entities
- KMS keys shared externally
- Lambda functions invokable externally

Uses "Zelkova" (automated reasoning/formal verification)
→ Mathematical proof of what's accessible, not just pattern matching

Also: Policy validation, Access preview, unused access
```

### Best Practices
```
1. Least privilege (start with zero, add as needed)
2. Use roles instead of long-term access keys
3. MFA for all human users
4. Regularly review and remove unused permissions
5. Use Access Analyzer to find over-permissive policies
6. Use SCPs to prevent dangerous actions org-wide
7. Use permission boundaries for delegated admin
8. Tag-based access control for scale
9. Never use root account for daily tasks
10. Monitor with CloudTrail + Athena queries
```

---
