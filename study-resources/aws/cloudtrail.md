# AWS CloudTrail Deep Dive — 2-Hour Study Plan

**Total Time:** ~120 minutes
**Created:** 2026-05-13
**Difficulty:** Intermediate / Advanced

## Overview

This plan goes beyond "CloudTrail logs API calls" and pushes into the parts practitioners actually get wrong in production: the three event-type taxonomy (Management vs Data vs Insights vs the new Network Activity events), advanced event selector filtering syntax for cost-controlled data event capture, CloudTrail Lake federation for cross-source SQL correlation, Organization-trail multi-account centralization (the Control Tower log-archive pattern), log file integrity validation via the SHA-256 digest hash chain, and the OIDC `AssumeRoleWithWebIdentity` `sub`-claim correlation workflow that ties CloudTrail back to the GitHub Actions hardening work from May 11. Builds directly on yesterday's CloudWatch deep dive — treats CloudTrail as the *audit/forensics* counterpart to CloudWatch's *operational telemetry*.

## Resources

### 1. CloudTrail Concepts + Understanding CloudTrail Events (Official Docs) - 18 min
- **URL:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-concepts.html
- **Companion:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-events.html
- **Type:** Official Docs
- **Summary:** Anchor the mental model. Read the concepts page first (trail vs event data store vs channel vs delivery), then the events page which is the authoritative source on Management / Data / Insights / Network Activity event categories and how the default trail behavior differs (management free for first copy, data events always paid). Pay attention to the readOnly field — it is the cheapest filter to halve your data-event volume.

### 2. Filtering Data Events with Advanced Event Selectors (Official Docs) - 15 min
- **URL:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/filtering-data-events.html
- **Type:** Official Docs (Reference)
- **Summary:** The single most important page for keeping the CloudTrail bill under control. Walks through the exact field grammar — `eventCategory`, `resources.type`, `resources.ARN`, `eventName`, `readOnly`, `userIdentity.arn` — and the operator set (`Equals`, `StartsWith`, `EndsWith`, `NotStartsWith`, `NotEndsWith`; no wildcards). Understand the SELECT-OR / DESELECT-AND combinatorics rule: it is the #1 source of "why isn't my filter matching?" tickets. Skim the JSON examples for S3 + Lambda + DynamoDB.

### 3. Last Week in AWS — CloudTrail Cost Reality Check (Corey Quinn) - 12 min
- **URL:** https://www.lastweekinaws.com/blog/the-amazon-prime-day-2023-aws-bill/
- **Type:** Practitioner Blog
- **Summary:** Read the CloudTrail subsection. Corey's Prime Day analysis showed CloudTrail processed 830 billion events and made the point that *data events cost roughly 20× the underlying API call you are monitoring*. This is the gut-check that turns "enable all data events" into "advanced event selectors with deny-by-default and explicit allow-lists for sensitive buckets." Reinforces why Resource #2 matters.

### 4. CloudTrail Network Activity Events for VPC Endpoints GA (AWS News Blog) - 12 min
- **URL:** https://aws.amazon.com/blogs/aws/aws-cloudtrail-network-activity-events-for-vpc-endpoints-now-generally-available/
- **Type:** AWS Blog (Feb 2025 GA announcement)
- **Summary:** The newest event category most practitioners haven't internalized yet. Covers the five supported services (S3, EC2, KMS, Secrets Manager, CloudTrail itself), the `VpceAccessDenied`-only filter pattern for cheap data-perimeter detection, and how this finally lets you see *who from outside your org* hit your VPC endpoints — the missing piece for data-perimeter SCPs.

### 5. CloudTrail Lake + Federation for Cross-Source SQL (Official Docs) - 20 min
- **URL:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-lake.html
- **Companion:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/query-federation.html
- **Type:** Official Docs
- **Summary:** Read the Lake overview, then the federation page. Lake stores events in columnar ORC and supports the full Trino SELECT grammar; federation registers the event data store with Glue Data Catalog + Lake Formation so you can JOIN CloudTrail events against Config items, Audit Manager evidence, and external sources from Athena. **Important 2026 note:** AWS announced CloudTrail Lake is closing to *new* customers from May 31, 2026 — if you're using it, understand the alternative is shipping logs to S3 + querying with Athena directly (covered in Resource 8).

### 6. Validating Log File Integrity — Digest File Structure (Official Docs) - 12 min
- **URL:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-intro.html
- **Companion:** https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-validation-digest-file-structure.html
- **Type:** Official Docs
- **Summary:** The forensics-grade story most engineers skip. SHA-256 hashes per log file, hourly digest files signed with SHA-256-RSA, each digest references the previous one's signature forming a *hash chain* that makes tampering computationally infeasible without detection. Learn the per-region private/public key pair model — critical when validating logs collected from one region in a different region's audit account. This is what auditors actually ask about for SOC 2 / PCI / ISO 27001.

### 7. AWS Control Tower + Organization CloudTrail Logging (Official Docs) - 12 min
- **URL:** https://docs.aws.amazon.com/controltower/latest/userguide/logging-using-cloudtrail.html
- **Companion (deeper):** https://aws.amazon.com/blogs/mt/use-existing-logging-and-security-account-with-aws-control-tower/
- **Type:** Official Docs + AWS Cloud Operations Blog
- **Summary:** The canonical multi-account centralization pattern. Control Tower landing-zone v3.0 deploys an *Organization-level* trail from the management account that captures every member account's events into a single S3 bucket in the dedicated Log Archive account — with the Audit account getting cross-account read access. Understand the gotcha: enabling Control Tower while you already have an Organization trail forces you to disable trusted access first, which breaks all existing org trails until reconciled.

### 8. Analyzing CloudTrail Logs with Athena — cloudonaut - 15 min
- **URL:** https://cloudonaut.io/analyzing-cloudtrail-with-athena/
- **Type:** Practitioner Blog (Andreas & Michael Wittig)
- **Summary:** The DIY alternative to CloudTrail Lake — partitioned Athena table over the S3 archive, with battle-tested SQL examples for common security investigations (root logins, failed AssumeRole, CreateAccessKey events). Crucial for the **OIDC `sub` claim correlation pattern**: shows the JSON-extract syntax you need to pull `requestParameters.webIdentityToken.sub` out of `AssumeRoleWithWebIdentity` events so you can answer "every GitHub workflow run from `repo:my-org/payments-api:ref:refs/heads/main` that assumed prod-deploy-role this month." This directly extends the May 11 GitHub Actions hardening doc.

### 9. Building Custom Threat Detection with EventBridge + CloudTrail - 12 min
- **URL:** https://medium.com/@davebhargavi507/building-custom-threat-detection-rules-in-aws-using-eventbridge-and-cloudtrail-1b38f54ae487
- **Backup (same pattern):** https://defersec.com/security-monitoring-in-aws-cloudtrail-cloudwatch-and-eventbridge/
- **Type:** Practitioner Blog
- **Summary:** The real-time-response half (CloudTrail is also a *streaming* source, not just an archive). Walks through EventBridge rule patterns for the canonical "sensitive event" set: `StopLogging`, `DeleteTrail`, `ConsoleLogin` with `userIdentity.type=Root`, `AuthorizeSecurityGroupIngress` with `0.0.0.0/0`, `CreateAccessKey`. Pattern is CloudTrail to EventBridge default bus to Lambda (formatter) to SNS to Slack webhook. Note the design rule: EventBridge for one-off critical events, CloudWatch metric filters for *aggregate* anomalies — they are not interchangeable.

## Study Tips

- **Build the mental hierarchy first:** Trail (delivery configuration) → Event Selectors (what gets captured) → Event Data Store / S3 bucket (storage) → Lake / Athena (query layer) → EventBridge (real-time fan-out). Most confusion comes from conflating storage with delivery configuration.
- **Cost-test your filters before deploying:** Use the `read-only=true` + `eventCategory=Data` permutation to estimate volume before flipping data events on for a high-traffic S3 bucket. Per Corey Quinn's math, a misconfigured S3 data event trail on a bucket fronting a CDN origin can multiply your bill by 5-10× overnight.
- **Connect to yesterday's CloudWatch knowledge:** CloudTrail answers "*who* did what, when" — CloudWatch answers "*what* happened to the system." They overlap on metric filters (CloudTrail logs to CloudWatch Logs group, filter pattern fires metric, alarm fires SNS) but their philosophies are different. Frame CloudTrail as the immutable audit ledger; CloudWatch as the mutable operational dashboard.
- **The OIDC correlation drill:** After Resource 8, write the exact Athena query for "list every `AssumeRoleWithWebIdentity` in the last 7 days where the `sub` claim starts with `repo:my-org/` and the assumed role ends with `-deploy`." That single query ties together May 11 (OIDC hardening), today (CloudTrail correlation), and tomorrow's likely topic (incident-response playbooks).

## Next Steps

- **GuardDuty deep dive** — GuardDuty's foundational data source *is* CloudTrail management events + S3 data events + VPC Flow Logs + DNS logs. Understanding what CloudTrail captures is a prerequisite to understanding what GuardDuty can detect (and the blind spots when data events are disabled).
- **AWS Config + Conformance Packs** — the configuration-state counterpart to CloudTrail's API-action stream. Together they answer "what is the state now, and how did it get this way?"
- **Amazon Detective** — consumes CloudTrail + VPC Flow + EKS Audit + GuardDuty findings into a behavior graph; the natural next layer above raw CloudTrail Lake queries for investigation workflows.
- **Build a hands-on lab:** Org trail to centralized S3 bucket in a log-archive account, KMS-encrypted with a CMK whose key policy denies decrypt to the source accounts, integrity validation enabled, Athena partition projection set up, and one EventBridge rule for `StopLogging` to Slack. End-to-end, this is the production minimum.
